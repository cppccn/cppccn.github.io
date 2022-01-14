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
* [{{ post.title }}]({{ post.url }})
<br/>
<br/>
{{ post.description }}
{% endfor %}
