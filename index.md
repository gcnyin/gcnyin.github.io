---
layout: page
title: Post
---

{% for post in site.posts %}
<p><span style="font-family: monospace;">{{ post.date | date: "%Y/%m/%d" }}</span>&nbsp;&nbsp;<a href="{{ post.url | prepend: site.baseurl }}">{{ post.title | escape }}</a></p>
{% endfor %}
