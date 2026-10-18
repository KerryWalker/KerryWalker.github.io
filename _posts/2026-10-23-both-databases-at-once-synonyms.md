---
layout: post
title: "Writing SQL as if Both Databases Were One: Synonyms"
pillar: migration
series: data-migration
excerpt: "Every script in a migration needs to read the old system and write to the new one. Three-part names everywhere make that brittle and impossible to test. There is a tidier answer and it takes ten minutes."
tags:
  - sql
  - sql-server
  - data-migration
---

So far: two systems with a common ancestor, ten years of independent development since, and a destination that now has somewhere to put everything that is coming. Time to start moving it.

Before any of the interesting work, there is a dull obstacle. Every script you are about to write needs to read from one database and write to another, and the obvious way of doing that is worse than it looks.

## The obvious way, and what it costs

Both databases are on the same server, so you can reach across:

```sql
INSERT INTO Account (...)
SELECT ... FROM OldSystemLive_20261023.dbo.Customer
```

That works, and it is how the first few scripts got written. The trouble starts when there are a hundred of them.

The database name is now baked into every script in the project. Restore a fresh copy of the source to test against and it comes back with a different name, because the name has a date in it, and now every script is wrong. Point at a smaller extract to iterate quickly and every script is wrong again. Run against the real thing on the day and you had better have remembered to change all of them back.

You can solve that with dynamic SQL, building each statement as a string with the database name substituted in. We rejected that early. It makes every query in the project unreadable, unformattable and un-syntax-checkable, to solve a problem that only exists because the name is in the wrong place.

## Synonyms

SQL Server has a feature for exactly this, and it is one of the ones nobody reaches for.

```sql
CREATE SYNONYM Src_Account FOR OldSystemLive_20261023.dbo.Customer
```

From that point on, `Src_Account` is a table name in the target database. You select from it, join to it, and it behaves like any other table. Nothing in any script mentions where it actually lives.

So the script above becomes:

```sql
INSERT INTO Account (...)
SELECT ... FROM Src_Account
```

and stays correct whichever source database the synonyms happen to point at.

The prefix is the convention that makes it work. Every source table gets the same one, so the rule is simple enough to hold in your head: no prefix means the new system, `Src_` means the old one. Reading a join and knowing instantly which side each table is on matters more than you would expect when a query has nine of them.

## Generating them, not writing them

There were several hundred tables. Nobody was going to write those by hand, and a list written by hand would be out of date within a week.

So they get generated. The source database name is a variable, which means reading its catalogue has to be dynamic, and this is one of only two places in the project where that is the right answer (the other is the teardown script further down). First, read the catalogue:

```sql
SET @sql = '
    INSERT INTO #SourceTable (TableName, SchemaName)
    SELECT TABLE_NAME, TABLE_SCHEMA
    FROM ' + @sourceDb + '.INFORMATION_SCHEMA.TABLES
    WHERE PATINDEX(''%[$*0-9]%'', TABLE_NAME) = 0
      AND PATINDEX(''[_]%'', TABLE_NAME) = 0'

EXEC (@sql)
```

Then one synonym per table, from a temp table with a processed flag:

```sql
WHILE @TableName IS NOT NULL
BEGIN
    IF EXISTS (SELECT 1 FROM sys.synonyms
               WHERE name = @prefix + @TableName
                 AND schema_id = SCHEMA_ID('dbo'))
    BEGIN
        EXEC ('DROP SYNONYM dbo.' + @prefix + @TableName)
    END

    SET @sql = 'CREATE SYNONYM dbo.' + @prefix + @TableName
             + ' FOR ' + @sourceDb + '.DBO.' + QUOTENAME(@TableName)
    EXEC (@sql)

    UPDATE #SourceTable SET Processed = 1 WHERE TableName = @TableName
    SET @TableName = (SELECT TOP 1 TableName FROM #SourceTable WHERE Processed = 0)
END
```

Note that it drops an existing synonym before creating it. Run the whole thing twice and the second run is harmless rather than a few hundred errors, which matters more than it sounds when this is the first step of something you are about to run repeatedly.

The two `PATINDEX` calls in the first block are the interesting part, and they are there because a database that has been live for a decade accumulates table names nobody intended as tables. Anything containing a digit is excluded, which catches the copies somebody took before a release and named with a date. So is anything with a currency symbol or an asterisk in it, and anything beginning with an underscore. None of those were being migrated, and a generator that fell over on the first one would have to be babysat every time the source database changed.

## Being able to throw them away

The counterpart is the one that gets forgotten, and it is six lines:

```sql
DECLARE @sql NVARCHAR(MAX) = N''

SELECT @sql += N'DROP SYNONYM ' + QUOTENAME(SCHEMA_NAME([schema_id]))
             + N'.' + QUOTENAME(name) + N';' + CHAR(13) + CHAR(10)
FROM sys.synonyms
WHERE SCHEMA_NAME([schema_id]) = N'dbo'

EXEC sys.sp_executesql @sql
```

Query `sys.synonyms`, build a `DROP` for each one, run the lot.

That script is what makes the whole arrangement comfortable rather than a commitment. Tear them down, point at a different source, build them again, and every script in the project is now running against the new source with nothing edited. Testing against last night's restore, or against a cut-down extract, or against the real thing, is a thirty second job.

`QUOTENAME` is doing real work in there, and it is the habit worth copying. The moment you are building SQL from names that came out of the catalogue, you are one awkwardly-named object away from a broken statement.

## What it costs

Synonyms hide where the data comes from, and that is the point of them, which is also the problem with them.

Six months later, somebody reads a script, sees `Src_Account` joined to `Customer`, and has to know the convention to understand that one of those is in a different database. Nothing in the query says so. There is no three-part name to notice, no warning, nothing. If the prefix had been something uninformative rather than `Src_`, that would be a genuinely nasty trap.

The other cost is that the prefix becomes load-bearing. Every script refers to those names literally, so changing it means changing all of them. Ours carries a comment saying as much, in capitals, which was the right call: a hardcoded constant with a loud warning beats making the whole project dynamic so that one string can vary.

## In short

If you are writing more than a handful of scripts against another database, put a synonym layer in front of it before you write the rest.

It takes ten minutes, it removes the database name from every script in the project, and it turns "test this against a different copy of the source" from an afternoon of find-and-replace into dropping and recreating a few hundred synonyms. Keep the teardown script next to the setup one, and pick a prefix you will not regret reading in two years.

Next, the thing all of those scripts exist to populate: a table for every kind of record, saying what each old identifier became.
