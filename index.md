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

## Repository

SDN: [https://github.com/wangjunxiao/sdn][1]

## About me

email: wangjunxiao DOT dalian AT gmail DOT com



[1]: https://github.com/wangjunxiao/sdn    "SDN"

