---
layout: default
permalink: /debug/
---
<h2>Debug</h2>
<p>ja: {{ site.ja | size }} items / en: {{ site.en | size }} items</p>

<h3>JA</h3>
<ul>
{% for p in site.ja %}
  <li>{{ p.path }} → <code>{{ p.url }}</code></li>
{% endfor %}
</ul>

<h3>EN</h3>
<ul>
{% for p in site.en %}
  <li>{{ p.path }} → <code>{{ p.url }}</code></li>
{% endfor %}
</ul>
