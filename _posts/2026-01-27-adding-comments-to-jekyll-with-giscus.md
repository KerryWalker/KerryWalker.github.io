---
layout: post
title: Adding Comments to a Jekyll Blog with Giscus
excerpt: How to set up Giscus for comments on a static Jekyll site, using GitHub Discussions as the backend.
tags:
  - jekyll
  - github-pages
  - giscus
---

Jekyll is a static site generator, so there's no built-in way to handle comments. The traditional solution was Disqus, but it comes with ads and tracking. I wanted something cleaner.

Enter Giscus — a comments system powered by GitHub Discussions. Comments are stored in your repo, there are no ads, and since it uses GitHub authentication, it tends to attract higher quality comments (especially on tech blogs where readers likely have GitHub accounts anyway).

## Why Giscus Over Other Options?

There are a few choices for Jekyll comments:

- **Disqus** — widely used but has ads and privacy concerns
- **Utterances** — uses GitHub Issues (works well, but Issues feel like the wrong place for comments)
- **Giscus** — uses GitHub Discussions (purpose-built for conversations)
- **Staticman** — comments become commits to your repo (more complex setup)

I went with Giscus because Discussions feels like the right fit, and the setup is simple.

## Step 1: Enable Discussions on Your Repo

1. Go to your repository on GitHub
2. Click **Settings**
3. Scroll to **Features**
4. Tick **Discussions**

If prompted, click through to set up the Discussions tab.

## Step 2: Install the Giscus GitHub App

1. Go to [github.com/apps/giscus](https://github.com/apps/giscus)
2. Click **Install**
3. Choose **Only select repositories**
4. Select your blog repository
5. Click **Install**

This allows Giscus to create and manage discussions on your behalf.

## Step 3: Configure Giscus

1. Go to [giscus.app](https://giscus.app)
2. Fill in the configuration:
   - **Repository:** `yourusername/yourrepo`
   - **Page ↔ Discussions Mapping:** I recommend **pathname** (creates one discussion per blog post based on its URL)
   - **Discussion Category:** Select **Announcements** or create a dedicated "Comments" category
   - **Features:** Enable reactions, lazy loading, etc. as you prefer
   - **Theme:** Choose one that matches your site, or use `preferred_color_scheme` to follow the user's system settings

3. Scroll down — giscus.app generates a `<script>` snippet. Copy it.

It will look something like this:

```html
<script src="https://giscus.app/client.js"
        data-repo="yourusername/yourrepo"
        data-repo-id="R_xxxxx"
        data-category="Announcements"
        data-category-id="DIC_xxxxx"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="en"
        crossorigin="anonymous"
        async>
</script>
```

## Step 4: Add to Your Jekyll Site

Create or replace `_includes/comments.html` with the script you copied:

```html
<div class="comments">
  <h2>Comments</h2>
  <script src="https://giscus.app/client.js"
          data-repo="yourusername/yourrepo"
          data-repo-id="R_xxxxx"
          data-category="Announcements"
          data-category-id="DIC_xxxxx"
          data-mapping="pathname"
          data-strict="0"
          data-reactions-enabled="1"
          data-emit-metadata="0"
          data-input-position="bottom"
          data-theme="preferred_color_scheme"
          data-lang="en"
          crossorigin="anonymous"
          async>
  </script>
</div>
```

If you're using the Hydeout theme, it already includes `comments.html` in the post layout, so this should just work.

## Step 5: Check Your Post Layout

Make sure your `_layouts/post.html` includes the comments partial. Look for something like:

```liquid
{% include comments.html %}
```

If it's not there, add it where you want comments to appear (usually at the end of the post content).

## Step 6: Deploy

```bash
git add _includes/comments.html
git commit -m "Add Giscus comments"
git push
```

After GitHub Pages rebuilds, visit any blog post and you should see the comments section at the bottom.

## Managing Comments

All comments appear in your repository's **Discussions** tab. You can:

- Reply to comments directly on GitHub
- Pin important discussions
- Lock discussions if needed
- Delete spam

Since it's all in GitHub, you have full control over your data.

## Theming

If you want the comments to match your site's dark/light mode, the `preferred_color_scheme` theme works well. Alternatively, giscus offers specific themes like `light`, `dark`, `dark_dimmed`, and others.

You can also create a custom theme if you want precise control — see the [giscus documentation](https://github.com/giscus/giscus/blob/main/ADVANCED-USAGE.md) for details.
