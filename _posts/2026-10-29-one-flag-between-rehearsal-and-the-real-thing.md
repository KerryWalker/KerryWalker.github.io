---
layout: post
title: "One Flag Between a Rehearsal and the Real Thing"
pillar: migration
series: data-migration
excerpt: "A migration you can only run once is a migration you cannot test. Ours ran against live data as often as we liked and threw it all away afterwards, which is the only reason the real one was boring."
tags:
  - sql
  - sql-server
  - data-migration
---

Everything so far has been about getting the data across correctly. This one is about the thing that actually decided whether it worked: being able to do it over and over until it was boring.

## The flag

The entire import runs inside one transaction, and the last thing the orchestrating script does is decide whether to keep it:

```sql
IF @test = 1 OR @rc > 0
BEGIN
    PRINT 'ROLLED BACK'
    ROLLBACK TRAN
END
ELSE
BEGIN
    PRINT 'COMMITTED'
    COMMIT TRAN
END
```

Set `@test = 1` and the whole run happens. Every step executes against real data, every row gets inserted, every mapping table fills in, every count can be inspected while it is going. Then none of it happened.

That is the dress rehearsal, and it is one variable.

What it buys is not really technical. It is that running the migration stops being an event. You can do it on a Tuesday afternoon because you are curious whether a change helped. You can do it after someone adds three hundred records to the source to see what they do. By the time the first real stage came around we had done it dozens of times, and the only difference on the night was that one digit was a 0. Then we did it again for the next group of sites, and the one after.

Running it in stages made that better rather than worse, in three ways.

It shrank what you had to hold in your head. One group of sites, got working, proved, done. The next group starts from something that already works rather than from a theory, and problems arrive a few at a time instead of all at once on the only night that matters.

Before the first group went anywhere near production, we could run the whole sequence end to end, every stage, against a copy. Not a sample or a representative subset. All of it.

And then the rhythm of the real thing did the rest. It ran a group of sites at a time over a couple of months, and each stage finished with a backup. That backup is exactly the state the next stage will start from, so restoring it gives you a rehearsal environment that matches production precisely, for free, as a by-product of having done the previous one. Between one stage and the next there was always a full dress rehearsal of what was coming, against the real thing as it actually stood.

Which is the part worth stealing. The usual problem with rehearsing a later stage is getting hold of a destination in the right state. Staging hands you one after every live run, whether you asked for it or not.

## One transaction, not forty

For that to work, the transaction has to wrap everything. The orchestrator opens it before the first step and the individual procedures deliberately do not open their own.

They say so themselves. The same comment appears near the top of nearly all of them, word for word:

> NOTE NO TRANSACTION, THE TRANSACTION CAN BE HANDLED IN THE CALLING FUNCTION

That it appears in almost every file rather than one of them is the point. A single procedure with that note is somebody explaining a decision they made this morning. Nearly all of them carrying it is a convention, written down where it will actually be read, because the obvious reading of a procedure that modifies data and does not open a transaction is that somebody forgot. Left unsaid, it is exactly the kind of thing the next person through helpfully "fixes".

The convention is the right call for steps meant to compose. A procedure that opens its own transaction has decided, on your behalf, that its work is a unit that can succeed while later work fails. For a migration that is exactly wrong: the thing that must be all-or-nothing is the whole import, not the account import.

It is also why nesting them would not have helped. SQL Server will let you `BEGIN TRAN` inside a transaction, but the inner one is bookkeeping rather than a real boundary, and only the outermost `COMMIT` means anything. Code that looks like it manages its own transaction, and in fact does not, is worse than code that says plainly that the caller is in charge.

The cost is a long-running transaction holding locks across the whole import. On a system nobody else was using during the window, that was free. It would not be on a database still serving traffic, and that is the trade to be aware of before copying this. Moving a group of sites at a time rather than everything at once helps here more than it looks: each run is a fraction of the whole, so the window it needs is a fraction too.

## What re-running means when you run it in stages

While the process was being developed, every mapping procedure dropped its table and recreated it. That is the right thing to do while the shape of the tables is still changing: a column gets added, the procedure is redeployed, the next run starts clean, and a failed attempt leaves nothing to tidy up because the next one flattens everything regardless. No reconciling, no merging, nothing to remember to do first.

It is also completely wrong once the thing is live and running in stages. Stage two cannot drop the mapping tables, because they hold stage one's translations, and stage one's records are already live in the surviving system. Dropping them would not just lose history. It would leave real rows pointing at identifiers that nothing could any longer explain.

So once the process was locked down, the drops came out. The tables are created once and never dropped, and the procedure that builds them checks for each one and only creates what is missing:

