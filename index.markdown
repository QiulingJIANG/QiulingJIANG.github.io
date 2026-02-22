---
layout: page
---

## Welcome to my digital hamster den 🐹

Notes sorted by:
- [📚 Reading Notes](/categories/reading-notes/)
- [📝 Work Notes](/categories/work-notes/)
- [🎲 Random Stuff](/categories/random-stuff/)

## Recent Posts

<ul style="list-style: none; padding-left: 0;">
{% for post in site.posts limit:1 %}
<li style="margin-bottom: 0.5em;">
  <span style="color: #828282; font-size: 0.8em;">{{ post.date | date: "%b %-d, %Y" }}</span><br>
  <a href="{{ post.url }}" style="font-size: 1.0em; ">{{ post.title }}</a>
</li>
{% endfor %}
</ul>