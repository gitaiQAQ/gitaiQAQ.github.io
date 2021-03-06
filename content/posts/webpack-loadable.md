---
layout:     post
title:      "按需加载和自定义 require"
date:       2019-08-07
author:     "Gitai"
tags:
  - 原理分析
series:
  - Webpack
---

Webpack 都写了那么多篇了，发现还有一些东西没写，所以这里补一篇。关于 Webpack 的按需加载，以及和他没啥关系的自定义 require 过程。

<!-- more -->

## 按需加载

先说按需加载，Webpack 分 chunk 可以有效减少 SPA 单次传输的网络压力，这种手段已经出现了很多年，AMD（异步模块加载）就是为其而生。

我们再来看看传统的 Ajax，大概就是下面这个样子

```js
// jQueryCallbackxxxxx({ data:{} });
$.ajax({
    type : "get",
    async: false,
    url : "http://localhost:2015/static/js/ambari_agent_disk_usage-md.js",
    dataType : "jsonp",
    jsonp: "callback",
    jQueryCallback: "jQueryCallbackxxxxx",
    success: function(json){
      console.warn(json);
    },
    error:function(err){
      console.error(err);
    }
});
```

而它一般接收的 JSONP 长得这个样子

```js
jQueryCallbackxxxxx({ data: null });
```

当请求完成时， JS 会被注入，执行 `jQueryCallbackxxxxx`，而这个方法被 jQuery 注册在全局，重定向到 Ajax 的 `success` 回调。


简单看一下 Webpack 生成的 chunk 

```js
(window["webpackJsonp"] = window["webpackJsonp"] || []).push([
    ["ambari_agent_disk_usage-md"], {

        /***/
        "./docs/ambari_agent_disk_usage.md":
            /***/
            (function (module, exports) {
                /***/
                module.export = {};
            })

    }
]);
//# sourceMappingURL=ambari_agent_disk_usage-md.js.map
```

看起来和 Ajax 的长相没有一点点一样的。。。把 `(window["webpackJsonp"] = window["webpackJsonp"] || []).push` 当成 `jQueryCallbackxxxxx` 倒是差不多了？

所以实现 Webpack chunk 的加载就是要实现一个对 `(window["webpackJsonp"] = window["webpackJsonp"] || []).push` 方法的包装，让其重定向结果到 `success`。

所以简单的覆盖默认的 `push` 实现就行了，这就能实现单 chunk 单资源 的 Loader

```js
let jsonpCallback = "jQuery19108266847";
(window['webpackJsonp'] = window['webpackJsonp'] || []).push = (webpackJsonp) => { // 预估只会出现单 chunk 的模块，所以可以直接弹出处理
    delete window['webpackJsonp'];
    let [ chunkIds, moreModules ] = webpackJsonp;
    let module = {};
    Object.values(moreModules).shift()(module)
    Function('json', `return ${jsonpCallback}(json)`)(module.exports) // 同上
};
```

但是严格来说这只是个加载特异 Ajax 脚本的加载器，设计上只是把 chunk 作为 JS 加载，并取出首个模块。不会去处理依赖关系，也没法解决资源路径问题（因为 chunk loader 就是为了其他服务调用目标服务的 chunk，所以必然存在其他机器上的依赖加载问题。）

所以有了下面这个，单 chunk 多资源 Loader，它就实现了对 webpack 传入的 require 进行支持，提供了 chunk 内部的依赖加载。并且可以修正外部资源的地址错误。但是其设计依赖于以下假设

* 资源均在当前 chunk
* 第一个模块是 HTML
* 其余模块是 `dataURL` 或者 `相对路径`

否则会出现严重 BUG

```js
var jsonpCallback = "console.log";
(window['webpackJsonp'] = window['webpackJsonp'] || []).push = (webpackJsonp) => { // 预估只会出现单 chunk 的模块，所以可以直接弹出处理
    delete window['webpackJsonp'];
    let [ chunkIds, moreModules ] = webpackJsonp;
    function fixUrl(path) {
        if (path.startsWith("data:")) return path;
        else return "http://www.xxx.xx/" + path;
    }

    function exec(module, ignoreFix) {
        let moduleWrapper = {};
        module(moduleWrapper, null, (key) => exec(moreModules[key])|| '包含当前加载器未实现的资源');
        return ignoreFix ? moduleWrapper.exports : fixUrl(moduleWrapper.exports);
    }
    
    let result = exec(Object.values(moreModules).shift(), true);

    // 调用 JQ 的回调
    Function('json', `return ${jsonpCallback}(json)`)(result) // 同上
};
```

## 自定义 require 过程

Webpack 的各项配置基本都可以自定义，最基本的传递一个函数进去，高级点的传递一个异步函数进去，在整个 webpack 的流程中，也有各种各样的钩子，但是之前没有叙述的大概还有不少，比如: `resolve` 配置

一般很少见人用它，毕竟默认的文件系统进行依赖加载已经很合理和常见了，覆盖了 90% 以上的模块加载方式应该毫不夸张。

但是为了造一个能把远程图片拉下来，然后统一管理的轮子，把[黑手](https://github.com/GitaiQAQ/remote-webpack-plugin)伸向了他。

```js
resolve: {
	plugins: [
		new RWP()
	]
}
```

这个 RWP 代码很简单

```js
const path = require('path');
const fs = require('fs');
const download = require('download');
const crypto = require("crypto");

module.exports = function (options) {
    let doApply = function (options, resolver) {
        resolver.getHook("before-resolve")
            .tapAsync("RemoteWebpackPlugin", (request, resolveContext, callback) => {
                // 判断请求类型
                if (!/^(?:\w+:)?\/\/(\S+)$/.test(request.request)) {
                    return callback();
                }

                let filename = crypto.createHash('md5').update(request.request).digest('hex') + path.extname(request.request);
                
                // 构造新请求
                let newRequest = path.resolve(path.join('.', 'node_modules', '.cache', 'remote-webpack-plugin', filename));

                // 检查缓存
                if (fs.existsSync(path)) {
                    request.request = newRequest;
                    return callback();
                } else {
                    // 下载并缓存
                    download(request.request, path.dirname(newRequest), {
                        filename
                    }).then(() => {
                        request.request = newRequest;
                        return callback();
                    });
                }
            })
    }
    return {
        apply: doApply.bind(this, options)
    };
}
```

以上。