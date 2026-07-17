---
layout: post
published: false
title: Streaming Plex Music to Sonos from Home Assistant — Without Remote Access
excerpt: I wanted to play music from my NAS to my Sonos speakers, controlled from Home Assistant, all locally. The documented method can't do it — here's why, and the setup that can.
tags:
  - home-assistant
  - sonos
  - plex
  - music-assistant
  - nas
---

I've got a Plex Media Server running on my Synology DS418j with a music library of around 20,000 files, a house full of Sonos speakers, and Home Assistant tying everything together. The goal was simple: play my own music from the NAS to any Sonos speaker, browse and control it from Home Assistant, and keep the whole thing local.

That last part mattered. I didn't want to enable Plex Remote Access just to play music inside my own house. Everything is on the same network — it should just work locally. It took most of an afternoon and some debug logging to find out that the documented method can't actually do that, and that a different tool does the job better anyway.

## The Setup

- **Library:** Plex Media Server on a Synology DS418j
- **Speakers:** Sonos, integrated with Home Assistant
- **Control:** Home Assistant on a Raspberry Pi

## The Obvious Approach

Home Assistant has a Plex integration and a Sonos integration. Add both, and you can play Plex music to a Sonos speaker with a service call:

```yaml
action: media_player.play_media
target:
  entity_id: media_player.dressing_room
data:
  media_content_type: music
  media_content_id: >
    plex://{ "library_name": "Music", "artist_name": "Adele",
             "album_name": "25" }
```

Both integrations auto-discover on the local network — the Plex integration finds the server on its local IP, the Sonos integration finds the speakers. On paper this is a fully local setup. I ran the service call and got:

```
Error calling SonosMediaPlayerEntity._play_media on
media_player.dressing_room: UPnP Error 800 received: from 192.168.0.103
```

## Chasing UPnP Error 800

Error 800 is Sonos's generic "I can't play what you gave me" — no detail, no reason. The advice you'll find everywhere is that it's Plex's HTTPS handling: Sonos struggles with the self-signed certificate on Plex's `plex.direct` addresses. The fixes that come up, roughly in order:

- **Secure connections.** Plex Web → Settings → Network → Secure connections. The old advice is to set this to Disabled, but that option has been removed — Preferred is now as low as it goes.
- **Allow the local subnet without auth.** Add your LAN (e.g. `192.168.0.0/24`) to Plex's "List of IP addresses and networks that are allowed without auth".
- **A custom access URL.** Add `http://<nas-ip>:32400` under Custom server access URLs so Plex advertises a plain-HTTP local address. I initially entered this without the `:32400` port — without the port it's a dead address and does nothing.
- **Restart Plex, then reload the integration.** Home Assistant caches the server connection, so any Plex-side change needs a Plex restart *and* an integration reload to take effect.

I did all of it. Still 800. The speaker itself was fine — it happily played Spotify from the Home Assistant dashboard — so the problem was specific to the Plex handoff.

## Turning On Debug Logging

When the standard fixes don't work, stop guessing and look at what's actually being sent. In Developer Tools → Actions:

```yaml
action: logger.set_level
data:
  soco: debug
  plexapi: debug
  homeassistant.components.sonos: debug
```

One thing that caught me out: the Settings → System → Logs page only shows warnings and errors, so the debug lines never appear there. You have to download the full log (the download icon on that page) or open `/config/home-assistant.log` directly.

When I did, the command being sent to the speaker told the whole story:

```
Sending AddURIToQueue [('EnqueuedURI',
'x-rincon-cpcontainer:1004206c...album?sid=212&flags=8300&sn=9'), ...
SA_RINCON54279_X_#Svc54279-0-Token ...
```

There's no NAS IP in there. No `http`, no `plex.direct`, nothing pointing at my server. `x-rincon-cpcontainer` with a service token (`Svc54279`) is Sonos-speak for "play this from a music service registered in the Sonos app". Home Assistant wasn't telling the speaker to fetch the file from my Plex server at all — it was telling it to use the **Plex-for-Sonos service**.

## Why It Can't Work Locally

All those certificate and network fixes were aimed at the wrong path. The integration doesn't stream Plex to Sonos directly — it drives the Plex music service registered inside the Sonos app, and that service has a hard requirement I'd missed. Straight from the Home Assistant Plex documentation, playing Plex music to Sonos needs:

