---
title: "Blog"
permalink: /blog/
layout: single
author_profile: true
---

{% for post in site.posts %}
- <span style="color: #999; font-size: 0.85em;">{{ post.date | date: "%Y-%m-%d" }}</span> [{{ post.title }}]({{ post.url }})
{% endfor %}
