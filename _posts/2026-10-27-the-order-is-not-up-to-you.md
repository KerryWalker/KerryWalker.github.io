---
layout: post
title: "The Order Is Not Up to You: Migrating in Dependency Order"
pillar: migration
series: data-migration
excerpt: "You cannot bring a ledger entry across until its account exists, because until then the account has no new identifier for the entry to point at. The foreign keys decide the running order, and then you write that order down once."
tags:
  - sql
  - sql-server
  - data-migration
---

So far: the old system is readable from the new one, and every kind of record has a table waiting to say what became what. Now the actual copying.

Anybody who has worked with a relational database knows you insert the parent before the child. That part is not interesting. What a merger does is take something you already know and turn it into the constraint that dictates the shape of the entire project.

## Where the order comes from

A ledger entry points at an account. To bring the entry across, you need to know what number its account has in the new system. That number does not exist until the account has been inserted, because the new system assigns it on insert.

So accounts have to be done before ledger entries. Not as a preference. There is no version of the work where it happens the other way round.

Follow that outwards and the whole project falls into a shape that nobody chose. Sites before accounts, because an account belongs to a site. Accounts before ledger entries, before payment plans, before service contracts. Contracts before the transactions raised against them. The foreign keys are a graph, and the migration is that graph walked from the roots down.

The practical consequence is that the target side of a mapping table cannot be filled in until its records have been inserted, and nothing below it can be imported until it has. What that does not say is that everything has to wait. Most of the work does not depend on a target identifier at all, and the finished process is arranged around that.

## Four phases, not forty loops

The obvious reading is a loop: for each kind of record, build its mapping, import it, write the new identifiers back, move on. The first version was arranged roughly that way. The finished one runs in four phases, and the dependency order only governs the last two.

**Create the scaffolding.** One procedure creates every mapping table, more than forty of them, each guarded by `IF NOT EXISTS`. Nothing is dropped. The second run finds them already there and leaves them alone, which matters a great deal later.

**Translate the static data.** Lookup tables and enumerations: category codes, ledger types, document types, payment methods. These are the same for every site and do not change between runs. Each gets its mapping filled in once, by hand where the two systems disagree, and from then on every run reads the same translations. Nothing in this phase depends on which sites are moving.

**Fill in the source side of everything.** Now the run is scoped to a group of sites, and every mapping table gets its rows for those sites: the source identifier, whatever context identifies the record, and for anything that carries a number people read, the number it is going to have in the destination.

```sql
INSERT INTO MigrationMap_Account
       (SourceAccountId, SourceAccountNumber, SourceCustomerId,
        TargetAccountNumber, RunId)
SELECT a.AccountId, a.AccountNumber, a.CustomerId,
       /* new number, worked out here */, @runId
FROM   Src_Account a
JOIN   MigrationMap_Site sm ON sm.SourceSiteId = a.SiteId
WHERE  sm.RunId = @runId
AND    a.AccountId NOT IN (SELECT SourceAccountId FROM MigrationMap_Account)
```

All of this can be done before a single row is inserted into a real table, because none of it needs a target identifier. The `NOT IN` at the end is what makes the table safe to add to run after run: anything mapped in an earlier stage is skipped, whichever run it came in on.

**Import, in dependency order, writing back as you go.** Sites first, then accounts, then everything that points at an account, and the same job at every level. Insert the rows, translating every foreign key through a mapping that already has its target side, then record the identity the destination gave each one against its source id.

The second half of that is the awkward bit. An insert can carry an `OUTPUT` clause that hands back the new identities, but it can only see the inserted row, and the source id is not on it. You get a list of new identities with nothing to pair them with. `MERGE` is the one statement whose `OUTPUT` can see both sides, so most of the imports are a `MERGE` that is never allowed to match. Here is the contract import, reading the account mapping that the previous level completed:

