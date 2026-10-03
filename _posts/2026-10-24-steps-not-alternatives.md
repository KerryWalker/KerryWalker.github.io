---
layout: post
title: "Steps, Not Alternatives: When a Chain Is Really a Pipeline"
pillar: patterns
series: design-patterns
excerpt: "Two bits of code I wrote, in two different jobs, with wiring you could not tell apart. In one the links are alternatives and only one of them answers. In the other they are steps, and that changes everything that follows."
tags:
  - design-patterns
  - csharp
  - dotnet
---

Last time I pulled apart a search chain: six handlers in a line, each one offered the request in turn, and the first that could answer ended it.

I left something hanging. In a different job, years earlier, I wrote something that looks almost exactly like it. A `SetNext` method. A field holding the next one. Objects wired into a line when the thing starts up.

It is not the same pattern. Working out why took me longer than it should have, and the reasons turn out to be the interesting part.

## What it does

It imports data files. A file arrives, gets parsed into rows, the rows get converted into something the application understands, those get saved, and then search indexes get built off the back of them.

Four stages. Here is how they get joined together:

```csharp
var parseProcessor   = new RowParsingProcessor();
var convertProcessor = new RowConvertProcessor();
var saveProcessor    = new SaveRowProcessor();
var indexProcessor   = new IndexProcessor();

parseProcessor.SetNextProcessor(convertProcessor);
convertProcessor.SetNextProcessor(saveProcessor);
saveProcessor.SetNextProcessor(indexProcessor);

```

Put that next to the search chain wiring and you would be hard pushed to say which was which. Same method name, same shape, same idea of a line you read top to bottom.

So what makes one a Chain of Responsibility and the other not?

## No stage is an alternative to another

In the search chain, every link was an alternative. Six searchers, one request, and the job was finding which of the six it belonged to. Most of them did nothing on any given search, and that was the point.

Here, no stage is an alternative to any other. Converting is not something that happens *instead of* parsing. It is what happens *to* whatever parsing produced. You cannot skip a stage, because the next one has nothing to work on if you do. There is no "mine, nobody else needs to look", because they all need to look.

That one difference changes everything downstream of it, which is what I want to spend the rest of this on.

## It needs to be told when to stop

A chain doesn't need a completion signal. The request arrives, it gets offered along the line, somebody answers or nobody does, and it's over. It's one request and it finishes on its own.

A pipeline is not handling one request. It is handling a stream, and a stream has to be told when it has ended. So there's this:

```csharp
NextProcessor?.AllItemsAddedToQueue();
```

When a stage has finished producing, it tells the next one down that nothing more is coming. Without it, the stage after would sit waiting for work that will never arrive. Nothing in the search chain needs anything like that.

Each stage owns a queue, and they all run as concurrent tasks started before any work is queued. Rows are read out of the file and pushed into the first queue as they come, and from then on the stages overlap: the reader is still partway through the file while the converter works on rows it has already emitted and the writer saves ones that went through before those.

That overlap is the actual reason to build it this way, and it is the only thing that justifies the complexity. Do it in phases instead, reading every row, then converting every row, then saving every row, and you are holding the entire file in memory three times over before a single row has been written. Let the stages overlap and you only hold what is in flight.

## What the return value actually means

Here is the loop at the heart of it.

```csharp
while (ItemQueue.Count != 0)
{
    var currentObject = PeekAtNextItem();
    if (ProcessIndividualItem(currentObject))
    {
        itemCount++;
        DeQueueNextItem();
    }
}
```

Look at an item, deal with it, and only then take it off the queue. Peek, process, dequeue.

The obvious reading of that boolean is "did it work", and that is not what it means. Across all four stages there is not one `return false`. Every path out of every implementation returns true, including the ones that have just caught an exception.

It means "I have finished with this item", and failing counts as finishing.

That sounds like a dodge until you look at what a failed item does instead:

