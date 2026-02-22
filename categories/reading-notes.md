---
layout: page
title: Reading Notes
permalink: /categories/reading-notes/
---

{% assign posts = site.categories["reading-notes"] %}
{% for post in posts %}
- {{ post.date | date: "%Y-%m-%d" }} : [{{ post.title }}]({{ post.url }})
{% endfor %}