```sql
DECLARE @inserted TABLE (SourceContractId INT, TargetContractId INT)

MERGE INTO Contract
USING   Src_Contract c
        JOIN MigrationMap_Contract m  ON m.SourceContractId = c.ContractId
        JOIN MigrationMap_Account  am ON am.SourceAccountId = c.AccountId
ON      1 = 0   -- never match: this is an insert wearing a merge
WHEN NOT MATCHED AND m.RunId = @runId THEN
    INSERT (AccountId, ContractNumber, StartsAt, ParentContractId)
    VALUES (am.TargetAccountId,   -- translated, not copied
            c.ContractNumber, c.StartsAt, NULL)
OUTPUT  c.ContractId, INSERTED.ContractId INTO @inserted;

UPDATE  m
SET     m.TargetContractId = i.TargetContractId
FROM    MigrationMap_Contract m
JOIN    @inserted i ON i.SourceContractId = m.SourceContractId
WHERE   m.RunId = @runId
```

Every foreign key on the row goes through a mapping table, and if one of them has no target yet the join drops the record, which is a blunt but effective way of finding out that something upstream was missed. The parent contract is deliberately left null, and the reason is a section of its own further down.

Accounts and sites did it the long way round, because they could. Each carries a number people read, and the previous post explained that the number was worked out before the insert, so a plain insert followed by a join back on the number collects the identity without the merge:

```sql
UPDATE  m
SET     m.TargetAccountId = a.AccountId
FROM    MigrationMap_Account m
JOIN    Account a ON a.AccountNumber = m.TargetAccountNumber
WHERE   m.RunId = @runId
```

Either way, that write-back is the step the next level is waiting for. Each import fills in the target identifiers the one below it needs, and so on down the graph.

## One table decides what is in scope

Running in stages adds a question every step has to answer: is this record part of *this* run?

The obvious way is a filter in every selection, listing the sites in the current run. With forty-odd procedures that is forty-odd places to keep in step, and the one you miss does not fail. It quietly brings across records belonging to a site that is still live in the old system, which then exists in both.

So the scoping goes through the site mapping instead, with a run id.

The site mapping holds every site, permanently, with its old code and its new one. Each run names the sites that are moving, as a list of codes at the top of the script, and a small procedure takes that list, works out the next run number, and stamps it onto those sites' rows:

```sql
SET @runId = (SELECT MAX(ISNULL(RunId, 0)) + 1 FROM MigrationMap_Site)

UPDATE  sm
SET     sm.RunId = @runId,
        sm.ImportStarted = GETDATE()
FROM    MigrationMap_Site sm
JOIN    @siteList l ON l.SiteCode = sm.SourceSiteCode
```

Every other selection already joins to that table, because every record belongs to a site and needs its new site identifier anyway. Filter that join on the run and it is doing two jobs at once:

```sql
INNER JOIN MigrationMap_Site sm ON sm.SourceSiteId = a.SiteId
WHERE      sm.RunId = @runId
```

It translates the old site identifier into the new one, which is what it looks like it is for. And because only this run's sites carry this run's id, anything belonging to a site that is not in the run finds no match and drops out of the result.

Scope stops being something each procedure has to remember and becomes a property of the data it is already joining through. You set the list once, at the top of the run, and forty procedures inherit it without knowing it happened.

It is the single most load-bearing join in the project, and it appears in around a hundred places across the import scripts. Which is worth being aware of in the other direction too: get that list wrong, with a site in it that should not be, and nothing stops you. Every one of those joins will cheerfully do what you asked.

## When the graph points at itself

Two cases break the neat version, and both turn up.

A table that refers to itself is the common one. A service contract has a parent contract, which is another row in the same table. You cannot translate the parent id while inserting, because the parent may not be inserted yet, and the order within one table is not something you control.

The answer is two passes. Insert everything with the self-reference left null, which is allowed because the column is nullable and nothing is validating it mid-migration. Once every row in that table has a mapping, go back and fill the column in, translating through the mapping you have just finished building.

```sql
UPDATE  p
SET     p.ParentContractId = pm.TargetContractId
FROM    Contract p
JOIN    MigrationMap_Contract m  ON m.TargetContractId = p.ContractId
JOIN    MigrationMap_Contract pm ON pm.SourceContractId = m.SourceParentContractId
WHERE   m.RunId = @runId
```

