---
layout: default
---

{% for post in site.posts %}
<div style="opacity: 0.5">{{ post.date | date_to_string  }}</div>
<h2><a href="{{ post.url }}">{{ post.emoji }} {{ post.title }}</a></h2>
{% endfor %}

<center style="opacity: 0.5; margin: 1em">&bull;&ensp;&bull;&ensp;&bull;</center>

{% for page in site.misc %}
<h2><a href="{{ page.url }}">{{ page.title }}</a></h2>
{% endfor %}


### <a href="index1.html">Original index</a>

### Neat stuff (to be sorted):

- [The Ultimate Game Hacking Resource](https://github.com/dsasmblr/game-hacking)
- [The Ultimate Online Game Hacking Resource](https://github.com/dsasmblr/hacking-online-games)
- [vgce - Video Game Content Extraction](https://github.com/ohio813/vgce/tree/master/docs)
