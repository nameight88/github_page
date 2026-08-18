---
layout: archive
author_profile: true
sidebar:
  nav: "study-categories"
---

<!-- archive 레이아웃은 page.title이 없어도 빈 h1을 출력하므로 숨긴다 -->
<style>
  #page-title { display: none; }
</style>

## 📝 최근 포스트

{% for post in site.posts limit: 5 %}
  {% include archive-single.html %}
{% endfor %}

## 📚 카테고리 둘러보기

{% assign known_keys = site.data.category_names | map: "key" %}
<ul class="category-list">
{% for c in site.data.category_names %}
  {% assign posts_in_cat = site.categories[c.key] %}
  {% if posts_in_cat and posts_in_cat.size > 0 %}
  <li><a href="{{ '/categories/' | relative_url }}#{{ c.key | slugify }}">{{ c.name }}</a> ({{ posts_in_cat.size }})</li>
  {% endif %}
{% endfor %}
{% for cat in site.categories %}
  {% unless known_keys contains cat[0] %}
  <li><a href="{{ '/categories/' | relative_url }}#{{ cat[0] | slugify }}">{{ cat[0] }}</a> ({{ cat[1].size }})</li>
  {% endunless %}
{% endfor %}
</ul>
