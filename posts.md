---
layout: default
title: Posts
permalink: /posts
---

<div class="posts-container">
  <div class="posts-listing">
    <h1>Publicaciones</h1>
    
    {% for post in site.posts %}
      <div class="post-item">
        <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
        <div class="post-meta">
          <small>{{ post.date | date: "%b %d, %Y" }}</small>
          <div class="post-categories">
            {% for category in post.categories %}
              <span class="category-tag">{{ category }}</span>
            {% endfor %}
          </div>
        </div>
      </div>
    {% endfor %}
  </div>

  <aside class="categories-sidebar">
    <h3>Categorías</h3>
    <ul>
      {% assign categories = site.categories | sort %}
      {% for category in categories %}
        <li>
          <a href="#{{ category[0] | slugify }}">{{ category[0] }}</a>
          <span>({{ category[1].size }})</span>
        </li>
      {% endfor %}
    </ul>
  </aside>
</div>