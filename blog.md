---
layout: default
title: Deep Dives
permalink: /blog/
---

<div class="hero">
  <h1>Deep Dives</h1>
  <p class="subtitle">
    Technical articles on distributed systems, schema governance, and ML pipelines. Each article is rooted in real engineering experience — referencing actual code, Apicurio Registry contributions, and production architecture decisions.
  </p>
</div>

<div class="card-grid">
{% assign sorted_posts = site.posts | sort: 'date' | reverse %}
{% for post in sorted_posts %}
  <div class="card">
    <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
    <p>{{ post.description }}</p>
    <div class="card-meta">
      <time>{{ post.date | date: "%B %d, %Y" }}</time>
    </div>
    <div class="card-tags">
      {% for tag in post.tags %}
      <span class="tag">{{ tag }}</span>
      {% endfor %}
    </div>
  </div>
{% endfor %}
</div>
