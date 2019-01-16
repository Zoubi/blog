---
layout: page
title: Publications
permalink: /publications/
---

{% for publication in site.publications %}
  <h2>{{ publication.title }}</h2>
  <a href="{{ publication.link }}"><img src="{{ site.baseurl }}/assets/img/{{ publication.image }}" alt="{{ publication.book }}" /></a>
  <p>{{ publication.authors }}</p>
{% endfor %}