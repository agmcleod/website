---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
---

My name is Aaron McLeod. I presently live & work in Canada as a Software Engineer. I work on web & mobile applications, while also exploring game development & rust in my spare time.

<h2 class="post-list-heading">Recent posts</h2>

<ul class="post-list">
  {%- for post in site.posts limit:5 -%}
    {% include post-item.html %}
  {%- endfor -%}
</ul>

<h3>
  <a href="/blog/index.html">See all posts</a>
</h3>