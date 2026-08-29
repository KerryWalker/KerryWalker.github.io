---
layout: post
title: The SQL Server Gotcha Where LEN and RIGHT Disagree
excerpt: LEN ignores trailing spaces but RIGHT doesn't, and the mismatch quietly corrupted a subset of our data during a system merge. A SQL Server gotcha that only showed up when the totals didn't add up.
tags:
  - sql
  - sql-server
  - gotcha
---

This one's a bit different from the last couple of posts. No gentle analogy, just a gotcha that cost us a fair bit of head-scratching, and a lesson in trusting `LEN` a little less than I used to.

## The Setup

We were merging two large systems into one. Part of that meant rewriting nominal codes, account numbers, that sort of thing. The first two characters of every nominal were a location code, and both systems had several locations using the same code, so a lot of them had to be swapped for a new one. Change the location code and you've got a mountain of related accounting data that needs updating to match.

The job looked simple enough. Take the new two-character location code, stick the rest of the nominal on the end, done. So we wrote something along these lines:

```sql
UPDATE Ledger
SET Nominal = @NewLocation + RIGHT(Nominal, LEN(Nominal) - 2);
```

Read it out loud and it sounds right. New location code, plus everything after the first two characters. We ran it, it completed without complaint, and we moved on.

## The Reports Didn't Add Up

At the end of the merge we ran the reports, and there were codes in there that shouldn't exist. Totals that should have reconciled were out, money sat under nominals nobody recognised, and the numbers we expected weren't there.

The strange part was that most of the data was fine. It was only some rows that had gone wrong, which made it all the harder to spot. Whatever had happened hadn't happened to everything.

The rows that broke had one thing in common: trailing spaces on the nominal.

## Why It Broke

Here's the gotcha. In SQL Server, `LEN` does not count trailing spaces. `RIGHT` does.

Take a nominal like `AB123456`, where `AB` is the old location code and `123456` is the account. On a clean row the update does exactly what you'd expect:

```sql
SELECT 'XY' + RIGHT('AB123456', LEN('AB123456') - 2);   -- 'XY123456'
```

`LEN` is 8, we take the rightmost 6 characters, `123456`, put the new location code on the front, and get `XY123456`. Perfect.

Now take the same nominal from one of the rows that had picked up a couple of trailing spaces somewhere along the way:

```sql
DECLARE @v varchar(20) = 'AB123456  ';   -- two trailing spaces

SELECT LEN(@v);          -- 8, the trailing spaces are ignored
SELECT DATALENGTH(@v);   -- 10, the spaces are counted
```

`LEN` still says 8, because it pretends the spaces aren't there. So the update asks for the rightmost 6 characters again, but this time `RIGHT` counts from the real end of the string, spaces and all:

```sql
SELECT RIGHT(@v, LEN(@v) - 2);   -- RIGHT('AB123456  ', 6) => '3456  '
```

The last 6 characters of `AB123456  ` are `3456` followed by the two spaces, not `123456`. So instead of `XY123456` we wrote `XY3456`. The `12` was thrown away, and every affected row landed under a made-up code. On the clean rows the maths lined up and the update did exactly what we wanted, which is precisely why it slipped through.

## The Fix

The direct fix is to stop trusting `LEN` to tell you how long the string really is. Trim it before you measure, or use `DATALENGTH` if you genuinely need the physical length (just remember `nvarchar` is two bytes per character, so you'd halve it).

But the better fix is to not do positional maths from the wrong end at all. When you're replacing a prefix, work from the left where the padding can't get in the way. `STUFF` does exactly this:

```sql
UPDATE Ledger
SET Nominal = STUFF(Nominal, 1, 2, @NewLocation);
```

Replace two characters starting at position one with the new location code. It never looks at the end of the string, so trailing spaces have nothing to grab hold of.

## The Lesson

`LEN` quietly hides trailing spaces, and the moment you combine it with a function that doesn't, like `RIGHT` or `SUBSTRING` counting from the end, the two disagree and your data pays for it. Trailing spaces creep in all the time, from imports, from hand-entered data, or from fixed-width `char` columns that pad automatically, so this is far more common than you'd think.

If you take one thing from this: when you're measuring a string to chop it up, trim it first, or anchor from the left and don't measure at all. And check your totals reconcile before you call a merge finished.
