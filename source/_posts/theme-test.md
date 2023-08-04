---
title: theme-test
date: 2023-08-03 21:03:31
excerpt: A test page on features of redefine theme
---

- 不知为何选项卡模块每次都需要刷新后才能正常使用，不管是本地还是云端服务都是如此。或许是因为 all-minifier 插件的缘故？
- 大号提示块似乎只支持 Font awesome 图标的后半部分，例如 `font-fire`，若是 `font-brand font-qq` 这样的图标则会不予显示。

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