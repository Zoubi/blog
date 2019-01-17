---
layout: page
title: Publications
permalink: /publications/
---

{% for publication in site.publications %}
<div class="publication">
    <a href="{{ publication.link }}"><img class="preview" src="{{ site.baseurl }}/assets/img/{{ publication.image }}" alt="{{ publication.book }}" /></a>
    <h3>{{ publication.title }}</h3>
    <p>{{ publication.authors }}</p>
</div>
{% endfor %}