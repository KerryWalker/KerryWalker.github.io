---
layout: post
title: Building self-updating pillar pages in Jekyll with a front matter field
excerpt: How I made a topic index page that builds itself in Jekyll, using a custom pillar front matter field and a Liquid loop, so new posts appear automatically and links never break.
tags:
  - jekyll
---

I recently added an [index page](/home-automation.html) that groups my home automation and energy posts into sections. The first version was a hand-written list of links, and it had two problems: I had to remember to update it every time I wrote a post, and I got the URL format wrong so half the links 404'd.

The fix was to let Jekyll build the list itself, the same idea as my [related posts](/2026/04/22/jekyll-related-posts-actually-related.html) that work off tags. Here is how it went.

## A front matter field, not tags

My first thought was to group by tag, but my tags overlap. A post about Predbat charging a Tesla is tagged both `predbat` and `tesla`, so it would appear in two sections at once. I wanted each post in exactly one section, so I added a single custom field instead:

```yaml
---
layout: post
pillar: energy
title: Setting up Predbat with Solar Assistant
---
```

Each post gets `pillar: energy`, `pillar: ev` or `pillar: dashboards`.

## The loop

On the index page I filter the posts by that field and loop over what comes back:

{% raw %}
```liquid
{% assign energy_posts = site.posts | where: "pillar", "energy" | sort: "date" %}
<ul>
{% for post in energy_posts %}  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}</ul>
```
{% endraw %}

`where` pulls just the posts in that pillar, `sort: "date"` puts them in order, and {% raw %}`{{ post.url }}`{% endraw %} generates the correct link every time. That last part matters: hardcoding the URLs is what broke my first attempt, and letting Jekyll output the link means I cannot get the format wrong again.

## Adding to it

The whole thing now looks after itself:

- **New post in a section?** Add `pillar: energy` (or whichever) to its front matter and it shows up on its own.
- **New section?** Copy those few lines of Liquid, change the pillar value and the heading.
- **A whole new topic?** Duplicate the page, say a `dev.md` for development posts, and give those posts `pillar: dev`.

No more hand-written lists, and no more broken links.
