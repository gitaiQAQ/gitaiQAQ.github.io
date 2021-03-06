---

layout:     post
title:      "加一层是解耦合的最佳方法"
date:       2018-01-28
author:     "Gitai"
categories:
    - 杂项

---

最近，在一个初创公司，而且核心是内容的初创公司。WordPress 作为基础框架的使用还是非常常见的。

但是 WordPress 非常的臃肿，逐渐稳定之后，必然需要将技术栈迁移到其他合适的自研框架中去。

至于为什么写这篇，自然是因为没找到别人写过类似的分析。

各种软件设计中对于解耦合最简单高效的方法无碍乎 “加一层”，无论是单个应用还是大型分布式系统。

> 计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决
>
> Any problem  in computer science can be solved by anther layer of indirection.
>
> —— 出处存在争议

<!-- more -->

首先梳理广泛运用的实现方式，对于大型业务系统，必然会被拆分多个模块。

然后为了处理统一的服务，会引入 SSO（单点登录），和统一的日志管理模块。等这些模块健壮了，就慢慢出现了 gateway 这种非常好用的东西。

但是，估摸着实习这段时间是不可能给他整个技术栈重构了，尤其用了 WordPress 那个课程插件。坑太大，暂时不填，等时机成熟给他填了写简历也是美滋滋...

于是，就展开了以下方法的探讨：

1. 在其他框架中，对 WordPress 发起 RPC 鉴权
2. 直接读取数据库，解密 cookies

## RPC 鉴权

前置 IP 设置个白名单，或者加个 token

目标框架将 cookies 转发，WordPress 写个函数，返回校验结果。

## 直接操作数据库

先去 WordPress 里面把有关代码翻出来。

* [`COOKIEHASH`](https://github.com/WordPress/WordPress/blob/master/wp-includes/default-constants.php#L220)
* [`LOGGED_IN_COOKIE`](https://github.com/WordPress/WordPress/blob/master/wp-includes/default-constants.php#L261)
* [`wp_validate_auth_cookie`](https://github.com/WordPress/WordPress/blob/master/wp-includes/pluggable.php#L602) 
* [`wp_generate_auth_cookie`](https://github.com/WordPress/WordPress/blob/master/wp-includes/pluggable.php#L712)

简述一下默认的生成原理

先获取 `COOKIEHASH`，默认为 `siteurl`，储存在 WordPress 数据库的 `wordpress_option`。并通过 md5 摘要获取识别码。

`LOGGED_IN_COOKIE` 是 `cookies` 的 `key`，形如 `wordpress_logged_in_ac0818ef3df8d5d8ee0c025b0c25a7af`（`wordpress_logged_in_` + `COOKIEHASH`）。

从上面得到的 `LOGGED_IN_COOKIE` 取出值 `admin%7C1518332456%7CQ86dRzSrTFJnnyeleONzDZ1XUDKkLJcm0A8CsmJvfRh%7Cf253da0429b6011c3b51f222e0b59ca3eb7c8aeac712b7af101468fca3bab63b` 通过 URL 解码，得到 `admin|1518332456|Q86dRzSrTFJnnyeleONzDZ1XUDKkLJcm0A8CsmJvfRh|f253da0429b6011c3b51f222e0b59ca3eb7c8aeac712b7af101468fca3bab63b`。对应 `username`, `expiration`, `token`, `cookie_hmac`。

之后进入校验流程。

1. 是否过期
2. 用户是否存在
3. 检查 `cookie hash` 是否匹配

`pwd_frag` 是用户密码串的 8~12 位

在 `wp_config` 取出 `WORDPRESS_LOGGED_IN_KEY` + `WORDPRESS_LOGGED_IN_SALT`，这是加密必要的盐。

原文本：`username` + `pwd_frag` + `expiration` + `token`
密钥：`WORDPRESS_LOGGED_IN_KEY` + `WORDPRESS_LOGGED_IN_SALT`

通过 `hmac-md5` 加密获取 `hash`，和 `cookie_hmac` 对照，完成登录流程。

## 参考

[django 的实现](https://github.com/ScilCoop/django-wordpress-auth-lite)
