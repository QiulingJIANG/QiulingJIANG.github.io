---
layout: page
title: Reading Notes
permalink: /categories/reading-notes/
---

{% assign posts = site.categories["reading-notes"] %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}