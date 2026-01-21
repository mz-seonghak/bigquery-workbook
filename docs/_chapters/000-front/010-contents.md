---
title: 목차
slug: contents
permalink: /contents.html
disable_toc: true
class: front-matter
---

{% assign sorted_chapters = site.chapters | sort: "path" %}
{% assign current_part = "" %}

{% for chapter in sorted_chapters %}
  {% assign path_parts = chapter.path | split: "/" %}
  {% assign folder = path_parts[1] %}

  {% if folder != "000-front" %}
    {% if chapter.layout == "part" %}
      {% if current_part != folder %}
        {% assign current_part = folder %}

### {{ chapter.title }}

{% if chapter.abstract %}
{{ chapter.abstract }}
{% endif %}

      {% endif %}
    {% else %}
- [{{ chapter.title }}]({{ site.baseurl }}{{ chapter.url }})
    {% endif %}
  {% endif %}
{% endfor %}
