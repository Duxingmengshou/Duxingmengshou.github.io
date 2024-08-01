---
layout: archive
title: "网站地图"
permalink: /sitemap/
author_profile: true
---

{% include base_path %}

网站上所有帖子和页面的列表。对于机器人来说，还有一个 XML 版本可供摘要。

<h2>页面存档</h2>
{% for post in site.pages %}
  {% include archive-single.html %}
{% endfor %}

<h2>博客记录</h2>
{% for post in site.posts %}
  {% include archive-single.html %}
{% endfor %}

<!-- {% capture written_label %}'None'{% endcapture %} -->

<!-- {% for collection in site.collections %}
{% unless collection.output == false or collection.label == "posts" %}
  {% capture label %}{{ collection.label }}{% endcapture %}
  {% if label != written_label %}
  <h2>{{ label }}</h2>
  {% capture written_label %}{{ label }}{% endcapture %}
  {% endif %}
{% endunless %}
{% for post in collection.docs %}
  {% unless collection.output == false or collection.label == "posts" %}
  {% include archive-single.html %}
  {% endunless %}
{% endfor %}
{% endfor %} -->