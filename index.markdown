---
layout: default
---


{% include career-profile.html %}
<hr>
{% unless site.data.data.sidebar.education %}
  {% include education.html %}
{% endunless %}

{% include experiences.html %}

<hr>
{% include skills.html %}
<hr>
{% include projects.html %}
