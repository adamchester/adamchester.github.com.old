---
layout: page
title: Rarely updated
tagline: A programming blog by Adam Chester, that may be updated, rarely. 
---
{% include JB/setup %}

Welcome to a rarely updated blog.

This blog is still under construction. Please pretend I wrote something useful and insightful here.

<section id="article">
  <h1>Recent posts</h1>
  <ul>
{% for post in site.posts offset: 0 limit: 10  %}
    <li>&raquo; <a href="{{ post.url }}">{{ post.title }}</a></li>
{% endfor %}
  </ul>
</section>