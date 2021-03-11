---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Education
======
* Tsinghua University, B.A., English (Linguistics), 2015, M.L. International Studies, 2018
* Johns Hopkins University, M.A., Economics, 2018
* University of Chicago, M.S., Computer Science, 2020
{% comment %}
* New York University, Ph.D, Neuroscience, 2026 (expected)
{% endcomment %}

Work experience
======
* International Monetary Fund: 2017-19, 2020-21 

{% comment %}
Skills
======
* Skill 1
* Skill 2
  * Sub-skill 2.1
  * Sub-skill 2.2
  * Sub-skill 2.3
* Skill 3
{% endcomment %}

Publications
======
  <ul>{% for post in site.publications %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
  
Talks
======
  <ul>{% for post in site.talks %}
    {% include archive-single-talk-cv.html %}
  {% endfor %}</ul>
  
Teaching
======
  <ul>{% for post in site.teaching %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>

{% comment %}
Service and leadership
======
* Currently signed in to 43 different slack teams
{% endcomment %}
