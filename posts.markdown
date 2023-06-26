---
layout: page
title: Posts
permalink: /posts/
nav_order: 1
has_children: true
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>