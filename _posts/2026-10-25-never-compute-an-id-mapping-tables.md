---
layout: post
title: "Never Compute an Id, Look It Up: Mapping Tables"
pillar: migration
series: data-migration
excerpt: "The obvious shortcut is to add a number to every identifier and call it done. Here is why that falls over, and what we used instead: more than forty tables whose only job was to remember what became what."
tags:
  - sql
  - sql-server
  - data-migration
---

Account 4192 exists in both systems and is two different customers. That was [the problem]({% post_url 2026-10-19-two-systems-that-drifted-apart %}). This is the answer, and it is less clever than you would hope.

## The shortcut that nearly works

The first idea anybody has is an offset. Add a hundred thousand to every identifier coming from the old system and the two cannot collide. Account 4192 becomes 104192, the ledger rows pointing at 4192 become rows pointing at 104192, and the arithmetic takes care of itself.

It is appealing because it needs no bookkeeping. It is also wrong in several ways that only show up once you have committed to it.

It assumes the target is empty, or at least that nothing in it will ever reach 100000. One of ours would have, eventually.

It assumes you only run once. Run the import twice and every record arrives again under the same computed number, because the arithmetic is deterministic and has no idea it has already been done. You can guard against that, but now the offset has a flag next to it and you are keeping records anyway.

It assumes a single column decides identity, which we will come back to, because that is the assumption that really hurts.

And when something goes wrong three weeks later and somebody asks which old record this new one came from, the answer is "subtract a hundred thousand, probably, unless it was one of the ones we fixed up by hand".

## Write it down instead

For every kind of record, a table. One row per source record, saying what it became.

```sql
CREATE TABLE MigrationMap_Account (
    SourceAccountId      INT,
    SourceAccountNumber  VARCHAR(8),
    SourceCustomerId     INT,
    TargetAccountId      INT,
    TargetAccountNumber  VARCHAR(8),
    RunId                INT)

CREATE INDEX IX_Account_Source_Target
    ON MigrationMap_Account (SourceAccountId, TargetAccountId)
```

That is the whole idea. Nothing is ever computed, inferred, or worked out from a pattern. When a ledger row needs to know what its account became, it joins to this table and reads the answer.

The index matters more than it looks. Every subsequent import joins through these tables, often several at once, and an unindexed mapping table turns a migration into an overnight job.

Two of those columns are not identifiers at all, and they earn their place.

An account has a human-facing number as well as a primary key, and the number it will have in the destination gets worked out here, before anything is inserted. That is partly because it might collide with one already in use, which is a story for the last post in this series. But it is also what makes the whole thing join back together afterwards. Identity columns are assigned by the destination on insert, so at the moment you insert a few thousand accounts you do not yet know what numbers they got. You do know what account number each one was given, because you calculated it. So you insert, then come back and match on the number to collect the identity. Most tables have no number like that, and collecting the identity for those takes a different trick, which the next post covers.

`RunId` records which stage of the migration put the row there. The move happened a group of sites at a time, the mapping tables are permanent and shared, and every row needs to say which run it belongs to. That column turns out to do a lot of work, and the next two posts are largely about what it does.

There were more than forty mapping tables by the end. Accounts, ledgers, category codes, sites, service contracts, payment plans and the transactions under them, journals, correspondence. Anything with an identity column that anything else referred to.

## The column that is really about people

Some of the tables carry one more column, `Checked BIT DEFAULT 0`, and it has nothing to do with the SQL.

Its only job is to record that a human being has looked at a mapping and agreed with it. Not every translation can be decided by a rule. Two lookup tables have overlapping values that mean almost the same thing, and somebody who knows the business has to say whether "almost" is close enough. A handful of records are duplicates of each other across the two systems and have to be merged rather than copied.

Those decisions get made once, by a person, and then they are a row in a table rather than something in an email. When somebody asks in six months why those two codes were combined, the answer is in the database.

It is only on the tables where that kind of judgement came up: sites, contracts, notes, a few others. The rest could be decided by a rule and did not need it. Where it does appear, it is the honest signal of where the work actually was. The hard part of a migration is not the SQL. It is sitting with somebody who understands the data and going through the cases where the computer cannot decide.

## Identity is often not one column

This is the assumption the offset trick kills you on, and it is worth seeing a real one.

```sql
CREATE TABLE MigrationMap_Contract (
    SourceContractId        INT,
    SourceAccountId         INT,
    SourceBatchId           INT,
    SourceOrderId           INT,
    SourceCreatedByOrderId  INT,
    SourceParentContractId  INT,
    TargetContractId        INT,
    RunId                   INT,
    Checked                 BIT DEFAULT 0)
```

Six source columns. One target.

You cannot say which contract in the new system a given old contract corresponds to by looking at the contract alone. You need the account it belongs to, the transaction batch it was raised against, the order it was sold on, the separate order that caused it to be created, and the contract it was renewed from. Two different orders, which is the kind of distinction that only exists once you have looked. A contract on its own is not identifiable; a contract in its context is.

That is normal rather than exotic. Anything that only exists as part of a larger arrangement behaves this way, and every one of those extra columns is a join you are going to need later when you work out what this record became.

Build the mapping table to hold the whole identifying context, not just the key. It costs nothing at create time and it is miserable to add afterwards.

## In short

Never compute an identifier. Write down what each one became, in a table, indexed, one per kind of record.

Build the table to hold the whole context that identifies a record rather than just its key, because identity is usually more than one column and you will want those joins later. And give it a column for recording that a person agreed with a mapping, because some of them cannot be decided any other way and you will want to know which.

What you cannot do is fill in the target side of any of them until its records are in, and nothing that points at those records can go in until that has happened. The order is decided for you, which is next.
