---
layout: post
title: "What Went Wrong Anyway"
pillar: migration
series: data-migration
excerpt: "Both systems issued their own reference numbers, and both had sites with the same two-letter code. Changing a code that customers read means rewriting every row that ever printed it, and that is before trailing spaces quietly mangled a few thousand account codes."
tags:
  - sql
  - sql-server
  - data-migration
---

The previous six posts describe an approach that worked. This one is what went wrong anyway.

## Both systems were still counting

Reference numbers are the thing nobody thinks about until it is too late.

Plenty of records get a human-facing number that is not the primary key: a reference printed on a document, quoted on the phone, written on a form. Both systems had been issuing them for years, from their own counters, and both counters had reached broadly similar places.

So once a group of sites came across, the surviving system carried on from where *its* counter had got to, and started handing out numbers that already existed on paperwork belonging to records that had just arrived with them. Two documents, the same reference, both legitimately issued, years apart.

The database has no opinion about this. The column is not unique, because it was never meant to be unique across two systems that did not know about each other. Nothing fails. It just becomes impossible to answer "bring me up the record for reference 41892" without asking which one.

The fix is unsubtle. Burn the range: fill the reference number table with every number up to well beyond anything either system had used, every one flagged as already issued, so the generator skips the lot and starts somewhere safe.

```sql
DECLARE @sites TABLE (SiteId INT)

INSERT INTO @sites
SELECT TargetSiteId FROM MigrationMap_Site WHERE RunId = @runId

;WITH n(n) AS
(
    SELECT 1
    UNION ALL
    SELECT n + 1 FROM n WHERE n < 600000
)
INSERT INTO ReferenceNumber (NumberType, SiteId, Number, IsUsed, UsedAt)
SELECT @accountNumberType, s.SiteId, n, 1, GETDATE()
FROM   n, @sites s
WHERE  NOT EXISTS (SELECT 1 FROM ReferenceNumber
                   WHERE NumberType = @accountNumberType
                     AND SiteId     = s.SiteId
                     AND Number     = n)
OPTION (MAXRECURSION 0)
```

Two things in there that will bite you if you copy it.

A recursive CTE stops at a hundred iterations by default and gives up with an error. `OPTION (MAXRECURSION 0)` removes the ceiling, and you want it set deliberately rather than discovered. The comment next to the upper bound in ours reads `IF YOU DON'T SET THIS NUMBER THIS WILL RECURSE UNTIL SQL GOES POP`, which is accurate.

And the `NOT EXISTS` is what makes it safe to run more than once, which matters more than it first appears. The sites come from the site mapping, filtered to this run, and the whole thing is the last step of the orchestrator, so it runs for every stage without anybody having to remember. Each time it only ever fills in the gaps.

Several hundred thousand rows per site, which is wasteful and completely irrelevant. Rows in a lookup table cost nothing. Reissued reference numbers cost somebody an afternoon every time one surfaces.

## The same two letters, two different places

Reference numbers were not the only thing colliding.

Sites had short codes, two letters, and those codes were not an internal detail. They were the leading part of customer-facing identifiers: account codes, document references, the thing somebody reads down the phone. Both systems had been assigning them independently for a decade, from the obvious abbreviations of real place names, so of course a handful landed on the same two letters for completely different sites.

One of them has to change. That is not a technical decision and the technical team should not make it, but it does need making early, because everything downstream waits on it. In our case it was the surviving system's codes that changed, which is not the way round you might expect, and it meant the first thing rewritten was data that was not moving anywhere.

And a site code is not a foreign key. If it were, changing it would be an update to one row and the relational model would carry the rest. It is denormalised on purpose, because it is part of a string that gets printed, so it is sitting in full inside identifiers on accounts, on ledger rows, on transactions, on correspondence, on anything that ever captured a reference at the time it was raised. Changing the site means rewriting every one of those, in every table that holds a copy.

A dedicated script, run before the first stage, did nothing else. Twenty-odd `UPDATE` statements across as many tables, every one the same shape:

```sql
UPDATE  i
SET     i.InvoiceNumber = c.NewSiteCode + RIGHT(i.InvoiceNumber, LEN(i.InvoiceNumber) - 2)
FROM    Invoice i
JOIN    @changedCodes c ON c.OldSiteCode = LEFT(i.InvoiceNumber, 2)
```

