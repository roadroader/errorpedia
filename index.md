---
layout: default
title: "Home"
permalink: /
---

# Errorpedia

短く・正確にエラーを直すためのメモリーベース辞典。

## 日本語（最新10件）
{%- assign ja_sorted = site.ja | sort: "date" | reverse -%}
{%- for p in ja_sorted limit:10 -%}
- <a href="{{ p.url | relative_url }}">{{ p.title | default: p.basename }}</a>
  <span class="meta"> — {{ (p.last_modified_at | default: p.date) | date: "%Y-%m-%d" }}</span>
  {%- if p.tags and p.tags != empty -%}
    <span class="meta"> • {{ p.tags | join: ", " }}</span>
  {%- endif -%}
{%- endfor -%}

## English (Latest 10)
{%- assign en_sorted = site.en | sort: "date" | reverse -%}
{%- for p in en_sorted limit:10 -%}
- <a href="{{ p.url | relative_url }}">{{ p.title | default: p.basename }}</a>
  <span class="meta"> — {{ (p.last_modified_at | default: p.date) | date: "%Y-%m-%d" }}</span>
  {%- if p.tags and p.tags != empty -%}
    <span class="meta"> • {{ p.tags | join: ", " }}</span>
  {%- endif -%}
{%- endfor -%}
