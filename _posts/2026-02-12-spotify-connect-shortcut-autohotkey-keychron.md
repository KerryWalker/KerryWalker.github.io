---
layout: post
title: One-Key Spotify Connect with AutoHotKey and a Keychron Keyboard
excerpt: How I set up a single keyboard shortcut to open Spotify's Connect panel and select a specific speaker using AutoHotKey and a Keychron V6 Max.
tags:
  - autohotkey
  - spotify
  - keychron
---

I use Spotify a lot around the house, and I got tired of the little dance you have to do every time you want to switch audio to a specific speaker — open Spotify, find the Connect icon, click it, wait for the device list, then click the right speaker. I wanted one button on my keyboard to do the whole thing.

## The Setup

I'm on Windows 11 with a [Keychron V6 Max ISO](https://amzn.to/4kugJt6) keyboard and AutoHotKey v2 installed. The goal: press a single key and have Spotify switch playback to my office speaker automatically.

## Why Not Just Use a Keyboard Shortcut?

Spotify doesn't have a keyboard shortcut for the Connect to Device panel. I checked the [official shortcuts list](https://support.spotify.com/us/article/keyboard-shortcuts/) — it's not there. So this has to be done with UI automation.

## The Approach: ImageSearch in AutoHotKey

The idea is straightforward:

1. Focus Spotify
2. Find and click the Connect icon (the little speaker/screen icon in the playback bar)
3. Wait for the device panel to appear
4. Find and click the speaker name in the list

AutoHotKey's `ImageSearch` can locate a small screenshot on screen and click it. You take a tight crop of the UI element you want to click, save it as a PNG, and AHK finds it for you.

## Capturing the Images

You'll need two small screenshots saved in the same folder as your AHK script:

- **connect_button.png** — a tight crop of the Connect/Devices icon from Spotify's bottom playback bar. Use Win+Shift+S to snip it.
- **office.png** — a tight crop of the speaker name from the device list. Click the Connect icon first to open the panel, then snip the name.

Keep the crops tight with minimal background. If Spotify's background changes (different album art, dark mode toggle) a loose crop with too much background will break the match.

## The Script

```
#Requires AutoHotkey v2.0

^!d:: {
    if WinExist("ahk_exe Spotify.exe") {
        WinActivate
        WinWaitActive , , 2
    } else {
        MsgBox "Spotify is not running!"
        return
    }

    WinGetPos(&winX, &winY, &winW, &winH, "ahk_exe Spotify.exe")
    searchLeft := winX + winW - 500
    searchRight := winX + winW
    searchBottom := winY + winH
    searchBottomTop := searchBottom - 200

    ; Click the Connect button (bottom-right corner of Spotify)
    try {
        ImageSearch(&btnX, &btnY, searchLeft, searchBottomTop, searchRight, searchBottom, "*50 connect_button.png")
        Click btnX, btnY
    } catch {
        MsgBox "Could not find the Connect button."
        return
    }

    Sleep 350

    ; Click the speaker in the device list (right-side panel)
    try {
        ImageSearch(&devX, &devY, searchLeft, winY, searchRight, searchBottom, "*50 office.png")
        Click devX, devY
    } catch {
        MsgBox "Could not find 'Office' in the device list."
    }
}
```

The hotkey here is Ctrl+Alt+D — you can change this to whatever you like.

## Speeding Up ImageSearch

The first version of this script searched the entire screen which was noticeably slow. ImageSearch scans every pixel in the region you give it, so the smaller the region, the faster it runs.

The Connect icon lives in the bottom-right of the Spotify window, and the device panel pops up on the right side. So instead of scanning the full screen, the script gets Spotify's window position and only searches a 500px-wide strip on the right edge. For the Connect button specifically, it only looks in the bottom 200px of that strip.

This made it feel almost instant.

## Why 350ms Sleep?

The device panel appears visually almost immediately, but `ImageSearch` needs the pixels to be fully rendered. During a fade-in animation the colours won't match your screenshot exactly. 350ms was the sweet spot on my machine — anything lower and it couldn't find the image reliably. You might need to adjust this depending on your system.

The `*50` parameter in `ImageSearch` is the colour tolerance (0-255). If matching is unreliable, try increasing it to `*80` or `*100`.

## Mapping It to the Keychron

The V6 Max supports macros through the [Keychron Launcher](https://launcher.keychron.com). Connect the keyboard via USB, open the Launcher in Chrome or Edge, then:

1. Go to **MACROS** and pick an unused slot (check which ones are already in use first — if a key shows M0, M1 etc. in the keymap view, that slot is taken)
2. Enter the key combo: `{KC_LCTL,KC_LALT,KC_D}`
3. Save
4. Go to **KEYMAP**, click the key you want to assign it to, then select the macro from the **MACRO** section

I used the top-right key (the one marked with ×) since I never use it for anything else. One press and the office speaker is selected.
