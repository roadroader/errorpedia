---
layout: default
title: "Errorpedia"
---

## 日本語
<ul>
{% assign docs = site.ja | sort: 'title' %}
{% for p in docs %}
<li><a href="{{ p.url | relative_url }}">{{ p.title }}</a>{% if p.updatedAt %} <small>（{{ p.updatedAt }}）</small>{% endif %}</li>
{% endfor %}
</ul>

## English
<ul>
{% assign docs2 = site.en | sort: 'title' %}
{% for p in docs2 %}
<li><a href="{{ p.url | relative_url }}">{{ p.title }}</a>{% if p.updatedAt %} <small>({{ p.updatedAt }})</small>{% endif %}</li>
{% endfor %}
</ul>
