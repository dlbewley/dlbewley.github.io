---
---

# Hello #

- Sup yall

# Posts I think #

<ul id="posts">
  {% for post in site.posts %}
    <li>
      <h2>{{ post.title }}</h2>
      {{ post.author }}
    </li>
  {% endfor %}
</ul>

{{ site.posts }}
