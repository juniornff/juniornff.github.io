---
layout: default
title: Posts
permalink: /posts
---

<div class="posts-container">
  <div class="posts-listing">
    <h1>Publicaciones</h1>
    
    <div id="posts-list">
      {% for post in site.posts %}
        <div class="post-item" data-categories="{{ post.categories | join: ',' | downcase }}">
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
  </div>

  <aside class="categories-sidebar">
    <h3>Categorías</h3>
    <ul id="categories-list">
      <li>
        <a href="#" onclick="clearFilter(); return false;">
          <strong>Mostrar todas</strong>
        </a>
      </li>
      {% assign categories = site.categories | sort %}
      {% for category in categories %}
        {% assign category_name = category[0] %}
        <li>
          <a href="#" 
             data-category="{{ category_name | downcase }}" 
             onclick="filterPosts('{{ category_name | downcase }}'); return false;">
            {{ category_name }}
          </a>
          <span>({{ category[1].size }})</span>
        </li>
      {% endfor %}
    </ul>
  </aside>
</div>

<script>
  function filterPosts(category) {
    const posts = document.querySelectorAll('.post-item');
    posts.forEach(post => {
      const postCategories = post.dataset.categories.split(',');
      if (postCategories.includes(category)) {
        post.style.display = 'block';
      } else {
        post.style.display = 'none';
      }
    });
    
    // Actualizar URL sin recargar la página
    history.pushState({}, '', `?category=${category}`);
    
    // Resaltar categoría seleccionada
    document.querySelectorAll('#categories-list a').forEach(link => {
      link.classList.remove('active');
    });
    document.querySelector(`#categories-list a[data-category="${category}"]`).classList.add('active');
  }
  
  function clearFilter() {
    const posts = document.querySelectorAll('.post-item');
    posts.forEach(post => {
      post.style.display = 'block';
    });
    
    // Limpiar parámetro de URL
    history.pushState({}, '', window.location.pathname);
    
    // Quitar resaltado de categorías
    document.querySelectorAll('#categories-list a').forEach(link => {
      link.classList.remove('active');
    });
  }
  
  // Filtrar al cargar la página si hay parámetro en la URL
  document.addEventListener('DOMContentLoaded', () => {
    const urlParams = new URLSearchParams(window.location.search);
    const category = urlParams.get('category');
    if (category) {
      filterPosts(category);
    }
  });
</script>