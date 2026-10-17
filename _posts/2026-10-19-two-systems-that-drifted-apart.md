---
layout: post
title: "Two Systems That Drifted Apart: Merging Databases With a Common Ancestor"
pillar: migration
series: data-migration
excerpt: "Two bespoke systems that were the same product a decade earlier, then developed independently ever since. The base tables still rhyme. Everything built since does not. This is what merging them actually involves."
tags:
  - sql
  - sql-server
  - data-migration
---

Some years ago we were asked to merge two systems into one.

Not two different products, which would at least be an honest integration problem. Two bespoke systems that had been **the same system** a decade earlier, before the businesses that owned them went their separate ways. One database was to carry on. The other was to stop existing, and everything in it had to arrive safely in the first.

It did not happen in one night. The move ran a group of sites at a time, a week or two apart, over a couple of months, which turns out to change the shape of almost everything that follows.

The pitch, as these things always are, was that this should be straightforward. Same ancestry. Recognisable tables. How hard can copying rows be.

## Ten years is a long time

The common ancestor is real and it is genuinely helpful. Accounts are still accounts. A ledger still has the shape a ledger has. Open both databases side by side and you can read them both without a guide.

But both systems had been **actively developed** for ten years, by different people, for different businesses, with different priorities. Neither was a frozen copy of anything.

That divergence shows up in three different ways, and they get progressively more awkward.

**The same table, different columns.** A table exists on both sides and has drifted, usually in both directions at once. On one of ours the two versions differ by more than twenty columns, and neither side is a superset of the other.

**Tables that only exist in the system being retired.** Ten years of development built features the other business never asked for, and those features have data in them that somebody is using. Each one needs a decision before it needs any code: does this survive the merger or not? If it does, there is nowhere to put it yet, and somebody has to build that first.

**Tables that only exist in the surviving system.** Less of a problem, because there is nothing to migrate into them, but they still have to be populated with something sensible for every record that arrives, and "sensible" is a business decision rather than a technical one.

## The second of those reshapes the project

That middle case is the one that reshapes the project, and it is worth a post of its own, which is where this goes next.

The short version: if the retiring system has a feature the survivor has never had, there is nowhere to put the data. Not a different shape of somewhere. Nowhere. So before any migration can happen, somebody has to build it, and that is ordinary feature development with a migration-shaped deadline attached.

## And none of the numbers mean anything

All of that is before the problem you actually expected.

Open both databases and look at account 4192.

In one it is a customer in Scotland who has been with us eleven years. In the other it is a different customer entirely, in Devon, who joined last spring. Both rows are correct. Both have been correct for years. They have nothing to do with each other beyond having arrived at the same number by counting upwards from the same starting point at different speeds.

Now multiply that by every table with an identity column, which is all of them, and by every table that refers to another, which is most of them.

A ledger row points at account 4192. Which one? Once the data is in the same place the question stops having an answer. Copy that row across unchanged and its foreign key now points at whoever happens to occupy that number in the destination. Not broken. Wrong. A payment attached to a stranger's account, reconciling perfectly, in a system that will never flag it.

That is the shape of the problem. The data is fine. The identifiers are fine. It is putting them together that turns them into lies.

## What has to be true at the end

Worth writing down, because it is more demanding than it sounds.

Every row that comes across keeps its relationships. The ledger entry is still attached to the same customer it was attached to before, under whatever number that customer now has.

Nothing comes across twice. Runs get repeated, and a migration that duplicates on the second attempt is one you can only ever try once.

We can prove it afterwards. Not "it looked fine when we spot-checked it", but a table showing what every single record became, and numbers that reconcile.

And anything that cannot be decided by a rule gets decided by a person, on purpose, with a record of the decision. There were a lot of those.

## The shape of the answer

What we ended up with is not complicated, and it is roughly what anyone doing this converges on.

**Make the retiring system readable from the surviving one**, so that every script can be written as though both were one database.

**Write down every translation.** For each kind of record, a table saying what each old identifier became. Never compute it, never infer it, never assume a pattern. By the end there were more than forty of these.

**Accept that the order is not yours to choose.** You cannot bring across a ledger entry until its account exists in the destination, because until then the account has no new number to point at. The foreign keys decide what happens first.

**Be able to run the whole thing again tomorrow.** All of it, unattended, as many times as it takes to stop being frightening.

Each of those is a post of its own, and the series finishes with what went wrong anyway, because something always does and the write-ups that leave that part out are no use to anybody about to do this.

## In short

Two installations of the same product diverge far more in ten years than anyone remembers. The base tables still rhyme, which is exactly enough familiarity to make you underestimate it.

Expect three jobs, not one. Build whatever the surviving system is missing before you can migrate into it. Translate every identifier, because none of them mean the same thing on both sides. And do it in an order the foreign keys dictate rather than one you choose.
