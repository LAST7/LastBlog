---
title: 2024 年终总结
date: 2024-12-30 22:17
categories: 随笔
excerpt: 在这稳步下沉的大船上度过又一个荒废的地球年
---

## 前言

-   2024 就这么结束了。我的心中一片茫然。
-   时间既没有停歇也没有加速，是我的感官模糊了，它们被分散到了其他地方。
-   只要目光不盯紧钟表的指针，它们就疯狂地转动不停。

## 2024 年

### 硬件更新

#### 手机

-   我这用了四年的 iPhone 11 终于是可以退役了。经过接连几次的系统更新之后他已经卡的让人无法接受了。
-   本人此前用过的两部手机都是 iPhone（8，11），对于苹果已经积攒了十足的怨气。其在硬件上的吝啬，技术上的不思进取和坐地起价的蛮横态度属实让我气愤不已的同时感到匪夷所思。类似于 NFC 和屏下指纹这样简单好用的功能迟迟不出，高分和高刷屏被价格高昂的 pro 型号独占，简直是把“赚钱”二字写在脸上了。
-   被其粉丝吹捧的 IOS 系统更是让我感到恶心不已。究其原因，以下几点：

    -   系统封闭。想安装什么软件还得看 App Store 里有没有。侧载安装每周都需要链接电脑刷新签名，无言了。
    -   卡顿。四年前该手机刚入手的时候系统流畅度确实无可挑剔，然而在接连几次系统更新之后不知怎的越来越卡。对于计划报废这些说辞我不置可否，但是卡顿的系统着实让我感到非常恶心。
    -   微信同步消息简直慢的让人不可置信。每次打开微信都需要等待 5s 以上的时间等待消息同步，我简直无法理解。询问旁人后得知，这就是 IOS 独有的问题。

-   如今终于可以摆脱这恶心的品牌，快哉快哉！

---

-   新手机是一部 One Plus Ace3v。由于我不需要玩手游也没有拍照需求，对于手机的要求仅仅就是个即时通讯器而已。再加之对于苹果长久以来各种毛病的怨气，我选择了一款价格便宜的安卓系统手机。
-   从 5 月份一直用到现在，说实话体验相当良好。2k 屏，高刷，NFC，屏下指纹以及应用分身。这些在苹果那里倘若不买价格 1w+ 的 pro 型号，通通没有。

---

-   当然，购买该平牌手机还有一个原因就是 root 方便，甚至还有一加盒子这样的社区工具，好评。
-   依稀记得本人一开始 root 的时候一股脑安装了许多模块，大概是其中某个模块发生问题，导致手机无法开机，也就是俗称的“变砖”。好在提前安装了一个“自动救砖”的模块，重启三次之后自动隔离了其他模块，算是有惊无险。

-   说实在的，本人对于刷 root 权限这件事其实也没有那么执着，仅仅是抱着“玩一玩”的心态去尝试的。对于各家厂商纷纷停止向用户开放 root 权限，我其实是可以理解的。毕竟，开屏广告没了的话，广告收入会削减不少。以及所谓的“用户数据也是手机厂商宝贵的资产”。
-   当然，理解归理解，该骂还是要骂。资本的最终目的还是为了增值。

#### 电脑

