---
layout: post
title: Automating a WSL Ruby Dev Environment with AutoHotKey
excerpt: How I automated opening three Ubuntu terminal sessions, running the right commands in each, and launching a browser — all with a single keyboard shortcut.
tags:
  - autohotkey
  - wsl
  - ruby
---

I've started working on a Ruby project that runs in WSL. Every time I want to spin up the dev environment I need to open three separate Ubuntu terminal sessions, navigate to the project directory in each, run a different command in each one, then open a browser. It's not complicated but it's tedious to do multiple times a day.

## What Needs to Happen

Three Ubuntu tabs in Windows Terminal, all in the same project directory, each running a different process:

1. `bin/sidekiq` — background job processor
2. `bin/rails server` — the web app server
3. `foreman start -f Procfile.dev` — runs additional processes like CSS/JS bundling

Then open `http://localhost:3000` in a browser.

## Opening Ubuntu Tabs from the Command Line

Windows Terminal supports a `-p` flag to specify a profile, and you can chain multiple tabs with `new-tab`. In AutoHotKey v2, semicolons need escaping with a backtick since AHK treats them as comments:

```
Run "wt.exe -p Ubuntu `; new-tab -p Ubuntu `; new-tab -p Ubuntu"
```

This opens a single Windows Terminal window with three Ubuntu tabs.

## The Full Script

```
#Requires AutoHotkey v2.0

^!w:: {
    ; Open 3 Ubuntu tabs in one Windows Terminal window
    Run "wt.exe -p Ubuntu `; new-tab -p Ubuntu `; new-tab -p Ubuntu"
    Sleep 2000

    ; Tab 1 (already focused) — sidekiq
    SendText "cd ~/projects/myapp && bin/sidekiq"
    Send "{Enter}"

    ; Switch to tab 2
    Sleep 500
    Send "^{Tab}"
    Sleep 300
    SendText "cd ~/projects/myapp && bin/rails server"
    Send "{Enter}"

    ; Switch to tab 3
    Sleep 500
    Send "^{Tab}"
    Sleep 300
    SendText "cd ~/projects/myapp && foreman start -f Procfile.dev"
    Send "{Enter}"

    ; Open the app in the browser
    Sleep 2000
    Run "http://localhost:3000"
}
```

The hotkey is Ctrl+Alt+W. A few things worth noting:

- `SendText` is used instead of `Send` for the commands so that special characters like `/` and `-` aren't interpreted as AHK hotkeys.
- `Ctrl+Tab` switches between tabs in Windows Terminal.
- The `&&` chains the `cd` and the run command together into a single line per tab.
- The `Sleep 2000` at the start gives WSL a moment to initialise. If your machine is slower you might need to increase this.
- The final `Sleep 2000` before opening the browser gives the Rails server time to start. Increase it if you see a connection refused error in the browser.

## Mapping to a Key

I've got this mapped to a macro on my Keychron V6 Max so I can start the whole environment with a single key press. See my [earlier post](/2026/02/12/spotify-connect-shortcut-autohotkey-keychron.html) on setting up Keychron macros with AutoHotKey if you want to do the same.

## Result

One key press and I'm ready to code. The three terminals spin up, the processes start, and the browser opens to the app. It's saved me a surprising amount of friction throughout the day.
