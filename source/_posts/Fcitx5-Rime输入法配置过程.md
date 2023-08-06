---
title: Fcitx5-Rime输入法配置过程
date: 2023-08-06 00:53:46
categories: 记录
tags:
    - 配置
    - 输入法
excerpt: Rime 的~~懒人~~配置过程
thumbnail: "https://s2.loli.net/2023/08/06/nUTmGeHNisgo7rB.webp"
---

## 动机

-   先前使用的是 [四叶草输入法](https://github.com/fkxxyz/rime-cloverpinyin)，但是总有些十分耳熟能详的高频词汇打不出来，抑或是出现莫名其妙的候选词，令我十分不爽。于是便一直想着更换一个输入法引擎，恰逢此日网上冲浪时浏览到[这篇博客](https://blacksand.top/2021/10/18/fcitx5-rime%E4%B8%AA%E4%BA%BA%E9%85%8D%E7%BD%AE%E4%BB%A5%E5%8F%8A%E8%B8%A9%E5%9D%91%E7%82%B9%E8%A7%A3%E5%86%B3%E7%AE%80%E8%A6%81/)，遂依葫芦画瓢。

## 前人栽树后人乘凉

-   由于 Rime 的输入法配置起来有些复杂，而本人也没什么兴趣研究，遂选择作为后人乘凉。
-   前人栽树：[接近原生的鼠须管 Rime 配置](https://github.com/wongdean/rime-settings)
-   感谢 wogdean 慷慨分享的配置文件。

{% notel yellow fa-triangle-exclamation 关于字体 %}

-   该配置文件自带了两个字体文件，经查询得知是花园明朝字体，为的是解决汉字的生僻字乱码问题。不过我感觉目前的字体使用并未出现过乱码，遂没有安装。
-   结果就是 rime 的使用大体没有问题，偶尔会出现乱码候选词，问题不大。

{% endnotel %}

## 自定义配置

### 全拼

-   由于本人不会使用双拼，所以将该配置中默认启用的小鹤双拼关闭，只使用全拼。
-   修改 `default.custom.yaml`：

```yaml
schema_list:
    # - schema: double_pinyin_flypy # 小鹤双拼
    - schema: luna_pinyin # 明月全拼
    # - schema: double_pinyin # 自然码
```

{% note info  %}
话说回来，双拼是什么？
{% endnote %}

### 模糊拼音

-   由于本人才疏学浅，时常会分不清某个字到底是前鼻音还是后鼻音，所以选择开启模糊拼音。
-   修改 `luna_pinyin.custome.yaml` （取消注释）：

```yaml
- derive/([ei])n$/$1ng/ # en => eng, in => ing
- derive/([ei])ng$/$1n/ # eng => en, ing => in
```

### 中英混输

-   该配置中自带了一个支持中英混输的英文输入法： `easy_en`。
-   安装教程来自 [Github](https://github.com/BlindingDark/rime-easy-en#%E6%89%8B%E5%8A%A8%E5%AE%89%E8%A3%85-easy_en)
-   修改 `default.custom.yaml`：

```yaml
schema_list:
    # - schema: double_pinyin_flypy # 小鹤双拼
    - schema: luna_pinyin # 明月全拼
    - schema: easy_en # English
    # - schema: double_pinyin # 自然码
```

-   修改 `luna_pinyin.custom.yaml`（取消注释）：

```yaml
__include: easy_en:/patch
```

-   _不知为何通过明月输入法和 easy_en 混用的时候一切正常，但是单独启用 easy_en 的时候则完全无法提供候选词，也无法自动插入空格。_

### 配置主题

-   研究了半天 `squirrel.custom.yaml` 才发现这是 Mac 专属的配置文件……

---

-   原本我使用的是 [catppuccin](https://github.com/catppuccin/fcitx5) 主题，现在有点看腻了，遂更换为 [dracula](https://github.com/drbbr/fcitx5-dracula-theme) 主题。然而这个主题有两个问题：

1. 界面太大了
2. 拼音内容不显示在输入法框内的时候，输入法界面是不透明的

-   前者通过修改配置文件中 `Margin` 的值解决了，后者则无计可施，去 Github 提了 [issue](https://github.com/drbbr/fcitx5-dracula-theme/issues/5)。

{% notel blue fa-clock 更新 %}
经该主题作者提示，我意识到输入法界面看上去不透明是因为其背景被 picom 给模糊了，在 picom.conf 中加入以下配置即可解决：

```conf
# ...
blur-background-exclude = [
    # ...
    "class_g = 'fcitx'", # new line
]
# ...
```

{% endnotel %}

### 其他

-   删除了 `luna_pinyin.mingxing.dict.yaml`。
-   在 `custome_phrase.txt` 中添加/修改了几个用户自定词汇。

## ~~和 polybar 的联动~~

-   在 Github 上发现了一个可以在 polybar 上实时显示当前输入法状态的 [script](https://github.com/RRRRRm/polybar-fcitx5-script)，安装后发现不起作用，遂删除。

## 设置 bspc 窗口规则

-   本人使用的是 bspwm 窗口管理器，我希望 Fcitx Configuration 能以浮动窗口的形式打开而非平铺。
-   在 bspwmrc 中加入以下规则：

```bash
bspc rule -a '*:*:Fcitx Configuration' state=floating center=true
```
