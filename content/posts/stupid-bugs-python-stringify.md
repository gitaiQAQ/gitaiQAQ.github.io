---
layout:     post
title:      "一个短路操作符失效的 BUG？"
date:       2019-01-28
author:     "Gitai"
tags:
    - Python
---

遇上一个百思不得其解的 BUG

```python
logger.debug("%s %s", test_case_id, test_case_info)
test_case_id = test_case_id or md5(test_case_info)
logger.debug("MD5: %s %s", test_case_id, md5(test_case_info))
```

```shell
09:18 None {'test_cases': [{'output': '420', 'input': '5 8'}]}
09:18 MD5: None b676c8555a8384fd346b062397524020
```

![惊呆](https://i.loli.net/2019/01/28/5c4ed5cdafb3f.png)

<!-- more -->

交代背景，这是通过

```shell
exec gunicorn --workers $n --threads $n --error-logfile /log/gunicorn.log --time 600 --bind 0.0.0.0:8080 server:app
```

`flask` Web 服务调用的一个方法

```python
class TestCaseInfo:
    def __init__(self, test_case_info=None, test_case_id=None):
        logger.debug("TestCaseInfo: %s %s", test_case_id, test_case_info)
        self.test_case_id = test_case_id or md5(test_case_info) # 直接上短路操作会失败，不知道为啥
        logger.debug("MD5: %s %s", test_case_id, md5(test_case_info))
```

通过下面这样调用的

```python
class JudgeClient(object):
    def __init__(self, run_config, exe_path, max_cpu_time, max_memory, test_case_id, test_case_info,
                 submission_dir, spj_version, spj_config, output=False):
        self._test_case_info = TestCaseInfo(test_case_info=test_case_info, test_case_id=test_case_id)
        
@app.route('/', defaults={'path': ''})
@app.route('/<path:path>', methods=["POST"])
def server(path):
    getattr(JudgeServer, path)(**request.json)
```

回到上面那个问题，然后把代码改成这样测试

```python
a = test_case_id
b = test_case_info
c = test_case_id or test_case_info
d = test_case_id or md5(test_case_info)
e = test_case_info or test_case_id
logger.debug("a:%s b:%s c:%s d:%s e:%s", a, b, c, d, e)
```

```shell
10:09 None {'test_cases': [{'output': '420', 'input': '5 8'}]}
10:09 a:None b:{'test_cases': [{'output': '420', 'input': '5 8'}]} c:None d:None e:{'test_cases': [{'output': '420', 'input': '5 8'}]}
```

![大吃一惊](https://i.loli.net/2019/01/28/5c4ed66ff195c.gif)

确信不是 `test_case_id`，`test_case_info` 变量状态问题，`md5` 这个函数也完全没问题，倒置表达式也是 OK 的，那么问题来了。。。短路操作符居然崩了？？？

然后又写了几个测试用例

```python
a = None or {"a": "a"}
b = None or test_case_info
c = test_case_id or {"a": "a"}
d = test_case_id or test_case_info
logger.debug("a:%s b:%s c:%s d:%s", a, b, c, d)
```

```shell
10:25 a:{'a': 'a'} b:{'test_cases': [{'input': '5 8', 'output': '420'}]} c:None d:None
```

那给前面加个赋值，`test_case_id = None` 看看

```shell
10:30 a:{'a': 'a'} b:{'test_cases': [{'input': '5 8', 'output': '420'}]} c:{'a': 'a'} d:{'test_cases': [{'input': '5 8', 'output': '420'}]}
```

![WTF?](https://wx1.sinaimg.cn/mw690/690c6f7cly1fzmh5ra5fuj20m80m8n3o.jpg)

那莫非是。。。默认参数的问题？于是单独写个 `.py` 文件。

```python
def a(id=None, info=None):
    print(id or info)

a(info={"a":""})
```

```shell
➜  python ./t.py
{'a': ''}
```

陷入深思。。。

```python
if test_case_id:
    self.test_case_id = md5(test_case_info)
else:
    self.test_case_id = test_case_id
```

暂时这么解决，为啥短路操作就不行？？？

~~Py 垃圾，除非谁给我解释一下这是啥问题。~~

我垃圾，从请求开始盯着这个变量，最后发现是有个调用的地方把他转化成了 `str` 。。。本年度最蠢的 BUG，真丢人。。。