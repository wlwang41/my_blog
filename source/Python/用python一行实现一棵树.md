---
title: "用python一行实现一棵树"
date: 2014-07-22 14:40
---

使用python自带的 `defaultdict`, 可以一行实现一棵树 `tree = lambda: defaultdict (tree)`:

    In [1]: from collections import defaultdict

    In [2]: tree = lambda: defaultdict (tree)

    In [3]: t = tree()

    In [4]: t['name']['crow'] = '__crow__'

    In [5]: t['name']['wlwang'] = '__wlwang__'

    In [6]: t
    Out[6]: defaultdict(<function <lambda> at 0x10bed7140>, {'name': defaultdict(<function <lambda> at 0x10bed7140>, {'crow': '__crow__', 'wlwang': '__wlwang__'})})

    In [7]: import json

    In [8]: print json.dumps(t)
    {"name": {"crow": "__crow__", "wlwang": "__wlwang__"}}

> 甚至可以不用赋值! 详情请戳[这里](https://gist.github.com/hrldcpr/2012250)
