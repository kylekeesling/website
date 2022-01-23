---
layout: default
front: yes
title: Kyle Keesling - An Indianapolis-based Pixel Pusher
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.date | date: "%Y-%m-%d" }} - {{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
