---
layout: post
published: true
title: Move a Git local commit to a new branch
tags:
  - git
---

I often work in the main branch and then create a new branch when I'm ready to commit changes, I find this easier than creating a new branch when I start something but it does means I sometimes commit changes to the wrong branch.

To move these changes to a new branch I run the following commands in a git terminal.

{% highlight console %}
git branch new-branch-name
git switch new-branch-name
git branch --force development origin/development
{% endhighlight %}

1. Creates a new branch pointing at the current commit.
2. Makes that branch the current branch.
3. Resets the development branch back to origin/development, removing the commit that was added by mistake without changing the new branch.
