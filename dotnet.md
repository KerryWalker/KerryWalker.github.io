---
layout: page
title: .NET & SQL
sidebar_link: true
excerpt: Design patterns explained with real production code, .NET performance in plain English, and the SQL Server gotchas that cost me an afternoon.
---

Twenty-odd years of writing line-of-business software in .NET, and this is where I write up the bits worth passing on: the patterns I use every day and what they're actually called, the performance problems I keep finding in code reviews, and the occasional gotcha that ruined a Friday.

No toy examples. Everything here comes from code that shipped.

<h2 id="design-patterns">Design patterns in real code</h2>

Most developers are already using half the Gang of Four catalogue without knowing the names. This series takes the patterns one at a time, each explained with an everyday comparison and a real example out of an application I work on.

{% assign pattern_posts = site.posts | where: "pillar", "patterns" | sort: "date" %}
<ol>
{% for post in pattern_posts %}  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}</ol>

## .NET performance in plain English

Why the code feels instant on your machine and falls over on real data, explained without the computer science.

{% assign performance_posts = site.posts | where: "pillar", "performance" | sort: "date" %}
<ul>
{% for post in performance_posts %}  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}</ul>

## Gotchas

The ones that look right, run clean, and quietly give you the wrong answer.

{% assign gotcha_posts = site.posts | where: "pillar", "gotchas" | sort: "date" %}
<ul>
{% for post in gotcha_posts %}  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}</ul>

## Building this site

Jekyll and GitHub Pages bits, mostly written up because I'd forget them otherwise.

{% assign jekyll_posts = site.posts | where: "pillar", "jekyll" | sort: "date" %}
<ul>
{% for post in jekyll_posts %}  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}</ul>
