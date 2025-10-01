---
layout: default
title: Blog
permalink: /blog/
---

<!-- Where I share what I'm learning about Web3, security, and development. -->

{% for post in site.posts %}
## [{{ post.title }}]({{ post.url }})
**Date:** {{ post.date | date: "%B %d, %Y" }}  
**Categories:** {{ post.categories | join: ", " }}

{{ post.excerpt | strip_html | truncatewords: 30 }}

[Read more →]({{ post.url }})

---
{% endfor %}

{% if site.posts.size == 0 %}
<p>No blog posts yet. Check back soon!</p>
{% endif %}