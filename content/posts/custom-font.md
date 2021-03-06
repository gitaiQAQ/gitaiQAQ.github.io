---

layout:     post
title:      "从自定义字体来看编码和显示的低耦合性"
date:       2017-10-28
author:     "Gitai"
categories:
    - 奇技淫巧
tags:
    - 自定义字体
    - 记录
---

![Glyphr Studio](https://i.loli.net/2017/10/28/59f42a350e2fd.png)

编程作业中难免不出现做计算器这种然并卵，但是涉及知识复杂，又难拔高的任务。

对于本文，不过就是我在这瞎说，给这个月凑一篇，不过也希望可以产生新的启发。

<!-- more -->

首先，按照我的理解，对整个程序进行解耦

1. 编译输入内容，构造中缀表达式序列
2. 构造二叉树
3. 后序遍历，生成后缀表达式
4. 用栈计算结果

细节在北大 C 语言程序设计（二），这个 MOOC 项目中有叙说。

但是中缀表达式是可以直接转换成后缀表达式的，上述流程直接缩短成 3 步。

自然看标题就发现，我也不是在说这个。

在第一步构造中缀表达式序列，之前的分割输入内容可能会遇到这样的例子 

$x^2,tan30,sin^{-1}{30}$

看这几个例子就让人头大，我们从显示和计算 2 个角度的讨论他。

1. 正常的界面组件，哪有能显示上标这种东西的，修改通用的编辑框，成本不得不说略高。
2. 在看上面的表达式，虽然可以用一堆判断拆开，但不免让代码更加混乱。

自定义字体常用于制作各种图标等'歧途'，那不妨在这里试试。

首先总结常见数学表达式用得上的各种符号，然后丢一边。

* 0～9
* +-*/
* 部分小写字母(e)

遂将 A~Z 直接替换成我们需要的特殊字形

* A: $sin$
* B: $sin^{-1}$
* C: $cos$
* D: $cos^{-1}$
* E: $tan$
* F: $tan^{-1}$

于是上述表达式就变成这样

* $x^2$: 平方符号早就存
* $tan30$: E30
* $sin^{-1}{30}$: B30

相比 `sin^{-1}{30}` 可见 `B30`， 解析起来是毫无压力。

对于该方法，可以继续拓展到诸如 $log,ln$，以及更多多字符组成的表达式。

于是就用自定义字符解决了多字符表达式的 2 个小问题。