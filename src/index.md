---
layout: default
front: yes
title: Kyle Keesling - An Indianapolis-based Pixel Pusher
---

<ul class="list-disc">
  {% for post in site.posts %}
    <li>
      <a class="underline" href="{{ post.url }}">{{ post.date | date: "%Y-%m-%d" }} - {{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
