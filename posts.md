---
layout: default
title: Posts
permalink: /posts
---

# Publicaciones

<ul class="posts-list">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <small>{{ post.date | date: "%b %d, %Y" }}</small>
    </li>
  {% endfor %}
</ul>