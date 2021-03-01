---
layout: content
---

{% for post in site.categories.blog limit:10 %}
   <div class="post-preview">
   <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
   <div class="post-date">{{ post.date | date: "%B %d, %Y" }}</div>
   {{ post.content | split:'<!--break-->' | first }}
   {% if post.content contains '<!--break-->' %}
      <span class="read-more"><a href="{{ post.url }}">read more ≫</a></span>
   {% endif %}
   </div>
   <hr>
{% endfor %}
