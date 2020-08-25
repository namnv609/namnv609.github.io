---
layout: base
permalink: /editors-choice/
title: Editor's Choice
---

{% assign posts_list = site.posts | where: "is_editor_choice", true %}
{% include list.html %}
