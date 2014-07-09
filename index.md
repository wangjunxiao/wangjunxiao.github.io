---
layout: page
title: Bluesy
tagline: Hi dude！
---
{% include JB/setup %}

Hi there!

I'm working on `SDN`.
    
## Blog posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

## About me

email: wangjunxiao DOT dalian AT gmail DOT com



