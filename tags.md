---
layout: page
title: Tags
permalink: /tags/
---

<section class="links">
    {% for category in site.categories %}
    <a class="link tag-anchor" href="{{ category[0] | prepend: site.baseurl }}">
        <i class="fa fa-tag muted"></i>
        {{ category[0] }} ({{ category[1].size }})
    </a>
    {% endfor %}
</section>