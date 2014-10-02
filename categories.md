---
title: 分类
layout: page
---
<ul class="entry-meta inline-list">
{% for cat in site.categories %}
<li><a href="#{{ cat[0] }}" title="{{ cat[0] }}" rel="{{ cat[1].size }}" class="tag">{{ cat[0] }}<span>{{ cat[1].size }}</span></a></li>
{% endfor %}
</ul>

{% for cat in site.categories %}
<article id="{{ cat[0] }}">
<ul class="related-posts">
  <h2>{{ cat[0] }}</h2>
{% for post in cat[1] %}
  <li class="listing-item">
  <h3>
  	<a href="{{ post.url }}" title="{{ post.title }}">
  		{{ post.title }}
  		<small>{{ post.date | date: "%Y-%m-%d"  }}</small>
  	</a>
  </h3>
  </li>
{% endfor %}
</ul>
</article>
{% endfor %}


