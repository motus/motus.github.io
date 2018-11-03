---
layout: default
title: Recent posts
---
# Recent Posts

{% for post in site.posts %}

---

#### {{ post.date | date_to_string }}

{{ post.excerpt }}

* **Read more:** [{{ post.title }}]({{ post.url }})

{% endfor %}
