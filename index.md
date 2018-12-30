---
layout: default
---

{% for post in site.posts %}
<h2><a href="{{ post.url }}">{{ post.title }}</a> <span style="font-size: 75%; opacity: 0.5">{{ post.date | date_to_string  }}</span></h2>
{% endfor %}

### <a href="static-recompilations">Some interesting static recompilation projects</a>

### <a href="index1.html">Original index</a>

### Neat stuff (to be sorted):
- https://github.com/ohio813/vgce/tree/master/docs
- [The Ultimate Game Hacking Resource](https://github.com/dsasmblr/game-hacking)
- https://github.com/dsasmblr/hacking-online-games
