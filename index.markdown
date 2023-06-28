---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
title: Home
nav_order: 1
---

[comment]: <> (<ul>)

[comment]: <> (  {% for post in site.posts %})

[comment]: <> (    <li>)

[comment]: <> (      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>)

[comment]: <> (      <p>{{ post.date | date: '%B %d, %Y' }}</p>)

[comment]: <> (      <p>{{ post.excerpt | strip_html | truncatewords:75}}</p>)

[comment]: <> (    </li>)

[comment]: <> (  {% endfor %})

[comment]: <> (</ul>)