- Remote access enabled for your Plex server
- Sonos speakers linked to your Plex account
- The Sonos integration configured

Remote Access. Required. The very thing I was trying to avoid. The debug log confirmed the mechanism — Home Assistant was reaching out to `plex.tv/api/v2/resources?includeHttps=1&includeRelay=1` to find the server's remote connection. The speaker doesn't reach the NAS on its local IP the way a laptop does; it goes through Plex's service, brokered via the cloud.

So the three things I wanted — Plex, no Remote Access, fully local — can't all be true on this path. That's not a configuration problem, it's how Plex-for-Sonos is built.

## Music Assistant Instead

There was a second problem with the native route anyway: no search. With nearly 20,000 tracks in the library, there's no free-text search anywhere in the Home Assistant media cards.

[Music Assistant](https://www.music-assistant.io/) solves both. It's a music library manager that sits between your sources and your players — it connects to Plex as a library provider and to Sonos as a player provider, gives you a proper interface with search and artwork, and streams the audio itself rather than going through the Plex-for-Sonos service.

It comes in two parts, and the split confused me at first:

1. **Music Assistant Server** — an add-on. Settings → Add-ons → Add-on Store → install and start it. Turn on "Start on boot" and "Watchdog", and toggle "Show in sidebar" — that last one is off by default, which is why it didn't appear for me at first. This needs Home Assistant OS or Supervised for add-on support.
2. **Music Assistant integration** — now an official component in Home Assistant core, which is why you *won't* find it in HACS any more (the old HACS custom integration is deprecated). Add it via Settings → Devices & Services → Add Integration → Music Assistant. It auto-discovers the running server.

Then, inside Music Assistant:

1. Add **Plex Media Server** as a music provider. The easiest route is "Authenticate Locally", which needs your Home Assistant machine's IP in Plex's "allowed without auth" list — the same `192.168.0.0/24` entry from the troubleshooting above already covers it. It auto-fills the local IP and port (32400) and pulls in the library.
2. Add **Home Assistant Media Players** as a player provider and tick your Sonos speakers.
3. Let it sync. On a Raspberry Pi, pulling in nearly 20,000 tracks takes a while — leave it.

## Proving It's Actually Local

Once Music Assistant was playing, I went back and tested whether it needed any of the things the native path demanded:

- **Turned Remote Access off in Plex.** Changed album, played fine.
- **Disabled the Home Assistant Plex integration entirely and restarted Home Assistant.** I expected the music to stop. It didn't — on restart I changed album and it played.

That confirms it. Music Assistant's Plex provider is a completely separate connection: it authenticates to Plex directly on the local IP and port, reads the library, and serves the stream to the Sonos itself. The Home Assistant Plex integration was never part of its path, and the Plex-for-Sonos service isn't involved at all. The Music Assistant server also runs independently of Home Assistant, which is why a Home Assistant restart doesn't interrupt playback.

You can undo the Plex network changes made while chasing the 800 — the custom access URL in particular — but leave your machine's IP (or the subnet) in the "allowed without auth" list, because Music Assistant uses that for local authentication.

## The Final Setup

- **Library:** Plex on the DS418j, local, Remote Access off
- **Bridge:** Music Assistant server on the Pi — Plex provider via local IP and port, Sonos as player provider
- **Control:** the Music Assistant panel for search and browsing; Home Assistant for automations
- **Not needed:** the Home Assistant Plex integration, the Plex-for-Sonos service, Remote Access

A note on the hardware: the DS418j is a modest ARM NAS with 512MB of RAM, but since it's only serving music files — no transcoding involved — it copes fine. The initial library sync on the Pi is the only slow part of the whole setup.

## Summary

The native Plex → Sonos integration does work, and if you're happy to run Plex Remote Access it's a legitimate setup. But it can't run locally-only, it leans on Plex's cloud to broker the connection, and it gives you no way to search a large library. Music Assistant solved all three: real search, a proper interface, fully local playback, and it survives a Home Assistant restart.

If you've landed here Googling "Sonos Plex UPnP 800", the short version: it's almost certainly not your certificates or your network. It's the Plex-for-Sonos service needing Remote Access. If that requirement bothers you, don't fight it — install Music Assistant and stream your library the way you actually wanted to.
