---
layout: page
title: Referral Codes and Discount Links
short_title: Referrals
sidebar_link: true
excerpt: Referral codes and discount links for Octopus Energy, Tesla, Puddle Ducks swimming lessons, and Yoto players. Products and services I actually use, with a discount or bonus for you when you sign up.
---

Here are referral codes and links for products and services I actually use. Each one gets you a discount or bonus for signing up, and each page explains honestly why we use it.

{% assign referral_pages = site.pages | where: "referral", true | sort: "title" %}
<ul>
{% for p in referral_pages %}  <li><strong><a href="{{ p.url | relative_url }}">{{ p.short_title | default: p.title }}</a></strong>{% if p.referral_summary %} - {{ p.referral_summary }}{% endif %}</li>
{% endfor %}</ul>

<small>Every link on these pages is a referral or recommend-a-friend link. I receive a credit or reward when you use one, at no extra cost to you. None of these pages are affiliated with or endorsed by the companies mentioned.</small>
