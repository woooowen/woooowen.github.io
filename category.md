---
layout: page
title: 分类
---

<div class="posts">  
  <ul class="related-posts">
    {% for post in site.posts %}
      <li>
        <h3>
          <a href="{{ site.baseurl }}{{ post.url }}">          
            {{ post.title }}
            <small>{{ post.date | date_to_string }}</small>
          </a>
		  <p>{{ post.categories }}</p>
        </h3>
      </li>
    {% endfor %}
  </ul>
</div>


