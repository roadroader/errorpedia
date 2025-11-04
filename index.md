---
layout: default
title: Home
---

# Errorpedia
短く・正確にエラーを直すためのメモリーベース辞典。

_Last updated: {{ site.time | date: "%Y-%m-%d" }}_

<style>
/* --- Topだけの最小CSS（気に入ったら後で layout に移動OK） --- */
.post-list{list-style:none;margin:0;padding:0}
.post-list li{padding:.5rem 0;border-bottom:1px solid #eee;display:flex;flex-wrap:wrap;gap:.5rem 1rem;align-items:baseline}
.post-link{font-weight:600}
.meta{font-size:.875rem;opacity:.75}
.tags{display:flex;gap:.25rem}
.tag{font-size:.75rem;border:1px solid #ddd;border-radius:9999px;padding:.1rem .5rem}
.section-title{margin-top:1.5rem}
</style>

## 日本語（最新10件）
<ul class="post-list" aria-label="日本語の最新記事">
  {% assign ja = site.ja | sort: "updatedAt" | reverse %}
  {% for p in ja limit:10 %}
  <li>
    <a class="post-link" href="{{ p.url | relative_url }}">{{ p.title }}</a>
    <span class="meta">
      <time datetime="{{ p.updatedAt | date_to_xmlschema }}">{{ p.updatedAt | date: "%Y-%m-%d" }}</time>
    </span>
    <span class="tags">
      {% for t in p.tags %}
        <span class="tag">{{ t }}</span>
      {% endfor %}
    </span>
  </li>
  {% endfor %}
</ul>

## English (Latest 10)
<ul class="post-list" aria-label="English latest posts">
  {% assign en = site.en | sort: "updatedAt" | reverse %}
  {% for p in en limit:10 %}
  <li>
    <a class="post-link" href="{{ p.url | relative_url }}">{{ p.title }}</a>
    <span class="meta">
      <time datetime="{{ p.updatedAt | date_to_xmlschema }}">{{ p.updatedAt | date: "%Y-%m-%d" }}</time>
    </span>
    <span class="tags">
      {% for t in p.tags %}
        <span class="tag">{{ t }}</span>
      {% endfor %}
    </span>
  </li>
  {% endfor %}
</ul>
