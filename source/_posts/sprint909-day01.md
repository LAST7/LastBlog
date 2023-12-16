---
title: sprint909-day01
date: 2023-12-11 20:15:43
categories: 记录
tags:
    - vim/nvim
    - gawk
excerpt: Calorie Counting
---

## Day 01

-   puzzle: [Calorie Counting](https://adventofcode.com/2022/day/1)

## Part 1

### Re

1. 将每只精灵所拥有食物的卡路里数值用加号连接：

```vimscript
:%s/\v\n$@!/+
```

-   其中，`$@!` 确保“换行符号后直接为行尾”，也就是空行前的换行符不会被替换为加号。

2. 计算每一行的数值：

```vimscript
:%s/.*/\=eval(submatch(0))
```

-   其中，`submatch(0)` 表示上一次匹配完成的结果。

3. 排序并留下最大的一行：

```vimscript
:%!sort -nr | head -1
```

-   其中，`!` 表示跳出 vim 的命令模式转而使用系统环境。

### global command

1. 将每只精灵所拥有食物的卡路里数值用加号连接：

```vimscript
:g/\d/-j | s / /+
```

-   这里其实包含了两个步骤：
    -   将每一行数字*空格* 连接起来
    -   将空格替换为加号

2. 计算每一行的数值：

```vimscript
:%! bc
```

3. 略

## Part 2

-   计算每只精灵所携卡路里总额同上。

-   计算前三名的和：

_total_of_top_three.gawk:_

```gawk
{s += $1}
!NF {
    a[NR] = s;
    s = 0;
}
END {
    asort(a);
    for (i=length(a);i>length(a)-3;i--)
        s+=a[i];
    print(s);
}
```

```bash
gawk -f total_of_top_three.awk calorie_input
```

或者:

```vimscript
:%! head -3 | paste -sd+ | bc
```

-   其中，`paste` 是一个用于合并文件的列的命令。
-   `-s`：串列进行而非平行处理，用于将单个文件的多行数据合并为一行显示。
-   `-d+`：将分隔符设定为加号。

## Exercise

1. first:
   `:g/\d/-j | s / /+/ | .!bc -lq`
   then:
   `%!sort -rn | head -3`
   finally:
   `:g/\d/-j | s / /+/ | .!bc`

2. `:%s/\(\d\)\(\n\)/\1+/g`：this is anti-human for sure. This command seeks for the "\n" symbol after digits and replace them with the plus sign. Seems that the result are not addressable for the next coming up command, but I can't provide a better answer.
