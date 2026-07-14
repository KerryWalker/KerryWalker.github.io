---
layout: page
title: Home Automation & Energy
sidebar_link: true
excerpt: A software engineer's guides to home automation, cutting your energy bill with Octopus and Predbat, EV charging, and Home Assistant dashboards.
---

I'm a software engineer, and this is where I write up how I actually run my own home: cutting the energy bill with Octopus and Predbat, charging the car for as little as possible, and building the Home Assistant dashboards that tie it together. Real setups, real config, and the numbers behind them.

## Cutting your energy bill

Getting a home battery and solar to charge at the cheapest times on Octopus tariffs, with Predbat doing the optimising.

{% assign energy_posts = site.posts | where: "pillar", "energy" | sort: "date" %}
<ul>
{% for post in energy_posts %}  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}</ul>

## EV charging

Charging a Tesla on the cheap slots, automating the charger, and getting the car's data into Home Assistant.

{% assign ev_posts = site.posts | where: "pillar", "ev" | sort: "date" %}
<ul>
{% for post in ev_posts %}  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}</ul>

## Home Assistant dashboards

Building a family dashboard: calendars, and a custom energy gauge done properly.

{% assign dashboard_posts = site.posts | where: "pillar", "dashboards" | sort: "date" %}
<ul>
{% for post in dashboard_posts %}  <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a></li>
{% endfor %}</ul>

---

New to Octopus? If you switch using [my referral link](https://share.octopus.energy/lush-cliff-190) you get £50 of credit, and so do I.

Thinking of getting a Tesla? Order with [my referral link](https://ts.la/kerry/303868) for 650 free Supercharging miles.

<small>Some links on this site are affiliate or referral links. I may earn a small commission or credit at no extra cost to you.</small>
