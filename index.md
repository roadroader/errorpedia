---
layout: default
title: Home
permalink: /
---

<div class="home" style="max-width:980px;margin:0 auto;padding:1rem;">
  <h1 style="margin:.25rem 0 1rem;">Errorpedia</h1>
  <p style="color:#475569;margin-bottom:1.25rem;">
    最新の記事を日本語と英語でそれぞれ10件ずつ表示します。
  </p>

  <!-- 最新（日本語） -->
  <section style="margin:1.5rem 0;">
    <h2 style="font-size:1.25rem;margin-bottom:.5rem;">最新記事（日本語）</h2>
    {%- assign ja_all = site.pages
        | where_exp: "p", "p.lang == 'ja' or p.path contains 'content/_ja/' or p.path contains 'content/ja/'"
        | sort: "updatedAt" | reverse -%}
    <ol style="line-height:1.6;margin-left:1.25rem;">
      {%- for p in ja_all limit:10 -%}
        <li style="margin:.25rem 0;">
          <a href="{{ p.url | relative_url }}">{{ p.title }}</a>
          {%- assign iso = p.updatedAt | default: "" -%}
          {%- if iso != "" -%}
            <small style="color:#64748b;"> — {{ iso | slice: 0, 10 }}</small>
          {%- elsif p.date -%}
            <small style="color:#64748b;"> — {{ p.date | date: "%Y-%m-%d" }}</small>
          {%- endif -%}
          {%- if p.tags -%}
            <span style="color:#94a3b8;"> · {{ p.tags | join: ", " }}</span>
          {%- endif -%}
        </li>
      {%- endfor -%}
      {%- if ja_all == empty -%}
        <li>まだ記事がありません。</li>
      {%- endif -%}
    </ol>
  </section>

  <!-- Latest (English) -->
  <section style="margin:1.5rem 0;">
    <h2 style="font-size:1.25rem;margin-bottom:.5rem;">Latest Articles (English)</h2>
    {%- assign en_all = site.pages
        | where_exp: "p", "p.lang == 'en' or p.path contains 'content/_en/' or p.path contains 'content/en/'"
        | sort: "updatedAt" | reverse -%}
    <ol style="line-height:1.6;margin-left:1.25rem;">
      {%- for p in en_all limit:10 -%}
        <li style="margin:.25rem 0;">
          <a href="{{ p.url | relative_url }}">{{ p.title }}</a>
          {%- assign iso = p.updatedAt | default: "" -%}
          {%- if iso != "" -%}
            <small style="color:#64748b;"> — {{ iso | slice: 0, 10 }}</small>
          {%- elsif p.date -%}
            <small style="color:#64748b;"> — {{ p.date | date: "%Y-%m-%d" }}</small>
          {%- endif -%}
          {%- if p.tags -%}
            <span style="color:#94a3b8;"> · {{ p.tags | join: ", " }}</span>
          {%- endif -%}
        </li>
      {%- endfor -%}
      {%- if en_all == empty -%}
        <li>No articles yet.</li>
      {%- endif -%}
    </ol>
  </section>
</div>
