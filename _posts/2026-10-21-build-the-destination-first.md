---
layout: post
title: "Before You Can Migrate, You Have to Build Somewhere to Put It"
pillar: migration
series: data-migration
excerpt: "Ten years of independent development meant one system had features the other had never heard of. You cannot map a table onto a table that does not exist, so a good part of a data migration project turned out to be ordinary feature development."
tags:
  - sql
  - sql-server
  - data-migration
---

[Last time]({% post_url 2026-10-19-two-systems-that-drifted-apart %}) I set out the problem: two systems that were the same product a decade earlier, developed separately ever since, and now to be merged into one.

The part that is easiest to leave out of an estimate, and the part nobody warns you about, is that a good deal of a data migration is not data migration at all.

## Somebody has to decide what survives

Before any of this is a technical problem it is a business one, and the technical team cannot answer it.

Go through the retiring system's tables and every one lands in a category. It has a home in the surviving system. It has no home and one needs building. Or it is not coming at all.

That third option is real and it is not a failure, though the reason is not the one you might expect. Nothing in either system was dead wood. Ten years of development on both sides had been enhancement and addition rather than replacement, so almost everything in the retiring system was working, maintained and in use the day it was switched off.

What differed was how the two businesses ran, and a good deal of what each system had grown existed to support its own company's way of working.

So the decision was never really about the features. It was about the direction the surviving business intended to take, and which ways of working it was keeping. Once that is settled, the list sorts itself: anything supporting a practice the merged business is carrying forward has to come across, and anything built around a practice it is leaving behind does not. Not because that feature is bad, and not because nobody used it, but because the thing it supports is ending.

That is the distinction worth holding on to. The question is almost never "is this feature any good". It is "where is this business going, and does this belong there", which is not a question a developer can answer.

There is a middle option worth knowing about too. Keep the retiring database, read-only, after it stops being the system of record. The data is not lost, it is just not live, and anybody who genuinely needs a ten-year-old record can still be given one. That turns "we must rebuild this" into "we must be able to answer a question about this twice a year", which is a much cheaper promise.

What the technical side can do is produce the list and cost it. What it cannot do is decide that inspections matter and some other feature does not. That is somebody who knows the business, going through the list, and the quality of that conversation sets the size of the project.

The same judgement shows up further down at column level. One of our imports carries a comment listing a dozen or so columns that are deliberately not carried, and you can tell from the names that several are configuration belonging to the other company rather than anything about a customer.

## You cannot map onto something that is not there

The retiring system had inspection records. Every unit had a history of checks, who carried them out, what they found, when the next one was due. Ten years of them, belonging to customers who were about to become the surviving system's customers.

The surviving system had no concept of an inspection. Not a different shape of one. None.

Inspections were on the list of things that had to survive, and that was not a close call: the moment the merger completes somebody asks when a unit was last checked, and the answer cannot be that we lost it.

Which means there is no clever mapping available. You cannot translate a table into a table that does not exist.

So before any migration work could happen, somebody had to build inspections into the surviving system. Tables and their lookups, the triggers that maintain them, the screens to view and edit them, and the permissions deciding who can.

That is ordinary feature development. It happens to have been triggered by a migration, but nothing about the work is migration work, and the team doing it needs product decisions rather than SQL.

We had several of these. Inspections, customer notes, and separate correspondence histories for two different parts of the system that the surviving side had only ever kept as one.

Which has a consequence for anybody estimating one of these. If your plan contains only migration tasks, the plan is wrong. Not optimistic. Missing a category.

## The more delicate version

There is a second case, and it is worse than the missing table, because it looks like it is already solved.

Sometimes the surviving system does have the concept, but in a shape that cannot hold what is arriving. It keeps one of something where the other keeps many. It stores the current value as a column where the other keeps a dated history. The data is conceptually the same; the structure cannot receive it.

That means restructuring something that is already live. New tables, then moving the existing records into the new shape, then changing everything that reads the old shape, and all of this on a system that is in daily use by people who did not ask for any of it and will not thank you if it goes wrong.

One of ours was exactly this. The script that did it is one of the larger ones in the project and it has nothing to do with the other system at all. It takes the surviving system's own data and rearranges it, so that months later some other data can arrive and fit.

Creating a new table is safe because nothing depends on it yet. Reshaping an existing one is a change to a working system, and it carries all the risk that implies.

## The quiet third case

The mirror image is easy to overlook because nothing fails.

The surviving system had tables the retiring one did not. There is nothing to migrate into them, which sounds like good news, right up until you insert a few hundred thousand records that have no row in any of them.

Every one of those needs something sensible. Sometimes a default is obviously correct. Sometimes it is a business decision nobody has been asked to make, and you will not discover that until somebody runs a report in month two and the arrived records are all missing from it, because they were never given the classification the report groups by.

That one is worth hunting for deliberately, because unlike the other two it does not announce itself. Nothing errors. The records simply arrive incomplete in a way that only matters later.

## And one you decide to lose

There is a fourth case that is not about missing structure at all. Sometimes both systems have the column, both have real values in it, and you overwrite them anyway.

Audit columns are the example. Every table has some version of who last changed this and when. The retiring system had ten years of them. The import procedures take a user and a timestamp as parameters and stamp every arriving row with those:

```sql
INSERT INTO Site (..., UpdatedBy, UpdatedAt, ...)
SELECT            ..., @updatedBy, @updatedAt, ...
```

So every record that came across is recorded as having been last touched by the migration, on the night its stage ran. A decade of who-changed-what stops at the boundary.

That was deliberate and I would do it again. The alternative is carrying usernames from a system whose user table is not the surviving system's user table, so the names would refer to accounts that do not exist, and anybody reading them afterwards would be misled rather than informed. Stamping them makes it obvious that these rows arrived rather than pretending they were always there.

But it is a loss, it is permanent, and it is worth being the one who decided it rather than the one who noticed afterwards.

## What it does to the shape of the project

Three consequences, and they all push the same way.

**The migration cannot be properly tested until the destination exists.** You can write the scripts, but you cannot run them end to end, because a good number of them have nowhere to insert into. So the feature work is not parallel, it is upstream.

**It needs people the migration team does not have.** Deciding what an inspection record should look like in the surviving system is a product question. Building the screens is front-end work. The database part is the small end of it.

**It is the part most likely to be cut.** When a date starts looking tight, "build the missing features" looks like scope that could move, while "migrate the data" does not. It cannot move. If it does, the data arrives with nowhere to live, and you are back to losing the thing you promised not to lose.

## In short

Merging two systems that have diverged is several jobs wearing one name.

Build whatever the surviving system is missing, which is feature development with a migration-shaped deadline. Reshape anything it has in the wrong form, which is surgery on a live system. Work out what every arriving record should say in the columns it has never had an answer for.

Only then does any of the actual data movement start, which is the next four posts.

If you take one thing from this: when you are estimating, go through the retiring system's table list and mark every one that has no home. That list is your real starting point, and it is longer than anybody expects.
