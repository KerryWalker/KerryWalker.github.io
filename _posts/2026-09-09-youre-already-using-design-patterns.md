---
layout: post
title: You're Already Using Design Patterns, You Just Don't Know Their Names
excerpt: Fifteen years ago I got asked about design patterns in an interview and didn't have a clue. Then I read the book and realised I'd been using them every day. Here's the reframe that finally made them make sense.
tags:
  - design-patterns
  - csharp
  - dotnet
---

About fifteen years ago I sat in an interview and was asked about design patterns. I didn't have a clue. The interviewer said a few names, Strategy, Observer, Factory, and waited, and I had nothing. I'd been writing code for years by that point and I could feel how bad it looked to have never even heard of the words. (Yes, I got the job :)).

So afterwards I bought Head First Design Patterns to fill the gap. And the strange thing was, the further I got into it, the more I realised I'd already been using half of these things for years. I just never knew they had names.

## The Names Are the Point

That's the bit nobody tells you. A design pattern isn't some clever trick you have to go and learn. Most of them are just a name someone gave to a solution people kept arriving at anyway.

And the name is the actual value. Once you both know what a "strategy" is, I can say "let's make that a strategy" and you instantly picture the shape of it, no whiteboard needed. The pattern is a shortcut for a conversation. That's why interviewers ask about them. They're not checking you can recite the book, they're checking you can talk to the rest of the team.

## Why the Book Didn't Click Straight Away

Here's the trap I fell into. A book of patterns is a catalogue, and reading a catalogue cover to cover is a terrible way to learn. You end up staring at a solution before you've ever felt the problem it fixes, so none of it sticks.

You don't learn a pattern by memorising it. You recognise it when you walk into the problem it solves, and that only clicks once someone points at something you've already built and tells you it has a name.

## You've Probably Already Written a Strategy

Take the one the interviewer led with. Here's an example straight out of an app I work on.

There's one reusable "find" screen: a search form, a results grid, paging, sorting, saved layouts, the lot. Written once. But every screen that uses it is searching for something different. Finding a customer is nothing like finding an invoice, different search boxes, different columns, different data underneath. So the find screen itself knows nothing about customers or invoices. It's handed something that does:

```csharp
public interface IFindViewModel
{
    IReadOnlyList<Column> Columns { get; }
    IReadOnlyList<SearchField> SearchFields { get; }
    Task GetData();
}
```

There's a `FindCustomersViewModel` that implements it, a `FindInvoicesViewModel`, an orders one, and a couple of dozen more. The find screen takes an `IFindViewModel` and just talks to the interface. Hand it the customers one and it's a customer search. Hand it the invoices one and it's an invoice search. The screen never changes.

That's the Strategy pattern. The find screen is the bit that stays the same, and each viewmodel is an interchangeable strategy that decides what "find" actually means on that page. If you've ever injected an interface so you could swap the implementation, you've written dozens of these without once thinking of the word.

That was the moment it turned around for me. Not "here's a new thing to learn", but "oh, *that's* what that's called."

(If you're wondering how each page gets handed the right one out of the couple of dozen, that's keyed dependency injection doing the choosing. It's a nice trick, and a post all of its own.)

## Where This Is Going

So I'm not going to hand you a catalogue. Over the next few posts I'll take the patterns worth knowing one at a time, with an everyday comparison and, more usefully, where you've almost certainly already met them in .NET. Streams, events, dependency injection, the framework is full of them.

Don't set out to memorise all twenty-odd. Most of the time you only need to recognise the handful you keep bumping into, and be able to put a name to what you're already doing. That's the part the book couldn't give me, and it's the part that would have got me through that interview.
