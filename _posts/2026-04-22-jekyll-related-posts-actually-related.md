---
layout: post
title: Making Jekyll's Related Posts Actually Related
excerpt: Jekyll's site.related_posts doesn't show related posts - it shows recent ones. Here's how to fix it with tag-based matching.
tags:
  - jekyll
  - github-pages
---

I noticed the "Related Posts" section at the bottom of my posts wasn't showing related posts at all. A post about dropper posts was showing family dashboard articles. Not exactly helpful.

## The problem

Jekyll has a built-in variable called `site.related_posts` that sounds like exactly what you'd want. The default template I was using did this:

```liquid
{% raw %}{% for post in site.related_posts limit:3 %}
  ...
{% endfor %}{% endraw %}
```

Turns out `site.related_posts` is misleading. By default it just returns the 10 most recent posts - nothing to do with actual relatedness.

Jekyll can calculate proper related posts using LSI (Latent Semantic Indexing), but that requires a gem that GitHub Pages doesn't support. So if you're hosting on GitHub Pages, `site.related_posts` is essentially useless.

## The fix

The solution is to match posts by tags instead. Loop through all posts, count how many tags they share with the current post, and show the ones with the most matches.

Here's what I replaced the template with:

```liquid
{% raw %}{% assign related = "" | split: "" %}

{% for post in site.posts %}
  {% if post.url != page.url %}
    {% assign shared_tags = 0 %}
    {% for tag in page.tags %}
      {% if post.tags contains tag %}
        {% assign shared_tags = shared_tags | plus: 1 %}
      {% endif %}
    {% endfor %}
    {% if shared_tags > 0 %}
      {% assign related = related | push: post %}
    {% endif %}
  {% endif %}
{% endfor %}

{% comment %}
  Sort by number of shared tags (descending).
  Liquid doesn't support custom sort, so we do multiple passes.
{% endcomment %}

{% assign sorted_related = "" | split: "" %}
{% for i in (1..10) reversed %}
  {% assign match_count = i %}
  {% for post in related %}
    {% assign shared_tags = 0 %}
    {% for tag in page.tags %}
      {% if post.tags contains tag %}
        {% assign shared_tags = shared_tags | plus: 1 %}
      {% endif %}
    {% endfor %}
    {% if shared_tags == match_count %}
      {% unless sorted_related contains post %}
        {% assign sorted_related = sorted_related | push: post %}
      {% endunless %}
    {% endif %}
  {% endfor %}
{% endfor %}

{% if sorted_related.size > 0 %}
<section class="related">
  <h2>Related Posts</h2>
  <ul class="posts-list">
    {% for post in sorted_related limit:3 %}
      <li>
        <h3>
          <a href="{{ post.url | relative_url }}">
            {{ post.title }}
            <small>{{ post.date | date_to_string }}</small>
          </a>
        </h3>
      </li>
    {% endfor %}
  </ul>
</section>
{% endif %}{% endraw %}
```

It's not elegant - Liquid doesn't have custom sort functions, so we have to do multiple passes to sort by match count. But it works, and the performance hit only happens at build time, not for visitors.

## Styling

The default styling made the related posts quite large. I added some CSS to shrink them down:

```scss
.related h2 {
  font-size: 1.2rem;
}
.related .posts-list h3 {
  font-size: 0.9rem;
  margin: 0.5rem 0;
}
```

Now the cycling posts link to each other, the Home Assistant posts link together, and the section doesn't dominate the page.
