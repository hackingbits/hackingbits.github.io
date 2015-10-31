---
layout: page
title: Author Index
excerpt: "An archive of posts sorted by author."
search_omit: true
---

{% capture site_authors %}{% for author in site.data.authors %}{{ author | first }}{% unless forloop.last %},{% endunless %}{% endfor %}{% endcapture %}
{% assign authors_list = site_authors | split:',' | sort %}

<ul class="tag-box inline">
{% for item in (0..site.data.authors.size) %}{% unless forloop.last %}
    {% capture author %}{{ authors_list[item] | strip_newlines }}{% endcapture %}
    {% assign counter=0 %}
    {% for p in site.posts do %}
      {% if p.author == author %}
        {% assign counter=counter | plus:1 %}
      {% endif %}
    {% endfor %}
    <li><a href="#{{ author }}" class="tag">{{ author }} <span>{{ counter }}</span></a></li>
  {% endunless %}{% endfor %}
</ul>

{% for item in (0..site.data.authors.size) %}{% unless forloop.last %}
  {% capture author %}{{ authors_list[item] | strip_newlines }}{% endcapture %}
  <h2 id="{{ author }}"><strong>{{ author }}</strong></h2>
  <ul class="post-list">
    {% for p in site.posts do %}
      {% if p.author == author %}
        <li><a href="{{ site.url }}{{ post.url }}">{{ p.title }}<span class="entry-date"><time datetime="{{ p.date | date_to_xmlschema }}">{{ p.date | date: "%B %d, %Y" }}</time></span></a></li>
      {% endif %}
    {% endfor %}
  </ul>
{% endunless %}{% endfor %}
