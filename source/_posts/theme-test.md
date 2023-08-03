---
title: theme-test
date: 2023-08-03 21:03:31
excerpt: A test page on features of redefine theme
---

{% tabs tests %}
<!-- tab note -->
{% notel red fa-fire custom_title %}
This is a big note module provided by Redefine.
{% endnotel %}

{% note info %}
This is a small note module(info) provided by Redefine.
{% endnote %}
{% note warning %}
This is a small note module(warning) provided by Redefine.
{% endnote %}
<!-- endtab -->
<!-- tab button -->
{% btn regular::Redefine-button::https://redefine-docs.ohevan.com/modules/buttons::fa-solid fa-circle-info %}
<!-- endtab -->
<!-- tab folding -->
{% folding gray::folder %}
This a folding module provided by Redefine.
{% endfolding %}
<!-- endtab -->
{% endtabs %}