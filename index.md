---
layout: default
title: Home
---

<!-- Hero Section -->
<section class="hero">
  <div class="container">
    <div class="hero-content">
      <h1>AWS & Cloud Services</h1>
      <p>Expert in-depth guides and best practices for cloud architecture and AWS services</p>
      <div class="hero-buttons">
        <a href="#blog" class="cta-button">Start Reading</a>
        <a href="/blog" class="cta-button secondary">All Articles</a>
      </div>
    </div>
  </div>
</section>

<!-- Latest Blog Posts -->
<section class="blog-section" id="blog">
  <div class="container">
    <h2>Latest Blog Posts</h2>
    <div class="blog-grid">
      {% for post in site.posts limit:6 %}
      <article class="blog-card">
        <div class="blog-date">{{ post.date | date: "%B %d, %Y" }}</div>
        <h3><a href="{{ post.url }}">{{ post.title }}</a></h3>
        <p>{{ post.excerpt | strip_html | truncatewords: 20 }}</p>
        <a href="{{ post.url }}" class="read-more">Read More →</a>
      </article>
      {% endfor %}
    </div>
    {% if site.posts.size > 6 %}
    <div class="view-all">
      <a href="/blog" class="cta-button">View All Posts</a>
    </div>
    {% endif %}
  </div>
</section>
