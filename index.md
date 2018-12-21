---
layout: default
title: Home
---

{% for post in site.posts %}

### {{ post.date | date_to_string }}

{{ post.excerpt }}

{% if post.excerpt != post.content %}
**Read more:** [{{ post.title }}]({{ post.url }})
{% endif %}

---

{% endfor %}
