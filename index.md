---
layout: default
title: Home
permalink: /
sitemap: true
---

<style>
  .home-wrap { max-width: 980px; margin: 0 auto; }
  .intro { margin: .25rem 0 1.25rem; color: #444; }
  .col-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 1.25rem; }
  .section h2 { margin: .25rem 0 .5rem; }
  .posts { list-style: none; padding: 0; margin: 0; }
  .posts li { padding: .6rem 0; border-top: 1px solid #eee; }
  .posts li:first-child { border-top: 0; }
  .posts a.title { font-weight: 600; text-decoration: none; }
  .meta { color: #667; font-size: .9rem; display: flex; gap: .5rem; flex-wrap: wrap; }
  .tags { display: inline-flex; gap: .35rem; flex-wrap: wrap; }
  .tag { background: #f1f5f9; border: 1px solid #e5e7eb; padding: .05rem .45rem; border-radius: .4rem; font-size: .8rem; text-decoration: none; }
  .more { margin-top: .75rem; }
  @media (max-width: 860px){ .col-2 { grid-template-columns: 1fr; } }
</style>

<div class="home-wrap">
  <div class="intro">
    <p><strong>Errorpedia</strong> は、Windowsや周辺技術の「原因→対処」を日英2言語でまとめる技術ノートです。下の最新記事からどうぞ。タグ別や全文検索は上部ナビの <em>Tags</em> / <em>Search</em> へ。</p>
  </div>

  {%- comment -%}
    JA/EN の記事を robust に収集：
    - site.pages から _ja / _en 配下を拾う
    - site.collections.*.docs に含まれている場合も concat して取り込む
  {%- endcomment -%}
  {%- assign ja_pages = site.pages | where_exp: "p", "p.path contains '/_ja/'" -%}
  {%- assign en_pages = site.pages | where_exp: "p", "p.path contains '/_en/'" -%}
  {%- for coll in site.collections -%}
    {%- assign _ja_docs = coll.docs | where_exp: "d", "d.path contains '/_ja/'" -%}
    {%- assign _en_docs = coll.docs | where_exp: "d", "d.path contains '/_en/'" -%}
    {%- assign ja_pages = ja_pages | concat: _ja_docs -%}
    {%- assign en_pages = en_pages | concat: _en_docs -%}
  {%- endfor -%}

  {%- comment -%}
    なるべく更新日( updatedAt )で新しい順に。無ければ date を利用。
    Liquid の sort は単一キーなので、まず updatedAt で降順表示を狙う。
  {%- endcomment -%}
  {%- assign ja_sorted = ja_pages | sort: "updatedAt" | reverse -%}
  {%- assign en_sorted = en_pages | sort: "updatedAt" | reverse -%}

  <div class="col-2">
    <section class="section">
      <h2>最新（日本語）</h2>
      <ul class="posts">
        {%- for p in ja_sorted limit:10 -%}
          {%- assign _title = p.title | default: p.errorText | default: p.url -%}
          {%- assign _u = p.updatedAt | default: p.last_modified_at | default: p.date -%}
          <li>
            <a class="title" href="{{ p.url | relative_url }}">{{ _title }}</a>
            <div class="meta">
              {%- if _u -%}
                <span>Updated: <time datetime="{{ _u | date_to_xmlschema }}">{{ _u | date: "%Y-%m-%d" }}</time></span>
              {%- endif -%}
              {%- if p.tags and p.tags.size > 0 -%}
                <span class="tags">
                  {%- for t in p.tags -%}
                    <a class="tag" href="{{ '/search/?lang=ja&tags=' | append: t | relative_url }}">{{ t }}</a>
                  {%- endfor -%}
                </span>
              {%- endif -%}
            </div>
          </li>
        {%- endfor -%}
      </ul>
      <div class="more"><a href="{{ '/search/?lang=ja' | relative_url }}">もっと見る →</a></div>
    </section>

    <section class="section">
      <h2>Latest (English)</h2>
      <ul class="posts">
        {%- for p in en_sorted limit:10 -%}
          {%- assign _title = p.title | default: p.errorText | default: p.url -%}
          {%- assign _u = p.updatedAt | default: p.last_modified_at | default: p.date -%}
          <li>
            <a class="title" href="{{ p.url | relative_url }}">{{ _title }}</a>
            <div class="meta">
              {%- if _u -%}
                <span>Updated: <time datetime="{{ _u | date_to_xmlschema }}">{{ _u | date: "%Y-%m-%d" }}</time></span>
              {%- endif -%}
              {%- if p.tags and p.tags.size > 0 -%}
                <span class="tags">
                  {%- for t in p.tags -%}
                    <a class="tag" href="{{ '/search/?lang=en&tags=' | append: t | relative_url }}">{{ t }}</a>
                  {%- endfor -%}
                </span>
              {%- endif -%}
            </div>
          </li>
        {%- endfor -%}
      </ul>
      <div class="more"><a href="{{ '/search/?lang=en' | relative_url }}">See more →</a></div>
    </section>
  </div>

  <div class="more" style="margin-top:1rem;">
    <a href="{{ '/tags/' | relative_url }}">→ Browse by Tags</a>
  </div>
</div>
