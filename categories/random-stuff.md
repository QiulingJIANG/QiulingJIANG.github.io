---
layout: page
title: Random Stuff
permalink: /categories/random-stuff/
---

{% assign posts = site.categories["random-stuff"] %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}