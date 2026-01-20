---
layout: post
title: Adding Google Analytics to a Jekyll Blog on GitHub Pages
excerpt: A quick guide to setting up Google Analytics 4 on a Jekyll site using the Hydeout theme.
tags:
  - jekyll
  - github-pages
  - google-analytics
---

I wanted to add some basic analytics to this blog to see which posts were getting traffic. If you're using Jekyll with the Hydeout theme (or most other themes), it's surprisingly straightforward.

## Create a Google Analytics Property

First, you need a Google Analytics account and property:

1. Go to [analytics.google.com](https://analytics.google.com) and sign in
2. Click **Admin** (gear icon, bottom left)
3. Click **Create** → **Account**
4. Give it a name (e.g. "My Blog")
5. Click through the setup, entering a property name and your preferences
6. Accept the terms and create the property

## Set Up a Data Stream

Once your property is created:

1. Select **Web** as the platform
2. Enter your website URL (e.g. `https://yourdomain.com`)
3. Give the stream a name
4. Click **Create stream**

You'll now see your **Measurement ID** — it looks like `G-XXXXXXXXXX`. Copy this.

## Add It to Your Jekyll Site

The Hydeout theme (and many others) have built-in Google Analytics support. You just need to add one line to your `_config.yml`:

```yaml
google_analytics: G-XXXXXXXXXX
```

Replace `G-XXXXXXXXXX` with your actual Measurement ID.

That's it. The theme handles injecting the tracking script automatically.

## Deploy and Verify

Commit and push your changes:

```bash
git add _config.yml
git commit -m "Add Google Analytics"
git push
```

Wait a few minutes for GitHub Pages to rebuild, then:

1. Visit your site
2. Go to Google Analytics → **Reports** → **Realtime**
3. You should see yourself as an active user

## A Note on Data

Google Analytics 4 can take 24-48 hours before historical data appears in the standard reports. The Realtime view works immediately though, so use that to verify everything is working.

## If Your Theme Doesn't Support It

If your theme doesn't have built-in Google Analytics support, you can add the tracking code manually. Create or edit `_includes/custom-head.html` and add:

```html
{% if site.google_analytics %}
<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id={{ site.google_analytics }}"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', '{{ site.google_analytics }}');
</script>
{% endif %}
```

Then add `google_analytics: G-XXXXXXXXXX` to your `_config.yml` as before.
