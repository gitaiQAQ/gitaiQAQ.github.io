---
layout:     post
title:      "又一个前端打包工具 - parcel"
date:       2019-07-13
author:     "Gitai"
tags:
    - JavaScript
---

这是我在找 [Webpack 自动配置生成](https://createapp.dev/webpack) 时，发现的彩蛋？

既然别人把他们放在一起，估计挺好用的？你看 rollup, brunch 就没放上去

点开说明文档，此处划重点 “利用多核处理提供极快的性能，并且你不需要进行任何配置。”

这不就是我们在 Webpack 使劲折腾的问题，以及 brunch 的那套方案？甚至更少的配置。

看Demo 和测试还算 Ok；但是会不会和 brunch 一样，接口不完善，生态又辣鸡，于是就完全跑不起来。

而且第一眼看到这个名字，我还以为时 preact，大概加班太多，神智有点不清不楚。

<!-- more -->

## 初探

看上去提供了 3 个主要的命令

* parcel 本地服务器测试
* parcel watch 监听文件变化，自动编译
* parcel build 编译

很通俗的几个操作，没发现亮点，除了不要加 dev，默认为 dev server 之外

看文档描述，他和 webpack 不同大概是入口，webpack 秉承着一切都是 JS 的原则，啥东西都打包成 JS；虽然 webpack 可以通过 loader 加载奇奇怪怪的 entry，但是输出就比较麻烦，我想用 webpack 打包 markdown 生成静态站的伟大构想，只能通过自定义 Plugin 或者各种骚操作实现，因为设计上所有的 loader 最后都会把文件转化为 JS 模块；preact 的中间状态暂时不知道，毕竟这只是个初探。

#### 所以总结一下：   

告别几百行复杂的 webpack 配置，内置了大量 webpack 配起来十分蛋疼的东西（比如 HMR、dev server、各种 loader）    

所以可能将来的构建工具就 2 种，**一种叫 webpack**，**一种叫 webpack 最佳实践**。      
抛开配置复杂度的问题（可以内部封装），但是性能优化只能重新设计插件和使用机制，这块 webpack 是历史遗留原因；而新实现的这些最佳实践，也都没法复用 webpack 的生态，所以要是生态没起来，迟早得 GG

比如 Brunch，随便找个啥插件都没有，这生态也不知道用他的意义是啥？说的就是 Ambari 那个项目。

## 深入尝试一下

对于这种工具包，最直接的使用方式就是帮他拓展功能；通过拓展功能熟悉 API，就能把它作为一个黑箱来揣测内部实现机制和可拓展性之类的问题。

比如之前通过写[Webpack Loader 和 Plugin](https://gitai.me/2019/05/loader&plugin/) 和  [rollup-plugin-output](https://github.com/GitaiQAQ/rollup-plugin-output) 大致分析了一下，webpack 和 rollup 的实现机制和拓展机制。

#### 得出如下没什么卵用的结论：

rollup 弱化了 webpack 复杂的插件机制，因为插槽少，所以拓展性受限，但是相对性能提高，且配置减少。

这里我们可以类比其他前端娱乐圈的项目，比如 Atom 和 VSCode，有些设计的确出彩，可拓展性极强；但是是否具有必要性？是不是我们只是需要一个简单的核心，或者能替换核心的接口。

而不是提供一个啥都没有的空框架，然后利用框架增加新特性？特指 Webpack，或许将来会出现，webpack 做快速实现，然后能导出插件调用关系的高性能构建工具，也就是对 webpack 本身的编译和优化，让他作为工具使用的时候，脱离其可拓展性和动态特性。

对了，这里是不是应该尝试开发 parcel 插件来着。

找了几个别人的插件看看，这里我们借鉴 webpack 插件的开发原理，拓展包含 loader 和 plugin，loader 提供的是对新文件解析的支持，而 plugin 提供的是新的流程。至于可视化的流程，应该看看我[之前的文章](https://gitai.me/2019/06/rollup/)。一切构建都可以被 Gulp 包括，因为他只是个任务流程定义工具，这些构建工具最后都会被切分成一堆插件提供的任务，一个包含上下文的函数入口。

对于上面的叙述，主要为了说明为什么我们选择[新模板（markdown）](https://github.com/agentcooper/parcel-plugin-static-zip/blob/master/packages/plugin/index.js)和[静态文件压缩（parcel-plugin-static-zip）](https://github.com/agentcooper/parcel-plugin-static-zip/blob/master/packages/plugin/index.js)这2个模块来分析，pug 只是新 loader，应该会转化为 HTML 进入通常的流程，而 zip 这个应该会介入 parcel 流程，在合适的地方放钩子，进行压缩。

好了，对应的就是上面的地址，markdown 是一个[新的资源](https://parceljs.org/asset_types.html)，实现注册和解析就完成了任务。

而 zip 是个插件，他用到了 parcel 的[接口](https://parceljs.org/api.html)，来监听任务完成的事件。

剩下的就没必要深入了