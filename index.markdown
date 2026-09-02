---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
title: M00se-3
---
## Projects
{% assign cadlib_posts = site.tags.cadlib | sort %}

| {{ site.cadlib.full_project_name }} - {{ site.cadlib.short_description }} [{{ site.cadlib.remote_domain }}]({{ site.cadlib.repository_url }}) | 
| ------- | {% for post in cadlib_posts %}
| [{{ post.title }}]({{ post.url }}) | {% endfor %}
