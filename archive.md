---
layout: page
title: Archives
permalink: /archive/
---

<section class="archive-post-list">

   {% for post in site.posts %}
       {% assign currentDate = post.date | date: "%Y" %}
       {% if currentDate != myDate %}
           {% unless forloop.first %}</ul>{% endunless %}
           <h1>{{ currentDate }}</h1>
           <ul class="codinfox-category-list">
           {% assign myDate = currentDate %}
       {% endif %}
       <li><a class="post-title" href="{{ post.url }}"><span>{{ post.date | date: "%B %-d, %Y" }}</span> - {{ post.title }}
            {% for tag in post.tags %}
                <a class="post-tag codinfox-tag-mark" href="/tag/#{{ tag | slugify }}">{{ tag }}</a>
            {% endfor %}
       </a></li>
       {% if forloop.last %}</ul>{% endif %}
   {% endfor %}

</section>
