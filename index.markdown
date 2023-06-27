---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
title: Home
nav_order: 1
---
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <p>{{ post.date | date: '%B %d, %Y' }}</p>
      <p>{{ post.excerpt | strip_html | truncatewords:75}}</p>
    </li>
  {% endfor %}
</ul>