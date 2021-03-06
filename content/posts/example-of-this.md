---
layout:     post
title:      "最简单的调用链和作用域"
date:       2019-01-17
author:     "Gitai"
tags:
    - JavaScript
---

在《你所不知的 JavaScript》中，作者列举了 this 的几种绑定原则，最基本的就是默认绑定，在此就来细说一下这最简单的默认绑定。

```js
global = typeof window !== 'undefined' ? window : global;

function print(tag) {
    console.log(this === global, tag, "a", Object.getOwnPropertyDescriptor(global, "a"), Object.getOwnPropertyDescriptor(this, "a"));
    console.log(this === global, tag, "b", Object.getOwnPropertyDescriptor(global, "b"), Object.getOwnPropertyDescriptor(this, "b"));
}

function printCallers(caller) {
    if (caller!=null) {
        return printCallers(caller.caller) + " -> " + caller.name
    }
    return "global"
}

function f1() {
    a = 2;
    b = 2;
    console.log(printCallers(arguments.callee));
    print.call(this, "f1");
    f2();
}

function f2() {
    a = 3;
    b = 3;
    console.log(printCallers(arguments.callee));
    print.call(this, "f2");
    f3.call(this);
}

function f3() {
    console.log(printCallers(arguments.callee));
    print.call(this, "f3");
    new f4();
}

function f4() {
    console.log(printCallers(arguments.callee));
    print.call(this, "f4");
    setTimeout(f5, 0)
}

function f5() {
    console.log(printCallers(arguments.callee));
    print.call(this, "f5");
}

a = 1;
var b = 1;
print.call(this, "global");
f1();
```

这段代码在 Nodejs 环境能触发一个很有意思的和浏览器完全不同的差异。

<!-- more -->

首先解释一下前面几行用于输出必要内容的小工具。

```js
global = typeof window !== 'undefined' ? window : global;
```

绑定浏览器的 `window` 到 `global` 或者返回 Nodejs 的 `global` 对象

`print` 是用来打印当前作用域的 `a` 和 `b` 属性的状态，因为这里需要把作用域传递到对应函数内部，所以只能通过 `call` 这类方法调用；`printCallers` 用于打印调用链，调用链通过递归 `caller` 属性，但是会在严格模式下失效。

在浏览器下执行输出如下结果

```js
// 刚开始在全局作用域，`a` 和 `b` 均被绑定在上面
true "global" "a" 1 1
true "global" "b" 1 1

// 之后进入 `f1`，其 `this` 指向 `global` 所以输出 true 而且 `ab` 相等
global -> f1
true "f1" "a" 2 2
true "f1" "b" 2 2

// 进入 `f2` ，直接被调用，依然是静态的词法分析作用域，`ab` 被当前作用域改变
global -> f1 -> f2
true "f2" "a" 3 3
true "f2" "b" 3 3

// 使用 call 绑定了 `f2` 的作用域进来，所以也是 `3`
global -> f1 -> f2 -> f3
true "f3" "a" 3 3
true "f3" "b" 3 3

// 进入 `f4` `ab` 结果出现差异，发现这是的 this 已经不指向 global，因为 f3 通过 LHS 修改了全局变量 `ab` 的值，但是 this 是通过 new 创建的空对象，所以产生了差异
global -> f1 -> f2 -> f3 -> f4
false "f4" "a" 3 undefined
false "f4" "b" 3 undefined

// 最后通过 setTimeout，其中 this 又回到了 global 上 
global -> f5
true "f5" "a" 3 3
true "f5" "b" 3 3
```

同样的代码在 Nodejs 上运行看看。

```js
false 'global' 'a' 1 undefined
false 'global' 'b' undefined undefined

global ->  -> f1
true 'f1' 'a' 2 2
true 'f1' 'b' undefined undefined

global ->  -> f1 -> f2
true 'f2' 'a' 3 3
true 'f2' 'b' undefined undefined

global ->  -> f1 -> f2 -> f3
true 'f3' 'a' 3 3
true 'f3' 'b' undefined undefined

global ->  -> f1 -> f2 -> f3 -> f4
false 'f4' 'a' 3 undefined
false 'f4' 'b' undefined undefined

global -> f5
false 'f5' 'a' 3 undefined
false 'f5' 'b' undefined undefined
```

其中 `b` 一直都是 undefined，而通过 `this` 调用 `a` 也基本都有问题，在调用链上面有个空白。

> 因为 node.js 为了实现 commonjs 的模块，会把代码包裹在一个函数里，所以你的第一段代码里的 var a 声明并不是一个全局变量，而是一个模块（函数）内的局部变量。[^nodejs]

那个空白是个包装了当前文件的匿名函数，而后来 `this` 和 `global` 相等是因为函数调用的默认的 `this` 指向全局作用域。

上面的输出其实和源代码不一致，因为除了 `var` 改变了对象属性描述符的 可配置属性 之外没什么奇怪的。

![](https://i.loli.net/2019/01/17/5c3ffe0243c73.png)

实际上在 f5 被调用前，会打印一个 undefined， 这是当前栈结束时由最有一行返回的，如果写上 return 1，就会是 1 了，参见函数调用的相关资料，而 ES6 的箭头函数会把最后一行的返回值返回回来。

[^nodejs]: https://www.zhihu.com/question/39596666