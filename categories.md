---
layout: page
title: Categories
permalink: /categories/
---

<div>
{% assign all_categories = site.posts | map: "category" | sort %}

{% assign applied_categories_raw = "" %}

{% for c in all_categories %}
    {% assign c_add_spliter = c | append:'|' %}
    {% unless applied_categories_raw contains c_add_spliter %}
        {% assign applied_categories_raw = applied_categories_raw | append:c_add_spliter %}
    {% endunless %}
{% endfor %}

{% assign applied_categories = applied_categories_raw | split:'|' %}

{% for category in applied_categories %}
	<a style="color: #6a9fb5; padding: 0.1rem 0.5rem; margin-right: .5rem; background: rgba(106,159,181,0.15);" href="#{{ category | slugify }}"> {{ category }} </a>
{% endfor %}

{% for category in applied_categories %}
    <p id="{{ category | slugify }}">{{ category }}</p>
    <ul style="list-style: none;">
    {% for post in site.posts %}
    {% if post.category == category %}
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