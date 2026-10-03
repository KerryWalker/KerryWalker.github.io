---
layout: post
published: false
title: "The Chain That Can't Stop: When a Pattern Is Only Wearing the Clothes"
pillar: patterns
series: design-patterns
excerpt: "Two bits of code I wrote, in two different jobs, that look almost identical. One is Chain of Responsibility. The other is a pipeline, and finding out what that difference costs took me a while."
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
parseProcessor.SetNextProcessor(convertProcessor);
convertProcessor.SetNextProcessor(saveProcessor);
saveProcessor.SetNextProcessor(indexProcessor);
```

Put that next to the search chain wiring and you would be hard pushed to say which was which. Same method name, same shape, same idea of a line you read top to bottom.

So what makes one a Chain of Responsibility and the other not?

## Nothing in it is allowed to stop

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

Each stage owns a queue, and all four run as concurrent tasks. Parsing is still chewing through the file while converting works on rows it has already emitted and saving writes out what came through before that. That is the actual reason to build it this way, and it is worth saying plainly because it is the only reason that justifies the complexity: **you do not want to hold the whole file in memory three times over.** Parse everything, then convert everything, then save everything, and a big import needs three copies of a large file resident at once. Let them overlap and you need the three stages plus whatever is in flight between them.

## What I got wrong

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

Look at an item for a moment, deal with it, and only then take it off the queue. Peek, process, dequeue. The intent was that an item is not removed until it has genuinely been handled.

Now follow what happens when `ProcessIndividualItem` returns false.

Nothing gets dequeued. The count does not change. The loop goes round, peeks at the same item, fails on it again, and does that as fast as the processor will let it, forever.

There is a boolean on that method that you must never return false from. It sits in the signature of an abstract method, looking for all the world like a decision a subclass is invited to make, and making it is the one thing a subclass must not do. Nothing in the code says so. I know because I wrote it, and I only know now because I went back and read it properly for this.

That is a different class of mistake from the ones in the search chain. If a searcher misbehaves there, you get a worse search result. One handler fails, the chain carries on, and the user sees nothing found instead of something found. A pipeline has no next candidate to fall back to, so a stage that can't cope has nowhere to put the problem.

## The queue nobody is watching

The other thing in there is a confession with a comment attached:

```csharp
// this one pauses the outer loop so it doesnt just loop and hammer the processor
if (!_allItemsAddedToQueue)
{
    Thread.Sleep(500);
}
```

When a stage has emptied its queue but the stage above is still producing, it has nothing to do. So it sleeps for half a second and looks again.

It works. It is also polling, and the comment is me noticing that and patching it rather than fixing it. A stage that finishes its work has to wait up to 500ms before it notices more has arrived, and across four stages that adds up on a small import. The right answer is a blocking collection or a channel, where a consumer waits on the queue itself and wakes the moment something lands.

I would also point out that there is no limit on how much can pile up in any of those queues. If parsing is fast and saving is slow, the queue between them grows until the import finishes or the process runs out of memory. That is the thing a pipeline has to get right and a chain never has to think about, because a chain only ever holds one request at a time.

## And the stages talk behind its back

Every stage gets handed the same `Hashtable`.

```csharp
public Hashtable PropertyBag { get; set; }
```

If two stages need to agree on something, they put it in there under a string key. It works, and it is the part I would change first. The real interface between two stages is invisible: you find it by searching for string literals and hoping you found them all. I could not tell you today which stages genuinely depend on which, and a typed context object passed down the line would have told me on day one.

Notice that the search chain needed nothing like this either. Handlers in a chain are independent by construction, because only one of them is going to run.

## How to tell which one you have

The wiring will not tell you. `SetNext` is in both, and so is a field holding the successor, and so is a line of objects assembled at startup.

Ask whether any link is allowed to end it.

If a link can say "mine, and nobody else needs to look", it is Chain of Responsibility. The links are candidates. Most do nothing on any given request. You are finding the one that applies without the caller knowing which it will be, and adding a link is cheap because the others neither know nor care.

If every link runs, and the output of one is the input of the next, it is a pipeline. The links are steps. Skipping one is a bug, not an optimisation. And because the data keeps moving, you inherit a set of problems a chain never has: when does it end, what happens when one stage falls behind, what happens when one stage fails halfway through, and how do stages share anything without becoming a tangle.

Those problems are not a reason to avoid pipelines. They are the price of streaming, and streaming is often worth it. They are a reason to know which one you have built, because solving a pipeline problem with chain thinking does not work.

## In short

Two bits of my own code, in two different jobs, with wiring you could not tell apart. One is a pattern about choosing. The other is a pattern about sequence.

The giveaway is the terminating condition. A chain has one, and any link can trigger it. A pipeline hasn't got one at all, it has an *ending*, which somebody has to signal, and everything awkward about pipelines follows from that.

Next time, the one most .NET developers actually picture when they hear the word chain, which is neither of these. It is middleware, and the thing that makes it different is what happens on the way back out.
