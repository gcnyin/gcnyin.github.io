---
layout: page
title: Tags
permalink: /tags/
---

<div>
{% assign rawtags = "" %}
{% for post in site.posts %}
	{% assign ttags = post.tags | join:'|' | append:'|' %}
	{% assign rawtags = rawtags | append:ttags %}
{% endfor %}
{% assign rawtags = rawtags | split:'|' | sort %}

{% assign tags = "" %}
{% for tag in rawtags %}
	{% if tag != "" %}
		{% if tags == "" %}
			{% assign tags = tag | split:'|' %}
		{% endif %}
		{% unless tags contains tag %}
			{% assign tags = tags | join:'|' | append:'|' | append:tag | split:'|' %}
		{% endunless %}
	{% endif %}
{% endfor %}

{% for tag in tags %}
	<a style="color: #6a9fb5; padding: 0.1rem 0.5rem; margin-right: .5rem; background: rgba(106,159,181,0.15);" href="#{{ tag | slugify }}"> {{ tag }} </a>
{% endfor %}

{% for tag in tags %}
	<p id="{{ tag | slugify }}">{{ tag }}</p>
	<ul style="list-style: none;">
	 {% for post in site.posts %}
		 {% if post.tags contains tag %}
		 <li>
		 <p>
		 <a href="{{ post.url }}">
		 {{ post.title | escape }}
		 </a>
		 </p>
		 </li>
		 {% endif %}
	 {% endfor %}
	</ul>
{% endfor %}
</div>