-   五月份又拜托 Jazz 哥，在旧台式的基础上翻新配置组了台新机器。配置是 AMD R7 5700x 和 AMD 6750 GRE 12g，迈出了逃离 N 卡的第一步，好！也是得益于此，后续我再次 [折腾 Hyprland](https://blog.imlast.top/2024/12/18/2nd-hyprland/) 的体验好了不少，也是给我用上 Hyprland 了（喜）。
-   当然，选择 AMD 的考量主要还是在于性价比。N 卡借助人工智能领域飞黄腾达之后，连带着价格也跟着水涨船高。在我发现一张 N 卡的价格要占到整机预算的 50% 以上之后，就再也不想考虑 N 卡了。
-   换了台式之后最大的感受不是性能的提升，而是骤减的散热压力。再也不会像曾今用笔记本时那样，cpu 动不动就涨到 90°C 了。

-   顺带的换了个鼠标。随便挑了一个国产的牌子，不曾想该鼠标续航能力简直惊人——充一次电能用将近两个月，属实令我惊讶不已。

#### Neo Ergo

-   购买了一把 Alice 布局的客制化键盘。阳极银、紫镜面配重。ALU 定位板，无绵 Gasket。轴体选了紫薇轴，我很喜欢这种脆响的声音。键帽是随便选的一套植物园配色。

![neo-ergo.jpg](https://s2.loli.net/2025/01/01/rQSlRhGBPKeaDkN.jpg)

-   使用体验相当优秀，声音也很令我满意。把底绵和夹心绵卸下之后，声音的脆响程度又上了一层楼，甚好，甚好。

### 开源之夏

-   今年暑假参加了[开源之夏](https://summer-ospp.ac.cn/)（OSPP）活动，具体内容是为 [MQTTX](https://github.com/emqx/MQTTX) 添加 AVRO 编码支持。
-   说实话，在挑选项目的时候我对自己有些过于不自信了，这个项目对我而言似乎是有点太过简单了……总共花在上面的时间恐怕不足三个星期，其中的大部分时间还都是花在沟通上，技术上可以说没遇到任何难题。
-   项目导师可以说十分尽职尽责，事无巨细的解答了我对于项目许多的问题：[相关博客](https://blog.imlast.top/tags/mqttx/)
-   开发工作完成后导师还邀请我加入了 GitHub 上 emqx 的组织，可惜此后我拿不出那么多时间就继续开发该项目，总感觉有些对不起导师。不过在如今潦草的经济大环境下，他应该能理解我。

### Hyprland

-   也是给我用上 Hyprland 了。经过重重险阻解决了大部分 BUG，详见 [这篇博客](https://blog.imlast.top/2024/12/18/2nd-hyprland/)。

![screenshot_29122024_052933.jpg](https://s2.loli.net/2024/12/29/kI57gHsNOZinFcu.png)

### 南京大学 ICS

-   上半年学期末考试之前，因为懒得复习所以找了点别的事情干。依稀记起此前群友有推荐过这个课程，便找来讲义想着体验一番。 哪成想一下陷入其中，断断续续的一直写到了 PA3。

-   这门课可以说是刚好处在我能力边缘的位置——努努力就可以达到其中的要求，在这个过程中还能学到不少计算机系统的运作原理。长时间阅读文档和 DE 后终于调通了某个功能之后，那种喜悦能让我感觉自己至今为止的努力都没有白费。

-   虽然用了一种很“俗套”的说法，但是那一瞬间的感触确实如是。

---

-   完成 PA2 结尾处选做任务声卡后录的纪念视频：

<iframe width="100%" height="468" src="//player.bilibili.com/player.html?isOutside=true&aid=113527444544674&bvid=BV1KtBBYXETA&cid=26903250566&p=1&autoplay=0" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"></iframe>

### 服务器

-   去年租的服务器到期了，第二年续期的费用居然高达接近 1k。记得当时租用一年的费用也不过 100 左右，果然优惠之后新客户才能享受，老客户和狗不得入内。
-   在各种主机类的博文中翻来找去，找到一个野草云的半价优惠码。于是浏览了一遍各型号的配置，租了一台 2C4G 的 HK 服务器，费用 ¥150 / 年。

---

-   由于懒惰和拖延，直到前一个服务器临近过期我才开始迁移上面的服务和配置。由于匆忙，遗漏了许多配置文件，最终这些破事全都加倍返还了回来……
-   另外不知为何，部署在这台服务器上的 Twikoo 后端导致博文中加载评论奇慢无比。但是我已经不愿意再为此付出时间精力了，能用就行吧。

### 学校

#### 辅修专业的毕业设计

-   24 年上半年，辅修计算机专业迎来了最后的毕业设计环节。
-   分配给我的指导老师实在是让我感觉无语至极，其漫不经心的态度和极差的沟通能力让我数次接近于失去耐心。整个毕业设计的项目源码总共就花了不到两个星期的时间，后面由于沟通不畅导致格式问题愣是修改了一个多月。
-   论文工作和家事交织在一起，这个两个月耗干了我所有的心性和精力。我不认为在这种情况下我还能拿出复习考研的精力了，那时候我只想着休息。
-   和指导老师沟通后，他为我宽限了一些论文的截止日期。看在这份上，我还是配合了他的工作和修改意见，尽管数次被气的喘不上气。

#### 恼人的论文课

-   主专业论文开题在即，或许是为此前被退回的论文感到警醒，学校安排所有学院都添加了一门论文课。
-   我们学院安排了一位经管专业的晋洪涛老师为我们授课。实话实说，这门课简直是一坨屎：

    -   前四周的课都在讲如何为论文拟题，最后发现论文的题目是由指导老师拟定的。
    -   所有的范例和论文结构讲解都只适用于经管领域的论文，与我们信管可以说是毫不相干，毫无帮助。
    -   该老师简直是精神病，排在大四——许多同学都有升学和实习安排的时期——的课程，愣是想尽办法的提升出勤率：

        -   旷课代价是平时分 -30，连请假都要扣分
        -   每节课都有不定次数，不定时间的签到
        -   每次签到都附带题目，甚至有用于验证的题目
        -   连已经出国留学（预科）的同学他都不予容忍，硬是逼得同学回国来上课

    -   该课程的作业和期末考核可以说是毫无意义：

        -   论文的具体内容涉及计算机领域的知识，这位经管出身的老师显然是无能为力。
        -   计算机专业的毕业设计论文结构基本固定，完全不需要加以额外指导。

    -   _虽然现在还没到学期末，不过可想而知最终的期末成绩不可能太好看。_

---

-   综上所述，我完全无法理解这门课的意义何在——该课程既没有提供有价值的知识，反而还极为注重诸如考勤和作业格式之类的形式要求。
-   学期中的时候我便实在无法忍受，去找辅导员攀谈，得到的结果也仅仅是“可以向学校提出意见”。
-   这位老师让我想起在 [上海交通大学生存手册](https://survivesjtu.gitbook.io/survivesjtumanual) 看到的一则观点：

    > …… 我们关注是否点名，重点应该看的是这个老师面对学生的态度。有的老师睁一只眼闭一只眼，即使不来上课，也不会太为难学生。**而另外一些比较自卑的老师却把学生不来上课当成对自己的奇耻大辱，甚至会想出课前课后点两次名外加数人头这样的手段防止学生翘课。**其实，如果老师想不计代价查出谁没来上课，这真的是太简单的一件事情。

-   这位晋洪涛老师算是让我见识到什么才是大学里最畜生、最恶心的老师，令人作呕。

#### 小组作业

-   最后一学期的一门信息系统课程，其主要的课程目标是安排我们以小班（25 人左右）为单位，其中每个小班分为 6 个小组，设计并开发一个完整的网页应用。
-   且不提这门课程提供的分工方式如何逆天，我想，只要是经历过小组作业的人都能预想到这将会是怎样的“盛况”，更不要提在我们这个不学无术的学校和学院。

-   同级其他班的绝大部分小组都选择了外包，对此我丝毫没有感到意外。 真正令我感到瞠目结舌的是，居然有人连外包都整不明白？而且很不幸，这一组人就出现在我们小班之中，担任着前端开发的要职。

---

-   这一组的同学先是外包了一份和第一组递交的设计稿毫不相关的前端网页，在后续的小组提醒之后他们才又递交了一份新的。然而，这份新的前端网页也同样和设计稿毫无交集……
-   就这么扯皮到了 Milestone 2 的提交节点，后续流程中的小组不得以将情况告知了课程老师，最终老师的方案是额外给我们班开个线上会议帮助解决进度问题。
-   谁成想，前端组居然干脆直接缺席线上会议，而且还缺席了两次，其中第二次还有人专门去通知其组长开会。最后外教老师不得以在线下授课期间为了我们小班单独开会解决问题，会上后三组的同学连外教老师说的话都听不懂，这给予了我极大的震撼……

-   最终，我帮助他们拟定了开发计划和时间节点，外教老师为我们小班延期了 2 周 ddl。
-   毫不意外的，最后他们做了一坨屎出来。前端组找的外包用现成的博客框架改了一个网站出来，简直蠢的让人无话可说。更不要提后端开发组使用了已经老掉牙的 PHP 开发，对此我已经无话可说了。

-   早早完成任务的第一组中一位已经保研的同学还曾忧心忡忡的来询问我具体的进度，说这是他第一次担心会在学校课程中挂科。这太荒唐了，简直令我捧腹大笑。

---

-   说真的，这个项目可以说是一点难度都没有。现在互联网应用的开发门槛已经非常低了，毫不夸张的说，让我独自一人开发都要不了两个星期。
-   至于我的同学们是如何落得如此地步，我只能说可能这就是普通高校的现状吧。

---

-   在我还住在学校里的时候，前端组的组长住在隔壁宿舍。我对他的印象基本上就是一个家境富裕但生活“难以自理”的家伙。
-   依稀记得在大一的时候曾在某门通识课就和他在同一组，大一的他和现在可以说是同一副德行：那时候他忙着物色女孩，小组作业分工到他头上的工作一直到截止之前都未能完成。至于这一次他到底在忙些什么，我就无从得知了。
-   我没法理解是什么经历促成了他这样对他人漠不关心的性格，以及对学习接近于零的热情。对于他我已经不想说什么“应试教育摧毁了学生的精神”之类的话了，或许他在其他的领域能够有所建树吧。

## 总结

-   2024 年就像一阵风一样转瞬即逝，正如同在去年的年终总结中写下的， _对于时光荏苒的感慨都已经是家常便饭了_ 。
-   2024 是苦难和解脱的一年，布满了独属于我这个年龄与这个时代的焦虑和迷茫。

-   在这样荒蛮又虚无的世界里，我要成为谁？
