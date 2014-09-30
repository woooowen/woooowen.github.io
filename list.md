---
layout: page
title: 目录
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
        </h3>
      </li>
    {% endfor %}
  </ul>
</div>
