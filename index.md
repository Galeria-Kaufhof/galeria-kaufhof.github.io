---
layout: page
title:
tagline:
---
{% include JB/setup %}

<ul class="gk-list gk-list-posts">
  {% for post in site.posts %}
    <li class="gk-post-preview">
      <div class="gk-post-preview__header">
        <span class="gk-post-preview__date">{{ post.date | date: "%Y-%m-%d" }}</span>
        <a class="gk-post-preview__title" href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
        <span class="gk-post-preview__author">by {{ site.authors[post.author].display_name }}</span>
      </div>
      {% if post.description != "" %}
      <p class="gk-post-preview__body">
        {{ post.description }}
        <a class="gk-post-preview__readmore" href="{{ BASE_PATH }}{{ post.url }}">Read more...</a>
      </p>
      {% endif %}
    </li>
  {% endfor %}
</ul>
