---
title: Rime 使用雾凇拼音
date: 2023-08-28 16:19:01
categories: 记录
tags:
    - rime
    - fcitx5
excerpt: 在 Rime 中使用雾凇拼音
---

## 前言

-   今日与群友讨论时得知了这个输入法，感觉颇为好用，便打算尝试一下。

## 安装

-   使用 git：

```bash
git clone https://github.com/iDvel/rime-ice.git
```

-   将该配置文件移动到 `~/.local/share/fcitx5/rime` 中。

## 自定义配置

-   关闭双拼，修改 `default.yaml` ：

```yaml
schema_list:
    - schema: rime_ice
    # - schema: double_pinyin
    # - schema: ...
```

-   增加候选词数量，修改 `default.yaml`：

```yaml
menu:
    page_size: 8
    # ...
```

---

-   大部分配置都已经经过了详细的预配置，基本不需要再修改。

## 体验

-   使用了几天，感觉确实好用。相较于我之前使用的配置，这个配置的模糊拼音更不容易出现错误识别，且候选词都更加合理了。
-   中英文混合输入（melt_eng）的体验也要远优于先前的配置，且单独使用的英文输入法也可以正常工作，不过对我而言英文输入法的使用还是有点不习惯。
