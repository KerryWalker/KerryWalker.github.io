---
layout: post
title: Async/Await Does Not Create New Threads
excerpt: The single most common thing people get wrong about async/await in .NET, explained with a trip to the garage. Second in a short series on .NET performance problems.
tags:
  - csharp
  - dotnet
  - async
---

You see that heading a lot when you're trying to work out what async/await actually is in .NET. I've worked with several "senior developers" who still believed all those keywords did was spin up a new thread. The trouble is there aren't many explanations that tie it to something from the real world. Here's mine.

This is the second in a short series about performance problems I keep meeting in .NET code, explained in plain English.

## The Garage

Imagine you're taking your car in for its MOT, or for some new tyres. You get to the garage, walk up to the counter, and hand over your keys.

The moment you hand the keys over, you stop. You don't do anything. You don't look around, you don't move, you don't go anywhere. You barely even breathe. You just tick over, and you stay like that the entire time your car is being worked on.

That's synchronous IO in .NET. You hand over to Windows to read a file, or to SQL Server to fetch some data, and you wait there for the reply. The thread that's running your code is that person stood frozen at the counter. It's doing nothing useful, but it's still there, taking up space, unable to help anyone else.

## What Actually Happens

Now think about what you'd really do.

You hand over your keys and you go and sit down. You watch some telly, you go for a walk around town, maybe you even go home. You get on with your day. The person behind the counter takes your keys and deals with the next customer.

Sooner or later you get a call from the garage telling you your car is ready. You go back, pick it up, and carry on where you left off.

That's async IO in .NET. You hand over your keys and then you (a)wait while the other process does its thing. Your thread doesn't stand there frozen. It goes back to the pool and gets on with other work. When the file read or the database query finishes, the runtime picks the work back up and carries on.

No new thread was created. That's the whole point. The thread you already had was freed up to do something useful instead of standing at the counter doing nothing.

## In Code

The synchronous version holds the thread the whole time:

```csharp
var data = File.ReadAllText("bigfile.txt");
```

The async version hands over and lets the thread go:

```csharp
var data = await File.ReadAllTextAsync("bigfile.txt");
```

The line barely changes. But while that file is being read, the thread that was running your method isn't blocked waiting. It's off serving another request.

## Why It Matters

On your machine, with one user, you'll never notice the difference. Both versions get you your data.

But picture a web server handling a thousand requests, each one reading a file or waiting on the database. With synchronous IO you've got a thousand people stood frozen at the counter, one thread each, all doing nothing but waiting. Threads aren't free, and you'll run out of them long before you run out of actual work to do.

With async IO those threads go back to the pool the moment they start waiting, ready to serve someone else. The same handful of threads can keep hundreds of requests moving, because none of them are stood around doing nothing.

So no, async/await doesn't create new threads. If anything it's the opposite. It's about not tying up the thread you already have.
