---
title: "Hello World"
---

# Hello #

- Sup yall

{{ page.title }}

# Posts #

<ul id="posts">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
      {{ post.author }}
    </li>
  {% endfor %}
</ul>

{{ site }}
