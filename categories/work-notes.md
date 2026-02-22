---
layout: page
title: Work Notes
permalink: /categories/work-notes/
---

{% assign posts = site.categories["work-notes"] %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}