---
title: Projects
layout: default

---

{% for post in site.posts %}
  <div class="post" onclick="window.location='{{ post.url }}';">
  <h3><a href="{{ post.url }}" class="post_title_a">{{ post.title }}</a></h3>
  <hr>
  {{ post.excerpt }}
  </div>
{% endfor %}
