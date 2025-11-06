---
layout: default
title: 最新記事（日本語）
permalink: /ja/
pagination:
  enabled: true
  collection: ja           # ← _config.yml の collections 名
  per_page: 10
  sort_field: "updatedAt"  # front matter の updatedAt を使う
  sort_reverse: true
---

<ol style="line-height:1.6;margin-left:1.25rem;">
  {% raw %}{% for doc in paginator.docs %}{% endraw %}
    <li style="margin:.25rem 0;">
      <a href="{% raw %}{{ doc.url | relative_url }}{% endraw %}">{% raw %}{{ doc.title }}{% endraw %}</a>
      <small style="color:#64748b;">（更新: {% raw %}{{ doc.updatedAt | default: doc.date | date: "%Y-%m-%d" }}{% endraw %}）</small>
      {% raw %}{% if doc.tags %}<span style="color:#475569;"> — {{ doc.tags | join: ", " }}</span>{% endif %}{% endraw %}
    </li>
  {% raw %}{% endfor %}{% endraw %}
</ol>

<nav class="pager" style="display:flex;gap:.75rem;align-items:center;">
  {% raw %}{% if paginator.previous_page_path %}{% endraw %}
    <a href="{% raw %}{{ paginator.previous_page_path | relative_url }}{% endraw %}">« 前のページ</a>
  {% raw %}{% endif %}{% endraw %}
  <span>{% raw %}{{ paginator.page }} / {{ paginator.total_pages }}{% endraw %}</span>
  {% raw %}{% if paginator.next_page_path %}{% endraw %}
    <a href="{% raw %}{{ paginator.next_page_path | relative_url }}{% endraw %}">次のページ »</a>
  {% raw %}{% endif %}{% endraw %}
</nav>
