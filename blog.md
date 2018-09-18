---
layout: default
title: Blog posts
permalink: /blog/
---

{% for post in site.posts %}

{% unless post.hidden %}
<article class='post'>
  <h1 class='post-title'>
    <a href="{{ site.path }}{{ post.url }}">
      {{ post.title }}
    </a>
  </h1>
  <div class="post-date">{{ post.date | date: "%b %-d, %Y" }}</div>
  {{ post.content }}
</article>
{% endunless %}

{% endfor %}