```csharp
catch (Exception ex)
{
    currentRow.Result.StatusId = (int)RowStatus.Failed;
    currentRow.Result.ErrorMessage = "An error has occurred see Exception Log";

    _service.SaveStatusFromException(currentRow.Result);
    LogException(ex, currentRow?.Result?.Reference);
    currentRow.Progress?.Report(Progress.Create(..., "Not imported"));
    return true;
}
```

The failure is written onto the item, persisted, logged and reported. Then the method returns true, the item is dequeued, and the stage moves on to the next one.

The thing that does not happen is the line that appears at the end of the success path:

```csharp
NextProcessor?.EnQueueItem(currentRow);
```

**That is the whole mechanism.** Forwarding is what is conditional, not dequeuing. A row that failed to convert is finished with as far as the converter is concerned, and it is simply never handed to the writer. It stops where it died, carrying the reason it died.

Which is the thing that separates the two shapes more sharply than anything else in this post. In the search chain, a handler that cannot cope passes the request on, because the next one might do better. In a pipeline there is no next candidate, only a next step, and a next step has nothing useful to do with a row that was never converted. So failure does not move forward. It stops, and it leaves a note.

There is a stop-everything version of the same idea too:

```csharp
lock (LockObj)
{
    if (CancelAll)
    {
        currentRow.Result.StatusId = (int)RowStatus.Failed;
        currentRow.Result.ErrorMessage = "Cancelled by user";
        // ...save, report...
        return true;
    }
}
```

Checked at the top of every item, under a lock because the stages are running concurrently. When somebody cancels an import, each stage drains its queue marking everything cancelled rather than halting mid-flight, and nothing gets forwarded. The pipeline empties itself instead of stopping dead, which matters when four stages are in motion and some of them are part way through writing to a database.

## What does a stage do when it has nothing to do?

That question only exists in a pipeline. In the search chain a handler is either being asked something or it isn't, and in between it does not exist as far as the code is concerned. Here a stage can have an empty queue and still have work coming, so somebody has to decide what it does in the gap.

This one sleeps:

```csharp
// this one pauses the outer loop so it doesnt just loop and hammer the processor
if (!_allItemsAddedToQueue)
{
    Thread.Sleep(500);
}
```

Without it the outer loop spins on an empty queue burning a core for nothing, which is what the comment is about. The cost of sleeping instead is latency: a stage that runs out of work waits up to half a second before it notices more has turned up. A blocking collection or a channel answers the same question a different way, with the consumer waiting on the queue itself so it wakes as soon as something lands.

There is also no limit on how much can pile up in any of those queues. If reading is fast and saving is slow, the queue between them grows until the import finishes or the process runs out of memory. That is a thing a pipeline has to get right and a chain never has to think about, because a chain only ever holds one request at a time.

## How to tell which one you have

The wiring will not tell you. `SetNext` is in both, and so is a field holding the successor, and so is a line of objects assembled at startup.

Ask whether any link is allowed to end it.

If a link can say "mine, and nobody else needs to look", it is Chain of Responsibility. The links are candidates. Most do nothing on any given request. You are finding the one that applies without the caller knowing which it will be, and adding a link is cheap because the others neither know nor care.

If every link runs, and the output of one is the input of the next, it is a pipeline. The links are steps. Skipping one is a bug, not an optimisation. And because the data keeps moving, you inherit a set of problems a chain never has: when does it end, what happens when one stage falls behind, what happens when one stage fails halfway through, and how do stages share anything without becoming a tangle.

Those problems are not a reason to avoid pipelines. They are the price of streaming, and streaming is often worth it. They are a reason to know which one you have built, because solving a pipeline problem with chain thinking does not work.

## In short

Two bits of my own code, in two different jobs, with wiring you could not tell apart. One is a pattern about choosing. The other is a pattern about sequence.

The giveaway is what ends it. A chain ends when a link succeeds, and any link can be the one. A pipeline ends when the work runs out, or when something goes wrong and an item stops moving forward. Both of them stop. They stop for completely different reasons, and everything awkward about pipelines follows from that.

Next time, the one most .NET developers actually picture when they hear the word chain, which is neither of these. It is middleware, and the thing that makes it different is what happens on the way back out.
