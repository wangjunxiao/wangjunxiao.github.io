---
layout: page
title: Bluesy
tagline: have a wonderful day !
---
{% include JB/setup %}

Hi there!

I'm working on `SDN/NFV`.
    
## Blog posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

## Repository

dlut-vnetlab: [https://github.com/wangjunxiao/dlut-vnetlab][1]
sdn-nfv-dev: [https://github.com/wangjunxiao/sdn-nfv-dev][2]

## About me

email: wangjunxiao DOT dalian AT gmail DOT com



[1]: https://github.com/wangjunxiao/dlut-vnetlab    "dlut-vnetlab"
[2]: https://github.com/wangjunxiao/sdn-nfv-dev		"sdn-nfv-dev"
