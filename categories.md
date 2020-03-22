---
layout: page-without-tag-category
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
    <a href="#{{ category | slugify }}"><span class="badge badge-success"><i class="fa fa-hashtag"></i>&nbsp;{{ category }}</span></a>
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