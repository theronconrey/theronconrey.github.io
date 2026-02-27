---
layout: default
title: Posts
permalink: /posts/
---

<div class="home">
  <h1 class="page-heading">Posts</h1>

  <div class="search-wrap">
    <input type="text" id="search-input" placeholder="Search posts…" />
  </div>

  <ul class="post-list" id="post-list">
    {% for post in site.posts %}
      <li data-title="{{ post.title | downcase }}" data-content="{{ post.content | strip_html | downcase | truncate: 500 }}">
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
        <h3>
          <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
        </h3>
        {% if post.excerpt %}
          <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 160 }}</p>
        {% endif %}
      </li>
    {% endfor %}
  </ul>

  <p id="no-results" style="display:none;">No posts found.</p>
</div>

<script>
  const input = document.getElementById('search-input');
  const posts = document.querySelectorAll('#post-list li');
  const noResults = document.getElementById('no-results');

  input.addEventListener('input', function () {
    const query = this.value.toLowerCase().trim();
    let visible = 0;

    posts.forEach(post => {
      const title = post.dataset.title || '';
      const content = post.dataset.content || '';
      const match = !query || title.includes(query) || content.includes(query);
      post.style.display = match ? '' : 'none';
      if (match) visible++;
    });

    noResults.style.display = visible === 0 ? '' : 'none';
  });
</script>
