---
layout: default
title: Home
---
<section class="post-list">
  <h2>Posts</h2>
  <ul>
    {% for post in site.posts %}
    <li>
      <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span class="post-date">{{ post.date | date: "%b %-d, %Y" }}</span>
      {% if post.summary %}<p class="post-summary">{{ post.summary }}</p>{% endif %}
    </li>
    {% endfor %}
  </ul>
</section>
