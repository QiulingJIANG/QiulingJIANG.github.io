---
layout: page
title: Work Notes
permalink: /categories/work-notes/
---

{% assign posts = site.categories["work-notes"] %}
{% for post in posts %}
- {{ post.date | date: "%Y-%m-%d" }} : [{{ post.title }}]({{ post.url }})
{% endfor %}