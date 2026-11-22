---
layout: default
title: Documents
---

# Documents

{% assign articles = site.pages | where_exp: "page", "page.path contains 'posts/'" | where: "published", true | sort: "date" | reverse %}
{% for article in articles %}

- [{{ article.title }}]({{ article.url | relative_url }}){% if article.description %} — {{ article.description }}{% endif %}

{% endfor %}