That join reads awkwardly and it is worth following slowly. It goes from the new row back to its mapping to find what it used to be, reads the old parent id off that mapping, then looks that up in the same table to find what the parent became.

A genuine cycle between two tables is the other case, where A refers to B and B refers back to A. Same treatment: insert both with one direction left null, then come back and stitch. If neither column is nullable you have a harder conversation, usually ending in a deferred constraint or a temporary one being dropped.

## Then write the order down once

Working out the dependency order is the hard part, and it is a one-off. Having worked it out, you encode it in a single script, and from then on nobody needs to know it again.

That is what made each real run a non-event. The top of the script is its entire interface:

```sql
SET @sourceDb     = 'OldSystemLive'
SET @prefix       = 'Src_'   -- DO NOT CHANGE: every procedure names these literally
SET @updatedBy    = 'MIGRATION'
SET @updatedAt    = GETDATE()
SET @cutoffDate   = '1 JAN 2000'
SET @debug        = 1
SET @showMappings = 0
SET @test         = 0

SET @siteCodes    = 'AB, CD, EF, GH, JK, LM, NP, QR, ST, UV'
```

Set a handful of variables, press run. The list of site codes is the only line that differs from one stage to the next. No list of scripts to work through, no order to remember, nothing to assemble correctly while under pressure.

Below that, five hundred-odd lines of steps, every one the same shape:

```sql
IF @rc = 0
BEGIN
    EXEC @rc = usp_Populate_Account_Mappings @debug, @runId
    IF @showMappings = 1
        SELECT * FROM MigrationMap_Account WHERE RunId = @runId
END

IF @rc = 0
BEGIN
    EXEC @rc = usp_Import_Accounts @updatedBy, @updatedAt, @debug, @runId
END
```

Three things about that are worth drawing out.

**You can read the four phases straight off the file.** Create the tables, translate the static data, populate every mapping, import every record type. Within the last two, the order of the calls is the dependency graph, written out longhand. The file is the documentation and it cannot drift, because it is also the thing that runs.

**It stops at the first failure.** Every block is guarded by `IF @rc = 0`. A step that fails does not just fail, it prevents everything after it, which matters when the steps after it would otherwise be joining to a mapping table that is half full.

**`@showMappings` lets you look.** With it on, every mapping table is selected out as soon as it is populated, filtered to this run. You can watch the scope and the translations take shape step by step, before the import phase has touched anything. Off, the run is silent apart from progress.

Each real run happened overnight, because that is when these things happen: the system is quiet, nobody is working in it, and there is a window to put it back if it goes wrong. Which means if something does fail, somebody is reading the output at three in the morning, tired, with a limited amount of time to decide whether to fix it or roll back.

In that state you do not want to be working anything out. Every procedure prints what it is about to do when the debug flag is on, in words, and every one traps errors and prints the message before handing the failure back up. So the output reads as a transcript: `IMPORT ACCOUNTS`, `UPDATE ACCOUNT MAPPING TABLE`, then either `ALL OK` or `ERROR` and the reason. Having the whole order in one file, with each step announcing itself, is what turns "it failed somewhere" into "it failed updating the account mappings, and here is why", which is the difference between a decision and an investigation.

## In short

The foreign keys decide the order. Roots first, then everything that refers to them.

But most of the work does not need the order at all. Create every mapping table once. Translate the static data once. Fill in the source side of everything for this run's sites, including the numbers the records are going to have. Only then import, level by level, catching the identity each new row was given with `OUTPUT` and writing it into the mapping, so the next level down can translate through it.

Scope the run through the one table everything already joins to, stamped with a run id, rather than in forty separate filters. Self-references and cycles get two passes, nulls first and stitched afterwards.

Then put the whole order in one script with the configuration at the top, so that the hard-won knowledge of what depends on what lives in a file that runs rather than in somebody's head. That script is the deliverable. The forty procedures are just where the work happens.

Which leaves one question, and it is the one that decides whether any of this is safe: how many times can you run it?
