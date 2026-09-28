---
layout: post
published: false
title: Your future-dated Jekyll post is never going to publish
pillar: jekyll
excerpt: Date a post ahead on GitHub Pages and it doesn't appear late, it never appears at all. Two sensible behaviours combining into one silent trap, and the scheduled rebuild that fixes it.
tags:
  - jekyll
  - githubpages
---

Say you write a post on a Sunday and you'd rather it went out on the Tuesday, so you name the file `2026-10-06-whatever.md` and push it.

Tuesday comes. Nothing. Wednesday, still nothing. The file is in the repo, the build is green, and the post simply isn't on the site. No error, no warning, no mention of it anywhere.

This catches people who never meant to schedule anything, too. Fat-finger a date in a filename and you get exactly the same silence.

Two perfectly reasonable behaviours are combining into one unhelpful result.

## Jekyll hides it on purpose

Jekyll has a `future` setting, and it defaults to `false`. Any post dated later than the moment of the build is left out entirely. That's sensible: it's how you write ahead without publishing ahead.

So on Sunday, when you push, Jekyll looks at your Tuesday post, decides it isn't due yet, and quietly drops it. Correct behaviour, and it tells you nothing, because there's nothing wrong.

## GitHub Pages only builds when you push

Here's the half that turns a feature into a trap.

Jekyll's exclusion isn't permanent. The post is only skipped because *this particular build* happened before its date. Build again on Tuesday and it appears.

Except nothing builds on Tuesday. GitHub Pages rebuilds when you push, and only when you push. Tuesday arrives, no push, no build, and the site stays exactly as it was on Sunday. The post isn't waiting to go out. Nothing has scheduled it, because nothing in the chain knows it's supposed to exist yet.

You can prove it by pushing anything at all on the Wednesday. A typo fix in an unrelated file will do it. The site rebuilds, Jekyll sees a post whose date has passed, and out it comes. Which is a peculiar way to run a publishing schedule.

## Doing the build yourself

The fix isn't a plugin or a setting. You need something to trigger a build on the day, and the simplest something is a scheduled GitHub Action.

That means taking over the build rather than letting Pages do it for you. It's less of a change than it sounds: Pages can deploy from an Action instead of from a branch, and the Action runs the same `jekyll build` you'd run locally.

```yaml
name: Build and deploy

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 7 * * *'
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.3'
          bundler: '2.7.2'
      - run: bundle install
      - uses: actions/configure-pages@v5
      - run: bundle exec jekyll build
        env:
          JEKYLL_ENV: production
      - uses: actions/upload-pages-artifact@v3

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

Three triggers. `push` keeps the behaviour you already had. `schedule` runs it every morning whether or not you've touched anything, which is the bit that was missing. `workflow_dispatch` puts a button in the Actions tab for when you want it out now.

The one thing you have to do by hand is in the repository settings: **Settings → Pages → Build and deployment → Source**, changed from *Deploy from a branch* to *GitHub Actions*. Until you do that the build will run and the deploy will fail, which is a confusing five minutes if you weren't expecting it.

The `bundler: '2.7.2'` line is worth a word. My theme's gemspec asks for bundler `~> 2.1`, and recent Ruby ships with 3.x, so without pinning it the install fails with a version conflict. If your build dies on `Could not find compatible versions`, that's why.

## Dates are read in UTC unless you say otherwise

One more thing that will make a post appear at an odd time. Jekyll reads dates in UTC by default, so in British Summer Time a post dated today technically becomes eligible at 1am rather than midnight. Not a disaster, but easy to fix:

```yaml
timezone: Europe/London
```

Now a date means what it looks like it means.

## What you get, and what you don't

Once it's in, a post dated ahead goes live on its date with nothing from you. Write four on a wet afternoon, date them a week apart, push once, and that's a month sorted.

Two things not to expect:

- **Cron is not punctual.** GitHub runs scheduled jobs when it has capacity, and under load that can be a quarter of an hour late or worse. Fine for a blog, useless if you need something on the hour.
- **Scheduled workflows switch themselves off.** Sixty days without activity in the repo and GitHub disables the schedule and emails you about it. Any push turns it back on. If you're pushing regularly you'll never see it, but it's a nasty surprise if you're relying on a queue draining while you're away.

## In short

A future-dated post on GitHub Pages doesn't publish late, it doesn't publish at all, because the thing that would notice its date only runs when you push. Jekyll is right to hide it and Pages is right not to rebuild for no reason. It's the gap between them that gets you.

Give something the job of rebuilding once a day and the gap closes.
