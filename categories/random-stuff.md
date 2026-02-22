---
layout: page
title: Random Stuff
permalink: /categories/random-stuff/
---

{% assign posts = site.categories["random-stuff"] %}
{% for post in posts %}
- {{ post.date | date: "%Y-%m-%d" }} : [{{ post.title }}]({{ post.url }})
{% endfor %}