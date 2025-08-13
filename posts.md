---
layout: default
title: Posts
permalink: /posts
---

# Publicaciones

<div class="posts-listing">
  {% for post in site.posts %}
    <div class="post-item">
      <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
      <small>{{ post.date | date: "%b %d, %Y" }}</small>
    </div>
  {% endfor %}
</div>