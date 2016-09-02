---
layout: page
title: 区块链实用手册
tagline: 导航
---
{% include JB/setup %}

* [NXT白皮书中文版]({{ site.url }}/NXT白皮书中文译本.pdf)
* [PoX的战争——区块链技术在金融工具中的应用]({{ site.url }}/PoX的战争.pdf)
    
## 手册导航

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

## 版权声明

原创文章的版权均属于[梧桐树](https://wutongtree.com)，如需合作请联系[我们](mailto:{{ site.author.email }})。
