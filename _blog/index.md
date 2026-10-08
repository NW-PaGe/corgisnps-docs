---
title: All Posts
layout: page
nav_order: -99999999
permalink: /posts/
---

{%- comment -%}
  Adding a post:
    1. Create a new Markdown file in `_blog/` (e.g. `_blog/my-new-post.md`).
    2. Give it front matter like:

         ---
         title: My New Post
         layout: page
         date: 2026-10-08
         author: Your Name
         description: One-line summary shown on this page.
         nav_order: -20261008   # minus the date as YYYYMMDD
         ---

    The negative date keeps the newest post at the top of the sidebar
    without renumbering older posts. Posts are published at /posts/<file-name>/.
{%- endcomment -%}

# Posts
{: .no_toc }

Notes, comparisons, and updates from the CorgiSNPs team.

{% assign posts = site.blog | where_exp: "p", "p.date" | sort: "date" | reverse %}
{% for post in posts %}
---

### [{{ post.title }}]({{ post.url | relative_url }})
<small>{{ post.date | date: "%B %-d, %Y" }}{% if post.author %} · {{ post.author }}{% endif %}</small>

{{ post.description }}
{% endfor %}