```sql
IF NOT EXISTS (SELECT 1 FROM INFORMATION_SCHEMA.TABLES
               WHERE TABLE_NAME = 'MigrationMap_Account')
BEGIN
    CREATE TABLE MigrationMap_Account (...)
END
```

Forty-odd of those, one after the other, and the second run through finds them all present and does nothing.

What **is** dropped and recreated on every deployment is the procedure itself. Each script file begins by removing the version of the procedure that is already there and then defines it again, so the code is always whatever the file says and never whatever happened to be deployed last time. Code is disposable. The mapping data is not.

That means the safety has to live somewhere else, and it lives in two places. Every mapping table that is filled per stage carries the id of the run that put each row there, so one stage's rows can always be told from another's. The static ones from the second phase, the lookup translations, have no run column at all, because they are filled once and shared, and the schema says so. And every procedure that fills a mapping table skips anything already in it:

```sql
WHERE  sm.RunId = @runId
AND    a.AccountId NOT IN (SELECT SourceAccountId FROM MigrationMap_Account)
```

Run the same stage twice and the second pass finds nothing left to add. Run the next stage and it adds only the new sites' records, alongside everything from before. Re-running is still safe, just not by construction any more. It is a property each procedure maintains for itself, and that is the price of the tables being permanent.

Which they are, and that was always the point. They outlive the final stage, and somebody asking months later why a record has the number it has, or what a code from the old system turned into, gets a query rather than an archaeology exercise. There are scripts in that project, written long after the last stage, that do nothing but join to them.

## Knowing where it broke

Two small habits that paid for themselves repeatedly.

Every procedure takes a debug flag and prints what it is doing when the flag is on:

```sql
IF @debug = 1
BEGIN
    PRINT 'IMPORT ACCOUNTS'
END
```

Noisy when you are watching, silent when you are not, decided in one place at the top of the run.

And every procedure wraps its body in the same error trap, which prints what went wrong and hands a failure code back to the caller:

```sql
BEGIN CATCH
    PRINT 'ERROR'
    PRINT ERROR_MESSAGE()
    SET @rv = 1
END CATCH

RETURN @rv
```

The orchestrator sees the non-zero return and stops, and the failure sits directly underneath the name of the step that caused it. The previous post covered why that matters at three in the morning.

## Tidying up after yourself

The last thing the script does, before deciding whether to keep the run, is drop the synonyms. Not after a successful commit. Before the decision, so they are gone either way.

It would be easy to leave them. They are harmless, they are only pointers, and recreating them is cheap. But leaving them means the new system holds references to a database that is about to be decommissioned, and a stray script that still joins to `Src_Account` would keep working right up until the day the old server was switched off. Clearing them at the end means that script fails immediately, in testing, rather than six months later in production.

## Knowing what has been run

The last piece was not built for the migration, and that is the interesting part.

We already had a table called `ScriptLog`, and had done for years. Any script run against a database as part of a release, a bug fix or a data correction inserted a row recording what it was:

```sql
INSERT INTO ScriptLog
    (Author, FileName, Description,
     ReleaseNumber, ReturnValue, Message, RunAt)
VALUES
    ('XX', 'usp_Populate_Account_Mappings.sql', 'Populate account mappings',
     '4.11.0', @errors, @errmess, GETDATE())
```

It turned out to answer the question that sinks migrations: **what has actually been run against this database?** Not what was supposed to be run, or what the release notes say, or what somebody remembers doing. A query. With timestamps, return values and error messages, against the database in question.

Over a hundred of the migration scripts write to it. When a test environment behaved differently from another one, that table usually explained why within a minute.

The general version, which I did not appreciate until this project: housekeeping conventions you keep up when they seem pointless are the ones that save you later. Nobody writes an audit table because they are planning a database merger.

## In short

Wrap the whole import in one transaction and let a single flag decide whether it commits. Run it against real data as often as you can stand, until a real stage is indistinguishable from a rehearsal.

Moving in stages helps more than it costs. Each live run ends with a backup, and that backup is the exact starting state for rehearsing the next one, so the schedule hands you a faithful practice environment between every pair of runs without anybody having to build one.

Make the mapping tables permanent and tag every row with its run, so earlier stages survive later ones and the tables are still there to answer questions when it is all over. Then make every procedure responsible for not adding what is already there, because that safety no longer comes for free. Let each step announce itself so a failure names itself, and clean up the scaffolding at the end so dependencies on it fail loudly rather than quietly.

And keep a record of what has been run where, which is dull right up until the morning it is the only thing that explains what you are looking at.

Last one: what went wrong anyway.
