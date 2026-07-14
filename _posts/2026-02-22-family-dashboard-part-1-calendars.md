---
layout: post
pillar: dashboards
published: true
title: "Family Dashboard Part 1: iCloud Calendars in Home Assistant"
excerpt: "Pulling four iCloud calendars into Home Assistant using CalDAV and displaying them with Atomic Calendar Revive."
tags:
  - home-assistant
  - apple
  - caldav
---

I'm building a family dashboard in Home Assistant. The kind of thing you stick on a tablet in the kitchen and everyone can glance at to see what's happening. First thing on the list: calendars.

We have four Apple accounts and a shared Family calendar. CalDAV handles this nicely — it's the protocol iCloud uses natively, so there's no syncing hacks or third-party bridges involved.

## App-Specific Passwords

You can't use your regular iCloud password. Apple requires an app-specific password for third-party access. For each of your Apple accounts:

1. Go to [appleid.apple.com](https://appleid.apple.com)
2. **Sign-In and Security → App-Specific Passwords**
3. Generate one and label it "Home Assistant" or similar

## Adding CalDAV

In Home Assistant: **Settings → Devices & Services → Add Integration → CalDAV**

- **URL:** `https://caldav.icloud.com/.well-known/caldav`
- **Username:** Your iCloud email
- **Password:** The app-specific password

HA discovers all calendars on that account automatically. Each becomes a `calendar.*` entity. Repeat for each family member.

The shared Family calendar doesn't have its own Apple ID — it lives under whichever account is the Family Sharing organiser. It'll probably appear under multiple accounts since they're all subscribed. Just pick one instance for the dashboard.

## Atomic Calendar Revive

The built-in calendar card shows a monthly grid which isn't great for a quick "what's on today" glance. [Atomic Calendar Revive](https://github.com/totaldebug/atomic-calendar-revive) from HACS gives you an agenda-style list instead.

Install via **HACS → Frontend**, search for it, install, restart HA. You might need to clear your browser cache.

```yaml
type: custom:atomic-calendar-revive
name: Family Calendar
entities:
  - entity: calendar.family
    color: '#2ECC71'
  - entity: calendar.person1
    color: '#3498DB'
  - entity: calendar.person2
    color: '#E74C3C'
  - entity: calendar.person3
    color: '#F39C12'
  - entity: calendar.person4
    color: '#9B59B6'
maxDaysToShow: 14
showLocation: true
showMonth: true
dimFinishedEvents: true
finishedEventOpacity: 0.6
showCurrentEventLine: true
showProgressBar: true
sortByStartTime: true
```

Each family member gets a colour so you can tell at a glance whose event is whose. `dimFinishedEvents` fades out past events which keeps the focus on what's coming next.

## Next

Next up I'm building a custom concentric horseshoe gauge that shows the entire solar/battery system on a single card. [Part 2: Custom Energy Gauge →](/2026/02/25/family-dashboard-part-2-energy-gauge.html)

The full dashboard config is in the [GitHub repo](https://github.com/KerryWalker/Growatt-Solar-Assistant-Home-Assistant-Dash).
