# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog at www.kerrywalker.uk built with Jekyll and the Hydeout theme (v4.2.0), hosted on GitHub Pages.

## Common Commands

```bash
# Local development server (http://localhost:4000)
bundle exec jekyll serve

# Build site
bundle exec jekyll build

# Lint SCSS
npm run stylelint
```

## Blog Post Format

Posts go in `_posts/` with filename format `YYYY-MM-DD-title-slug.md`:

```yaml
---
layout: post
title: Post Title Here
excerpt: Brief description for previews and SEO
tags:
  - home-assistant
  - predbat
---

Content in Markdown...
```

## Blog Style

- British English, first person, practical tone. Posts open with the setup/problem, use short `##` sections, and end with a summary.
- Round numbers in prose ("around 20,000 files"), not exact counts ("19,388 files"). Keep exact values only where precision is the point (prices, config values, measurements).
- Drafts are flagged with `published: false` in front matter — there is no `_drafts/` folder. Remove the flag (or set `true`) to publish.
- Home automation/energy posts carry a `pillar:` field (`energy`, `ev`, `dashboards`) that feeds the index page at `home-automation.md`. Dev posts have no pillar.

## Architecture

- **Theme**: Hydeout (Jekyll theme) - layouts in `_layouts/`, partials in `_includes/`, styles in `_sass/hydeout/`
- **Configuration**: `_config.yml` (site settings, plugins, pagination)
- **Custom styling**: `assets/css/main.scss` (green theme color: #90a959)
- **Comments**: Giscus (GitHub Discussions-based) configured in `_includes/comments.html`
- **Analytics**: GA4 in `_includes/google-analytics.html` (production only)

## Git Conventions

- Do not add Co-Authored-By lines to commit messages.

## Customization Hooks

The theme provides empty includes for customization without modifying theme files:
- `_includes/custom-head.html` - Additional head tags
- `_includes/custom-foot.html` - Footer scripts
- `_includes/custom-nav-links.html` - Extra sidebar navigation
- `_includes/custom-icon-links.html` - Extra sidebar icons
