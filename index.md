---
# You don't need to edit this file, it's empty on purpose.
# Edit theme's home layout instead if you wanna make some changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: page
title:
tagline:
---

<ul class="gk-list gk-list-posts">
  {% for post in site.posts %}
    <li class="gk-post-preview">
      <div class="gk-post-preview__header">
        <a class="gk-post-preview__title" href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
        <div class="gk-post-preview__meta">
          <span class="gk-post-preview__author-prefix">by</span>
          <span class="gk-post-preview__author">{{ site.authors[post.author].display_name }}</span>
          <span class="gk-post-preview__date">@ {{ post.date | date: "%Y-%m-%d" }}</span>
        </div>
      </div>
      {% if post.description != "" %}
      <p class="gk-post-preview__body">
        {{ post.description }}
        <a class="gk-post-preview__readmore" href="{{ BASE_PATH }}{{ post.url }}">...</a>
      </p>
      {% endif %}
    </li>
  {% endfor %}
</ul>