It creates a handful of indexes before it starts, because every one of those joins is matching on the first two characters of a string. Hold on to that `LEN`. It comes back.

Knowing where the copies were was not the problem. We had both systems for years and the schemas held no surprises. The problem is the sheer number of places, each needing its own statement, each one a chance to get a `WHERE` clause slightly wrong on data that is about to go live.

The arriving data got the same treatment on the way in. This is the expression that gave every arriving account its new number, as the mapping tables were being filled:

```sql
REPLACE(LEFT(a.AccountNumber, 2), sm.SourceSiteCode, sm.TargetSiteCode)
    + RIGHT('000000' + LTRIM(RTRIM(SUBSTRING(a.AccountNumber,
            PATINDEX('%[0-9]%', a.AccountNumber), LEN(a.AccountNumber)))), 6)
```

Swap the first two characters if the site code changed, keep the digits, pad back to six. It runs for every account whether its code changed or not, which is the simplest way of making sure the ones that did are not a special case anywhere downstream.

Then there is the part that is not in the database at all. Paperwork has already gone out with the old code on it. Customers have it written down. Whatever you do, for some period afterwards, people will quote references that no longer exist, so somebody needs to be able to translate. Which is why the mapping holds the old number next to the new one, and why every account whose number changed had a note added to it saying what it used to be.

The general shape is worth knowing before you cost one of these. Every identifier a human reads has been copied into rows as a string, deliberately, because that is what printing it requires. A merger is the one event that makes two of those collide, and the size of the job has nothing to do with how many sites changed and everything to do with how many places kept a copy.

## Trailing spaces, and arithmetic that disagreed with itself

That rewrite of the surviving system's references is where the next one went wrong, in the `RIGHT(x, LEN(x) - 2)` on every line of it. It has a post of its own because it is a good standalone trap: [LEN ignores trailing spaces but RIGHT does not](/2026/08/24/sql-server-len-trailing-spaces-gotcha.html).

Short version. Replacing the site prefix and keeping the rest was written as "take everything after the first two characters" using `LEN` to work out how much that was. On a clean value it did exactly that. On a value with two trailing spaces, `LEN` reported the length without them and `RIGHT` counted from the real end, so the two disagreed by two characters and the result quietly lost a digit off the front of the number.

Most rows were fine. Some were not, and the ones that were not landed under codes that had never existed, which is how it was eventually noticed: the reports did not reconcile.

The fix is still visible in the file. On one line the `LEN` is commented out and a fixed width is there in its place, which is the right answer when the column is fixed width anyway.

The lesson I took from it is not about those two functions. It is that string surgery on data you did not create is where migrations go wrong, because you are reasoning about what the data *should* look like rather than what it does. Anchor from the left, or trim before you measure, or ideally do not measure at all.

## How you actually write two hundred of anything

Those twenty-odd updates, and the two hundred lookup translations in the static mappings before them, bring up something that is not a thing that went wrong but is the reason more things did not.

A lot of a migration is statements that are identical except for their values. One `UPDATE` per lookup code. One row per site. One `INSERT` per translation somebody has signed off. Two hundred of them, differing in two places each.

Typing those is where the mistakes live. Not big mistakes. The one in the middle where the second value did not get changed along with the first, which runs perfectly and does the wrong thing to one row.

So you do not type them. You put the values in a spreadsheet, build the statement once in a formula with the literal text and the cell references concatenated together, and drag it down the column. Paste the result into a `.sql` file and it looks exactly like somebody sat and wrote two hundred statements by hand.

What it buys is consistency by construction. A mistake in the formula is one mistake, in one visible place, applied identically to every row, which means it is either obviously wrong everywhere or right everywhere. Hand-typed, the same mistake is in row 147 only, and nothing about row 147 looks different from the rest. It also runs in both directions: `SELECT` the current state out, paste it in next to the intended state, and generate from the comparison.

## In short

Two systems that have been issuing their own numbers will carry on doing so after you merge them, and nothing will tell you. Burn the range.

Any identifier a human reads has been copied into rows as a string, because printing it requires that. When two of them collide, the cost is the number of places holding a copy, not the number of records whose code changed.

String arithmetic on data you did not create will find an edge case you did not consider, and it will do it quietly, on a subset, in a way that reconciles until somebody adds up a column.

And when the job is two hundred statements that differ only in their values, generate them from a spreadsheet rather than typing them, so that a mistake is in the formula and not in row 147.

That is the series.
