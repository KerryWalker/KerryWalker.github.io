---
layout: post
published: false
title: Switching from Google Analytics to Cloudflare Web Analytics
excerpt: Google Analytics wasn't showing any data thanks to ad blockers. Cloudflare Web Analytics is a better fit if your domain is already on Cloudflare.
tags:
  - jekyll
  - github-pages
  - cloudflare
---

I recently [added Google Analytics](/2026/02/07/adding-google-analytics-to-jekyll/) to this blog. The setup was straightforward, the tracking code was firing correctly, but nothing was showing up in the realtime reports. No visitors, no page views, nothing.

After some investigation, the issue was almost certainly ad blockers. Google Analytics is on every single ad blocker list out there - uBlock Origin, Brave, AdGuard, you name it. The script gets blocked before it can send any data back to Google. Given how many people run ad blockers these days, there's a good chance a significant chunk of visitors are simply invisible to GA4.

## Cloudflare Web Analytics

My domain is already on Cloudflare - I use it for DNS and the `cloudflared` tunnel to access my Home Assistant instance. So it made sense to use Cloudflare's free Web Analytics instead.

The key advantage is that Cloudflare injects the analytics at the edge, through their proxy. The tracking script comes from your own domain rather than from `googletagmanager.com`, so ad blockers don't recognise it as a tracking script and don't block it.

If I'd thought about it before setting up GA4, I probably would have gone straight to Cloudflare in the first place.

## Setting It Up

If your domain is already on Cloudflare, it takes about a minute:

1. Log into the [Cloudflare dashboard](https://dash.cloudflare.com)
2. Select your domain
3. Go to **Analytics & Logs** > **Web Analytics**
4. Enable it

That's it. No code changes, no config files to update, no scripts to add to your site. Cloudflare handles everything automatically.

## What You Get

It's simpler than GA4 - no user flows, events, or realtime view - but it covers the basics:

- **Page views** and **visits** over time
- **Top pages** - which posts are getting the most traffic
- **Referrers** - where visitors are coming from
- **Countries**, **browsers**, and **devices**

For a personal blog where I just want to know if anyone is actually reading my posts, that's more than enough.

## What About GA4?

I've left the Google Analytics tracking code in place for now. It doesn't hurt anything being there, and it'll still capture data from visitors who don't have ad blockers. But Cloudflare is where I'll be checking for actual numbers.
