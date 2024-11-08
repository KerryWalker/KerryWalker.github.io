---
layout: post
title: Git partial stash
tags:
  - git
---

I was recently working on a task but needed to switch to a different task but carried on working without stashing the changes from the original task before starting the new one. Later on I needed to create a new branch and wanted to create seperate stashes for each task.

Recent versions of git provide an easy way to do this with the `--staged` option:

To move these changes to a new branch I run the following commands in a git terminal.

{% highlight console %}
git add <some-files>                  # pick (stage) the files to stash
git stash save --staged 'my stash'    # stash only staged
{% endhighlight %}