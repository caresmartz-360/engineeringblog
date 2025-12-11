---
layout: default
---

# Latest Posts

{% for post in site.posts %}
  <article>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <p class="post-meta">{{ post.date | date: "%B %-d, %Y" }}</p>
    {% if post.excerpt %}
      {{ post.excerpt }}
    {% endif %}
  </article>
{% endfor %}
