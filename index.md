---
layout: default
title: Home
---

# Long Horizon Observatory - Engineering Blog

Welcome to the open-source engineering and R&D blog for [Long Horizon](https://longhorizon.eco). We document our workflows, models, and architectures here.

## Latest Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span> - {{ post.date | date: "%B %d, %Y" }}</span>
    </li>
  {% endfor %}
</ul>
