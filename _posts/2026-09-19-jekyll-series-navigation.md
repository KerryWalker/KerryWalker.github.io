---
layout: post
published: false
title: Series navigation in Jekyll, without hand-maintaining the links
pillar: jekyll
excerpt: A blog shows posts newest first, which is the wrong order for a series. Here is how I added "Part 3 of 5" banners and previous/next links using a front matter field, a data file and about thirty lines of Liquid.
tags:
  - jekyll
---

A while back I added a [pillar page](/2026/07/14/jekyll-pillar-index-pages-with-front-matter.html) that groups posts into sections using a front matter field. That solved finding things by topic. Then I started writing a series where each post builds on the last, and hit a different problem.

A blog lists posts newest first. A series reads oldest first. So the front page shows my series backwards. Worse, most people never see the front page at all. They arrive from a search engine, land on part three, and nothing on it tells them there is a part one.

The pillar page does list them in the right order, but only if you find the pillar page. What I wanted was for the post itself to say where it sits.

## A field for the series

Same trick as the pillar field. Each post in the series gets one line:

```yaml
---
layout: post
title: I Called It a Factory Without Knowing If It Was One
pillar: patterns
series: design-patterns
---
```

That is enough to gather them: {% raw %}`site.posts | where: "series", page.series`{% endraw %} gives every part, and `sort: "date"` puts them in reading order.

## A data file for the name

I did not want the words "Design Patterns in Real Code" typed into twenty posts, because the day I reword it I would be editing twenty posts. So the slug points at an entry in `_data/series.yml`:

```yaml
design-patterns:
  name: Design Patterns in Real Code
  url: /dev.html#design-patterns
```

Anything in `_data` is loaded automatically and available as `site.data.<filename>`, so the include can look the slug up:

{% raw %}
```liquid
{% assign series_meta = site.data.series[page.series] %}
```
{% endraw %}

The name and the link to the full list now live in one place. A new series is three lines of YAML.

## Working out which part you are on

This is the fiddly bit. Liquid has no "give me the index of this item" filter, so you find your own position by looping over the list and remembering where you were:

{% raw %}
```liquid
{% assign series_posts = site.posts | where: "series", page.series | sort: "date" %}

{% assign current_index = 0 %}
{% for series_post in series_posts %}
  {% if series_post.url == page.url %}
    {% assign current_index = forloop.index0 %}
  {% endif %}
{% endfor %}

{% assign part_number = current_index | plus: 1 %}
```
{% endraw %}

`assign` inside a loop survives after it, which is what makes this work. Once you have the index, the neighbours are just arithmetic. Liquid is happy to take a variable as an array index:

{% raw %}
```liquid
{% assign previous_index = current_index | minus: 1 %}
{% if current_index > 0 %}
  {% assign previous_post = series_posts[previous_index] %}
  <a href="{{ previous_post.url | relative_url }}">{{ previous_post.title }}</a>
{% endif %}
```
{% endraw %}

Same again with `plus: 1` for the next one, guarded with {% raw %}`{% if next_index < series_posts.size %}`{% endraw %}.

## Where it goes

The whole thing sits in `_includes/series-nav.html` and gets called twice from the post layout, with a parameter to say which half to render:

{% raw %}
```liquid
{% include series-nav.html position="top" %}
<div class="post-body">{{ content }}</div>
{% include series-nav.html position="bottom" %}
```
{% endraw %}

At the top, a one-line banner: *Part 3 of 5 in Design Patterns in Real Code · Start at the beginning*. At the bottom, previous and next. The top one matters most, because it is the only one a reader sees before deciding whether they are in the right place.

The include starts with {% raw %}`{% if page.series %}`{% endraw %}, so every post without the field renders exactly as before.

## Two things I got for free

**Drafts stay hidden.** I keep unpublished posts as `published: false` in `_posts` rather than in a drafts folder, and Jekyll leaves those out of `site.posts` entirely. So a half-written part four is not counted, not linked, and does not turn the banner into "Part 3 of 6" before it exists. Publish it and it appears in both the count and the links with no further work.

**The count is honest.** Because the list is built at build time, "Part 3 of 5" always reflects what is actually on the site, not what I intended to write. I cannot promise a part six that never arrives.

## Adding another series

Three lines in `_data/series.yml`, then `series: whatever` in each post. No template changes, no lists to maintain, and the links cannot rot because Jekyll generates every one of them.

Which is the same reason the pillar page has looked after itself ever since I built it. The rule I keep relearning: if the site can work something out from the posts themselves, do not type it in by hand.
