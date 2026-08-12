---
layout: page
title: Categories
permalink: /categories/
# Reuses the site's existing jekyll-toc setup to render a jump list of every
# category at the top of the page.
toc: true
---

{%- assign sorted_categories = site.categories | sort -%}
{%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}

{% for pair in sorted_categories %}
{%- assign category = pair[0] -%}
{%- assign posts = pair[1] -%}

## {{ category }}

<ul class="post-list">
  {%- for post in posts -%}
  <li>
    <span class="post-meta">{{ post.date | date: date_format }}</span>
    {%- comment -%}
      `no_toc` keeps post titles out of the jump list at the top of the page,
      which should only list the categories themselves.
    {%- endcomment -%}
    <h3 class="no_toc">
      <a class="post-link" href="{{ post.url | relative_url }}">
        {{ post.title | escape }}
      </a>
    </h3>
    {%- if post.description -%}
      <p class="post-subtitle">{{ post.description | escape }}</p>
    {%- endif -%}
  </li>
  {%- endfor -%}
</ul>
{% endfor %}
