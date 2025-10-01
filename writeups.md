---
layout: default
title: Write-ups
permalink: /writeups/
---

<!-- # Write-ups -->

<!-- ## All Write-ups -->

{% for writeup in site.writeups %}
## [{{ writeup.title }}]({{ writeup.url }})
<!-- **Date:** {{ writeup.date | date: "%B %d, %Y" }}   -->
**Category:** {{ writeup.categories | join: ", " }}

---



{% endfor %}

{% if site.writeups.size == 0 %}
<p>No write-ups yet. Check back soon!</p>
{% endif %}