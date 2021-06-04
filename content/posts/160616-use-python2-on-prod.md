---
title: "最近几年在百万级别日活用户后台服务中使用 Python 2.x 经验笔记"
date: 2016-06-16T00:00:00+08:00
draft: false
---

<!--more-->

如果没有特别说明，下面的 Python 运行环境指 Python 2.7 on Linux。

## 测试

**多写单元测试，多写单元测试，多写单元测试**，尽可能覆盖所有对外服务接口。

除了测试 RESTful 接口外，还可以使用 [Selenium](http://docs.seleniumhq.org) 模拟用户访问网页测试页面。


## 变量作用域(Scope)


    class RankItem(object):
        def __init__(self, name, idx):
            self.name = name
            self.idx = rank


    rank_list = [
        RankItem('a', 1),
        RankItem('b', 2),
        RankItem('c', 3),
    ]

    for rank_obj in rank_list:
        _ = [rank_obj.idx for rank_obj in rank_list]
        rank_obj.idx = 10
        break

14 行 列表表达式里面的 rank_obj 变量会覆盖外层 for 循环的变量，
15 行期望修改 rank_list[0].idx，实际会修改 rank_list[2].idx。

解决方法是, 列表表达式 里面使用简短、和外层不一样变量名避免覆盖污染

    _ = [obj.idx for obj in rank_list]

或 使用 生成器 代替 列表表达式

    _ = (rank_obj.idx for rank_obj in rank_list)


另见

Short Description of Python Scoping Rules
http://stackoverflow.com/questions/291978/short-description-of-python-scoping-rules

Python 2.7.12 documentation - 4. Execution model
https://docs.python.org/2.7/reference/executionmodel.html


## 性能瓶颈定位和优化(Profiling)

Python 中各个 Web 框架 profiler 功能大多数是基于 [profiler/cProfile](https://docs.python.org/2/library/profile.html) 模块实现。

Flask、Django 中可以使用 中间件(Middleware) 方式，通过非入侵式代码启用 profiler 功能。Flask ProfilerMiddleware 示例 见
https://gist.github.com/shuge/3b651ecc7bb39870fe5e832036e5b3d4

注意： `µs` is short for `microsecond`, `ms` is short for `millisecond`，1000 ms = 1 second(秒)。

在生产环境，通常有多台 Web 机器，只需要在一台机器上设置随机 dump profiler 文件即可，然后下载到本地分析

    $ python -m pstats samples/POST.api.v1.foo.1277ms.1450101273.prof
    % sort time calls
    % stats 15

输出

    Mon Dec 14 21:54:34 2015    samples/POST.api.v1.foo.1277ms.1450101273.prof

             6962 function calls (5876 primitive calls) in 1.277 seconds

       Ordered by: internal time, call count
       List reduced from 492 to 15 due to restriction <15>

       ncalls  tottime  percall  cumtime  percall filename:lineno(function)
            1    0.249    0.249    0.249    0.249 /usr/lib/python2.7/json/decoder.py:372(raw_decode)
         1278    0.209    0.000    0.209    0.000 {isinstance}
           15    0.187    0.012    0.189    0.013 ./champion/models.py:52(get_by_name_en)
        574/1    0.159    0.000    0.371    0.371 ./client_msg_parser.py:17(lower_keys)
            5    0.141    0.028    0.141    0.028 {_codecs.utf_8_decode}
            2    0.120    0.060    0.312    0.156 ./client_msg_parser.py:211(parse_team)
            2    0.096    0.048    0.096    0.048 /usr/local/lib/python2.7/dist-packages/kombu/utils/__init__.py:143(uuid4)
            1    0.052    0.052    0.054    0.054 /usr/local/lib/python2.7/dist-packages/flask_restful/reqparse.py:85(source)
            2    0.043    0.021    0.043    0.021 {method 'recv' of '_socket.socket' objects}
         1505    0.002    0.000    0.002    0.000 {method 'lower' of 'unicode' objects}
        504/9    0.002    0.000    0.371    0.041 ./client_msg_parser.py:21(<genexpr>)
          920    0.001    0.000    0.001    0.000 {method 'lower' of 'str' objects}
            1    0.001    0.001    1.275    1.275 /usr/local/lib/python2.7/dist-packages/flask_restful/__init__.py:569(dispatch_request)
          411    0.001    0.000    0.001    0.000 {method 'find' of 'unicode' objects}
           22    0.001    0.000    0.001    0.000 {method 'split' of 'str' objects}

从上向下阅读；
`tottime`(**tot**al **time**) 0.249 是指 decoder.py 文件 372 行中 `raw_decode` 函数调用（不包括里面子函数调用）耗时 0.249 秒(second)；
方括号`{}` 表示 Python 内置模块/内置函数，不能直接优化，只能通过优化业务逻辑减少调用优化；
上面的例子中，需要优化的地方是 `./champion/models.py` 文件 52 行，`get_by_name_en` 函数。

另外，如果有大量 `{method 'recv' of '_socket.socket' objects}` 慢日志，
要特别注意存储(数据库缓存没有命中或命中率特别低，数据库冗余 I/O 过多？）是否遇到瓶颈。


### dict 和 list 中查找

在一个大 list 中查找特定元素，比 dict 中通过 key 获取，慢接近 100 倍：

    import timeit

    import sample_item_list
    import sample_item_dict


    TARGET_ITEM_ID = 3086

    def test_list():
        for item in sample_item_list.items:
            if int(item['id']) == TARGET_ITEM_ID:
                return item

    def test_dict():
        return sample_item_dict.items.get(TARGET_ITEM_ID)


    if __name__ == '__main__':
        print('list length %s' % len(sample_item_list.items))
        eta_dict = timeit.timeit("test_dict()", setup="from __main__ import test_dict", number=10000)
        print('test_dict %f' % eta_dict)
        eta_list = timeit.timeit("test_list()", setup="from __main__ import test_list", number=10000)
        print('test_list %f' % eta_list)
        print('dict faster than list %s times' % int((eta_list / eta_dict)))

输出结果

    list length 225
    test_dict 0.004473
    test_list 0.421391
    dict faster than list 94 times


dict 的实现是 hash，时间复杂度是 O(1)，而 list 是 O(n)，list 越长越慢。

### 核心关键模块改用 POSIX C 实现

通常 POSIX C 改写模块要符合以下条件

 - 频繁调用
 - 重计算(非I/O)


以下是某业务核心解密纯 Python 实现 和 POSIX C 实现模块调用评测

    python2 ./benchmarks/benchmark_x_encryption.py
    test_x_decrypt_py 131.605892
    test_x_decrypt_c 8.931102

近14倍的性能提升。


## 影响性能的几个重要因素

 - 存储I/O —— Web 一般应用中九成瓶颈跟存储相关，使用队列+连接池+内存缓存可以缓解
 - 字符串高频创建/复制 —— 有可能导致内存碎片
 - 重计算型 —— 可以使用 C 实现模块或改用 pypy 解释器
