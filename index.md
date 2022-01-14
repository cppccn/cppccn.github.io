---
layout: default
permalink: /
---

# Cappuccino

## Presentation ☕

Cappuccino is a free organisation focused on technological watch
- Rust \| C++ \| C \| WebAssembly
- Decentralised networking
- Nix \| NixOS \| NixOps \| DevOps as a Service
- React.js \| React Native


[GitHub](https://github.com/cppccn) / [Yvan Sraka](https://github.com/yvan-sraka) / [Adrien Zinger](https://github.com/adrien-zinger)

## Posts

{% for post in site.posts %}
* [{{ post.title }}]({{ post.url }}) - <i>By:{% for author in post.authors %} {{author}};{% endfor %}</i>
<br/>
<br/>
{{ post.description }}
{% if post.reviewers %}

<i>Reviewers: {% for reviewer in post.reviewers %} {{reviewer}};{% endfor %}</i>

{% endif %}
{% endfor %}


## About this page

This page is written in [Markdown](https://daringfireball.net/projects/markdown/), hosted by [GitHub Page](https://pages.github.com/), automatically converted to HTML by [Jekyll](https://jekyllrb.com) and looks like the authentic Markdown through [a fork of Hack CSS](https://github.com/yvan-sraka/hack), you can find the sources [here](https://github.com/yvan-sraka/yvan-sraka.github.io).
