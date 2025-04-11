---
layout: page
title: About
description: Come on
keywords: Jeffrey, 大海星
comments: true
menu: 关于
permalink: /about/
---

## [联系方式]

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}

{% if site.url contains 'double12gzh.github.io' %}
<li>
微信公众号：<br />
<img style="height:192px;width:192px;border:1px solid lightgrey;" src="{{ site.url }}/assets/images/qrcode.jpg" alt="double12gzh" />
</li>
{% endif %}
</ul>


## [会干些啥]

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
