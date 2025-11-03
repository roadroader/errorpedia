---
layout: default
title: "記事一覧（日本語）"
permalink: /ja/
---
<ul>
{% assign docs = site.ja | sort: 'title' %}
{% for p in docs %}
  <li><a href="{{ p.url | relative_url }}">{{ p.title }}</a></li>
{% endfor %}
</ul>
