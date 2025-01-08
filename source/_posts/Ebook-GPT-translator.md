---
title: Ebook-GPT-translator
date: 2023-08-08 20:12:26
categories: 工具使用
tags:
    - python
    - chatgpt
    - micromamba
excerpt: 使用 ChatGPT 翻译外语文档
---

最近朋友发给我一个 [Github 项目](https://github.com/jesselau76/ebook-GPT-translator/tree/main)，简单查阅一下 README 后，发现是一个使用 chatGPT api key 来获取其服务并对本地外语文档进行翻译的项目。对于 chatGPT 的翻译能力我是早有耳闻，于是便来了兴趣。

## 创建虚拟环境 && 安装依赖库

-   本人使用 [micromamba](https://mamba.readthedocs.io/en/latest/user_guide/micromamba.html) 创建虚拟环境，命令如下：

```bash
micromamba create -n GPT-translator -f requirements.txt -c conda-forge
```

_（其中 `requirements.txt` 为项目自带）_

-   运行该命令时出现报错：

```plaintext
error    libmamba Could not solve for environment specs
    The following package could not be installed
    └─ mobi does not exist (perhaps a typo or a missing channel).
critical libmamba Could not solve for environment specs
```

-   经检查发现，导致该错误的原因是 conda-forge 频道内并没有 `mobi` 这个库。经测试后发现 default 频道也没有。
-   解决方案：使用 pip 安装该包。

-   spec file:

```yaml
name: GPT-translator
channels:
    - conda-forge
dependencies:
    - pdfminer
    - pdfminer.six
    - openai
    - tqdm
    - ebooklib
    - bs4
    - python-docx
    - chardet
    - pandas
    - lxml
    - openpyxl
    - pip:
          - mobi
```

-   安装指令：

```bash
micromamba install -n GPT-translator -f GPT-translator.yml -c conda-forge
```

## 获取 chatGPT api key

-   八仙过海，各显神通！

## 更改配置

-   将 `settings.cfg.example` 改名为 `settings.cfg`。
-   将获取到的 chatGPT api key 填入 `openai-apikey` 一项内。
-   将 `bilingual-output` 的值更改为 `False` 来关闭保留双语。

## 测试功能

-   说实话这个翻译模块最大的问题应该就是速度了，仅仅是翻译该项目的 README 文件（8 kb）都要花费 2 分钟左右。

-   从 Neovim 的 Github wiki 复制了一篇文稿权作测试：

{% folding white::原文 %}

Vim is a powerful text editor with a big community that is constantly growing. Even though the editor is about two decades old, people still extend and want to improve it, mostly using Vimscript or one of the supported scripting languages.

Motivation

Over its more than 20 years of life, Vim has accumulated about 300k lines of scary C89 code that very few people understand or have the guts to mess with.

Another issue is that as the only person responsible for maintaining Vim's big codebase, Bram Moolenaar, has to be extra careful before accepting patches, because, once merged, the new code will be his responsibility.

These problems make it very difficult to have new features and bug fixes merged into the core. Vim just can't keep up with the development speed of its plugin ecosystem.

Solution

Neovim is a project that seeks to aggressively refactor Vim source code in order to achieve the following goals:

-   Simplify maintenance to improve the speed that bug fixes and features get merged.
-   Split the work among multiple developers.
-   Enable the implementation of new/modern user interfaces without any modifications to the core source.
-   Improve the extensibility power with a new plugin architecture based on coprocesses. Plugins will be written in any programming language without any explicit support from the editor.

By achieving those goals new developers will soon join the community, consequently improving the editor for all users.

It is important to emphasize that **this is not a project to rewrite Vim from scratch** or transform it into an IDE (though the new features provided will enable IDE-like distributions of the editor). The changes implemented here should have little impact on Vim's editing model or Vimscript in general. Most Vimscript plugins should continue to work normally.

The following topics contain brief explanations of the major changes (and motivations) that will be performed in the first iteration.

Migrate to a CMake-based build

The source tree has dozens (if not hundreds) of files dedicated to building Vim on various platforms with different configurations, and many of these files look abandoned or outdated. Most users don't care about selecting individual features and just compile using `--with-features=huge`, which still generates an executable that is small enough even for lightweight systems by today's standards.

All those files will be removed and Vim will be built using CMake, a modern build system that generates build scripts for the most relevant platforms.

Legacy support and compile-time features

Vim has a significant amount of code dedicated to supporting legacy systems and compilers. All that code increases the maintenance burden and will be removed.

Most optional features will no longer be optional (see above), with the exception of some broken and useless features (e.g. NetBeans and Sun WorkShop integration) which will be removed permanently. Vi emulation will also be removed (setting `nocompatible` will be a no-op).

Platform-specific code

Most of the platform-specific code will be removed and libuv will be used to handle system differences.

libuv is a modern multi-platform library with functions to perform common system tasks, and supports most unixes and Windows, so the vast majority of Vim's community will be covered.

New plugin architecture

All code supporting embedded scripting language interpreters will be replaced by a new plugin system that supports extensions written in any programming language. Compatibility layers will be provided for legacy Vim plugins written in some of the currently supported scripting languages such as Python or Ruby. Most plugins should work on Neovim with little modification, if any.

Moreover, GUIs are implemented as plugins, decoupled from the Neovim core.

See the Plugin Architecture page for a detailed overview.

{% endfolding %}

{% folding white::翻译稿 %}

介绍
Vim 是一个功能强大的文本编辑器，拥有庞大的社区，不断发展壮大。
尽管这个编辑器已有将近二十年的历史，但人们仍在扩展和改进它，主要使用 Vimscript 或支持的脚本语言。
动机 在 Vim 的二十多年的发展过程中，累积了大约 30 万行可怕的 C89 代码，很少有人理解或敢于去修改。
另一个问题是，作为唯一负责维护 Vim 大型代码库的人，Bram Moolenaar 在接受补丁之前必须格外谨慎，因为一旦合并，新代码就成为他的责任。
这些问题使得将新功能和错误修复合并到核心中变得非常困难。
Vim 无法跟上其插件生态系统的发展速度。
解决方案 Neovim 是一个项目，旨在积极重构 Vim 源代码，以实现以下目标：简化维护，提高错误修复和功能合并的速度。

分配工作给多个开发人员。
在核心源代码不进行任何修改的情况下，实现新的/现代的用户界面。
通过基于协处理器的新插件架构，提高可扩展性能力。
插件将可以使用任何编程语言编写，而不需要编辑器的明确支持。
通过实现这些目标，新的开发人员将很快加入社区，从而改进编辑器适用于所有用户。
重要的是强调，这不是一个从头开始重写Vim或将其转变为IDE的项目（尽管所提供的新功能将使编辑器可以像IDE一样发布）。
这里实施的更改对Vim的编辑模型或Vimscript总体上应该没有太大影响。
大多数Vimscript插件应该继续正常工作。
以下主题简要解释了第一次迭代中将要进行的主要变更（及其动机）。

迁移到基于CMake的构建。
源代码树中有数十个（如果不是数百个）文件专门用于在不同配置下构建Vim在不同平台上构建的文件，并且其中许多文件看起来已经被放弃或过时了。
大多数用户不关心选择单个功能，只需使用--with-features=huge进行编译，这仍然会生成一个在如今标准下，甚至对于轻量级系统来说都足够小的可执行文件。
所有这些文件都将被删除，Vim将使用CMake构建，CMake是一个现代的构建系统，为最相关的平台生成构建脚本。
遗留支持和编译时特性。
Vim有大量的代码专门支持遗留系统和编译器。
所有这些代码增加了维护负担，将被删除。
大多数可选的特性将不再是可选的（参见上文），除了一些已经损坏和无用的特性（例如NetBeans和Sun WorkShop的集成），它们将被永久删除。

Vi的仿真也将被移除（设置nocompatible将不起作用）。
平台特定的代码大部分将被移除，并且将使用libuv来处理系统差异。
libuv是一个现代化的多平台库，具有执行常见系统任务的功能，并支持大多数的Unix和Windows系统，因此大部分Vim的用户群将被涵盖。
新的插件架构所有支持嵌入式脚本语言解释器的代码将被一个新的插件系统替代，该系统支持用任何编程语言编写的扩展。
为一些当前支持的脚本语言（如Python或Ruby）编写的传统Vim插件提供了兼容层。
大多数插件在Neovim上应该可以少量修改甚至不用修改就能工作。
此外，GUIs被实现为插件，与Neovim核心解耦。
详细概述请参阅插件架构页面。

{% endfolding %}

{% note info fa-circle-exclamation %}
耗时：1 min 3 s
{% endnote %}

-   可以看到，翻译效果还是很不错的，扫了几眼，并没有发现任何翻译错误。
-   然而，格式上的问题较为严重，很多原文中的标题都变成了正文开头的一个空格隔开的句子，不过总的来说无伤大雅。

## 总结

-   对于英语能力并不理想但仍有需求阅读英文资料的人来说，使用该项目作为阅读方案不失为一种不错的选择。
-   然而，如果读者有能力流畅的阅读英文资料，那么使用该项目作为阅读方案就显得有点浪费时间和多此一举了。
