---
---

# Hello #

- Sup yall

# Posts I think #

<ul id="posts">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.author }}
    </li>
  {% endfor %}
</ul>

{{ site }}
