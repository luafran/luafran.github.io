---
layout: splash
permalink: /home
header:
  overlay_color: "#5e616c"
  overlay_image: /assets/images/rio-2.png
  caption:
excerpt: 'Software developer interested in DevOps, IOT and Context Aware Computing.'
---

<h3 class="archive__subtitle">{{ site.data.ui-text[site.locale].recent_posts | default: "Recent Posts" }}</h3>

{% for post in paginator.posts %}
  {% include archive-single.html %}
{% endfor %}

{% include paginator.html %}
