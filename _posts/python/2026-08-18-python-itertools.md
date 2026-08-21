---
title: Python itertools：一个永远不会停的 for 循环，和装它的瑞士军刀
tags: [python]
categories: [Python]
last_modified_at: 2026-08-18 23:49:09 +0800
---

先看两行代码：

```python
from itertools import count
for n in count(1):
    print(n)
```

没有 `range`、没有终止条件、没有 `break`。你猜会发生什么？跑一下就知道——它从 1 开始，2、3、4、5……一直往下打，**永远不会停**，直到你手动按 Ctrl+C 或者电脑断电。

一个标准库函数，居然敢返回一个"无穷大的序列"。要是它返回的是列表，`list(count(1))` 会瞬间吃光你所有内存；可它就这么大剌剌地存在于每个 Python 安装里，安全得很。这中间藏着整个 itertools 的设计秘密。

更怪的是，这个模块里有二十多个怪名字的函数——`chain`、`islice`、`tee`、`starmap`、`zip_longest`——看起来像一堆零散的小工具，网上教程也常把它们列成一张"用法速查表"就完事。但用久了你会发现，它们其实只是**三种基本动作的不同组合**，一旦看穿这一点，整个模块就从"二十几个要背的函数"变成了"一块可以反复拼装的乐高"。

要看穿它，你得先抓住一个被所有函数共同遵守的铁律——

> **itertools 里的函数，进去的是迭代器，出来的还是迭代器。它们不存数据，只在你 next 的时候当场算一个给你。**

<!--more-->

## L1：先会用——三组最常用的工具

你在第一层，目标是：拿到一个"我想对序列做点什么"的需求，知道该从哪几个函数里挑。

itertools 里的函数大致分三类：**把多个迭代器拼成一个**、**从一个迭代器里切一段或分块**、**把多个迭代器重新组合**。先看最常用的那几个。

### 第一组：拼接与扁平化——`chain`

你有两个（或多个）可迭代对象，想当成一个连续的序列来遍历：

```python
from itertools import chain

a = [1, 2, 3]
b = "abc"
print(list(chain(a, b)))           # [1, 2, 3, 'a', 'b', 'c']

# 更常用：一个"嵌套"的列表，扁平化一层
groups = [[1, 2], [3, 4], [5]]
print(list(chain.from_iterable(groups)))   # [1, 2, 3, 4, 5]
```

和 `a + b` 的区别在于：`chain` 不复制数据，也不要求两边是同一种容器——列表、字符串、生成器、文件对象，它都能一个接一个地往外吐。`chain.from_iterable` 是当你手上是"一堆可迭代对象装在一个大迭代器里"时用的（比如你没法把它们一个个写成参数），它是扁平化一层嵌套的标准写法。

### 第二组：切片与分块——`islice`

生成器和迭代器不能用 `[1:5]` 切片（`iterable[1:5]` 直接 `TypeError`），因为切片需要知道长度、还要能随机访问，而迭代器只会"往后 next"。`islice` 就是给迭代器用的切片：

```python
from itertools import islice

gen = (x * x for x in range(100))
print(list(islice(gen, 3)))        # [0, 1, 4]        —— 取前 3 个
print(list(islice(range(100), 2, 8, 2)))  # [2, 4, 6] —— 起、止、步长
```

它做的事就是老老实实地 next 这么多次：要前 3 个就 next 3 次，要从第 2 个开始就先 next 掉 2 个再开始给你。没有魔法。

> Python 3.12 还加了一个 `batched(iterable, n)`，把迭代器按每 `n` 个切成一组：`list(batched('ABCDEFG', 3))` → `[('A','B','C'), ('D','E','F'), ('G',)]`。在老版本上，官方文档的 "itertools recipes" 里有个用 `zip` 实现的 `grouper`，等下你会看到它。

### 第三组：组合——`product`、`permutations`、`combinations`

这一组处理"多个集合之间的配对"：

```python
from itertools import product, permutations, combinations

# 笛卡尔积：所有可能的组合
print(list(product("AB", [1, 2])))
# [('A',1), ('A',2), ('B',1), ('B',2)]

# 排列：考虑顺序，不重复选同一个
print(list(permutations("ABC", 2)))
# [('A','B'), ('A','C'), ('B','A'), ('B','C'), ('C','A'), ('C','B')]

# 组合：不考虑顺序，不重复选
print(list(combinations("ABC", 2)))
# [('A','B'), ('A','C'), ('B','C')]
```

手写嵌套循环能做，但多层 `for` 一嵌套就再也读不下去；这三个函数把"枚举所有组合"这件事变成一行调用。`product` 尤其常被用来替代多层嵌套循环——比如你想遍历"大小 × 颜色 × 款式"的所有 SKU，`for size, color, style in product(sizes, colors, styles)` 比三层缩进干净得多。

### 还有三个值得认识的"无限迭代器"

回到开头的 `count`，它和 `cycle`、`repeat` 构成 itertools 里最魔幻的一组——**三个会无限产出东西的迭代器**：

```python
from itertools import count, cycle, repeat

list(islice(count(10, 2), 5))      # [10, 12, 14, 16, 18] —— 从 10 开始，步长 2
list(islice(cycle("AB"), 5))       # ['A', 'B', 'A', 'B', 'A'] —— 无限循环
list(repeat("x", 3))               # ['x', 'x', 'x'] —— repeat 给了第二个参数就有限
```

它们单独看像玩具，但和别的工具一组合就威力巨大——比如给数据加序号 `zip(count(1), rows)`，比 `enumerate(rows, 1)` 更底层但等价；比如把一个固定值拼进每条记录 `zip(repeat("INFO"), logs)`。

### 先预测，再看答案

```python
from itertools import chain
result = chain([1, 2], (x*x for x in range(3)))
print(list(result))
print(list(result))   # 第二次
```

<details><summary>点我看答案</summary>

第一次输出 `[1, 2, 0, 1, 4]`，第二次输出 `[]`（空列表）。

`chain` 返回的是个**迭代器**，迭代过一次就耗尽了——和生成器一模一样。这是整个 itertools 最容易踩的坑：它给你的不是列表，是一个"一次性的流"。要多次使用，就重新构造一次，或者自己用 `list(...)` 落成数据。

</details>

到这里你已经会用最常用的那几个工具了。但"会查文档"和"理解这个模块"是两回事——为什么所有函数都返回迭代器而不是列表？为什么开头那个 `count` 可以是无限的？为什么它们看起来这么"碎"，一个功能要拆成一个函数？答案要往下挖一层。

## L2：为什么它们全是迭代器

把 L1 那些函数列在一起对比，你会发现一个一致的模式：输入一个或几个可迭代对象，输出一个迭代器；中间不保存多余的数据。这不是巧合，是整个模块刻意的设计。

回顾[迭代器那篇](/2026/08/17/python-iterator.html)讲过的：迭代器是**惰性**的——你不 next，它就不算；它**不存数据**，只记住"我算到哪了"。itertools 里的每个函数，本质上都是给已有的迭代器"套了一层壳"，和装饰器给函数套壳异曲同工：

- `chain(a, b)` 记住"现在在 a 里，等 a 耗尽再切到 b"；
- `islice(it, 3)` 记住"还需要吐 3 个，多一个都不要"；
- `filter(pred, it)` 每 next 一次就从底下 `it` 里 next 到一个通过 `pred` 的为止；
- `product(a, b)` 内部记住两个游标走到哪了，按一下走一格。

它们用的都是**O(1) 额外空间**（相对输入长度而言），因为它们不复制数据，只记位置。这就是为什么 `count()` 可以是无限的——它根本不需要把"所有整数"存起来，只要记住"当前数到几了"即可。

### 你可以自己实现一个

看穿这个模式，你自己就能写出一个简化版的 `chain`，从而彻底理解它：

```python
def my_chain(*iterables):
    for it in iterables:        # 遍历传进来的每个可迭代对象
        for x in it:            # 逐个吐出元素
            yield x             # 自己是个生成器，所以返回的是迭代器

print(list(my_chain([1, 2], "ab")))   # [1, 2, 'a', 'b']
```

没有任何黑魔法：用 `yield` 做个生成器，把多个来源的元素顺着一根管子往外吐。itertools 里的绝大部分函数都能这样用几行生成器模拟出来——只是官方版本是 C 写的，更快、更省内存、签名更讲究。

> 这也是为什么 itertools 和[生成器](/2026/08/17/python-generator.html)、[高阶函数](/2026/08/18/python-lambda-hof.html)总被放在一起讲：它们的精神完全一致——**把"怎么遍历数据"和"数据本身"解耦，用一个个小的、可组合的对象描述数据流。**

### 组合起来才是真正的威力

因为每个工具的输出还是迭代器，你可以把它们像水管一样接起来：

```python
from itertools import chain, islice, filterfalse

# 多个数据源 → 扁平化 → 只要偶数 → 取前 5 个
data = [[1, 2, 3], [4, 5, 6], [7, 8, 9, 10]]
pipeline = islice(
    filterfalse(lambda x: x % 2, chain.from_iterable(data)),
    5
)
print(list(pipeline))   # [2, 4, 6, 8, 10]
```

- `chain.from_iterable(data)` 把嵌套列表摊成一条流：1,2,3,4,5,6,7,8,9,10；
- `filterfalse(...)` 在它外面套一层，只让偶数通过；
- `islice(..., 5)` 再套一层，攒够 5 个就停。

**注意**：整个过程中从头到尾没有任何中间列表被建立，所有计算都是在最后那个 `list(...)` 触发遍历的瞬间发生的。每来一个元素，就像在流水线上穿过三道工序，不合格的当场丢弃，攒够了就拉闸。

这就是为什么返回迭代器如此关键：如果 `chain` 先把所有数据物化成列表返回，再交给 `filterfalse` 又物化一次，三层管子就要三份内存，而且永远没法处理无限输入——可你给 `count(1)` 接上 `islice(..., 10)` 是完全合法的，它会老老实实给你 1 到 10。

## L3：本质——它是一个"迭代器代数"

去掉所有函数名和文档，itertools 到底是什么？

> **它是一组最小的、可以互相拼接的"迭代器积木"：每个函数只做一件简单的事，把若干条数据流变成另一条数据流，而真正的复杂逻辑由你把它们接出来。**

这有个专门的说法，叫 **iterator algebra（迭代器代数）**——"代数"这个词听着吓人，意思其实很朴素：就像代数里几个基础运算符（加、减、乘）能拼出任意复杂的表达式，itertools 给的是一组对"数据流"的基础运算符，你把它们组合起来，就能描述几乎任何遍历模式，而不必每次都回去手写带状态变量的 `for` 循环。

打个比方：普通的 `for` 循环像**一把菜刀**，切菜剁肉都行，但每次都要你自己盯着火候、自己掌握节奏；itertools 像一套**专用刀具**——切丝的、切片的、雕花的，每把只干一件事，但你把它们排成流水线，蔬菜从一头进去、成品从另一头出来，中间不需要你反复插手。

### 为什么是这个设计，而不是"一个万能函数"

一个显然的替代方案是：提供一个参数巨多的超级函数，能切片、能过滤、能拼接，一把梭。Python 为什么偏要拆成二十几个小函数？

- **组合优于配置**：一个 `super_iter(data, slice=..., filter=..., flat=True)` 看起来方便，但它的参数组合是**乘法级**爆炸的（切片且过滤且扁平化是一种，切片且不扁平又是一种……），每加一种功能，参数组合翻倍。拆成小函数后，组合数量交给用户用"管道"去搭，标准库本身永远只有 N 个函数。
- **可测试、可重用**：`islice` 只做切片这一件事，它的正确性可以被独立验证；你把它用在任何地方都不用担心"它会不会顺便改了我的数据"。大函数的逻辑是缠在一起的，改一处可能坏一片。
- **内存友好是被架构逼出来的**：一旦约定"进去迭代器、出来迭代器"，每个函数都自然是 O(1) 空间、自然支持无限输入、自然可以串成长管线。这不是性能优化，是设计选择带来的必然红利。
- **和 Unix 哲学同构**：一个工具做好一件事，用管道连接。`chain | filterfalse | islice` 在精神上就是 `cat | grep | head`。

被否决的"万能函数"方案会怎样？想象一个函数既想切片又想过滤还想扁平化，它内部必须同时维护所有这些状态、必须能处理参数缺失的每种组合、出错时你完全分不清是哪一步的问题。小而正交的工具拼出来的流水线，每一段都简单到显然没有 bug。

### 三个最值得记住的"配方"

官方文档的 [itertools recipes](https://docs.python.org/3/library/itertools.html#itertools-recipes) 一节给了几十个用基础函数拼出来的常用小工具，它们没有被直接收进模块，但你可以复制到自己的代码里。挑三个最常用的：

```python
from itertools import islice, zip_longest, chain

# 1. 把迭代器按 n 个一组切块（老版本替代 3.12 的 batched）
def grouper(iterable, n, fillvalue=None):
    args = [iter(iterable)] * n
    return zip_longest(*args, fillvalue=fillvalue)
# 关键技巧：[iter(...)] * n 造的是 n 个指向"同一个"迭代器的引用，
# 所以 zip 每次 next 它们，其实是在轮流从同一个流里取元素。

# 2. 相邻两两配对（老版本替代 3.10 的 pairwise）
from itertools import tee
def pairwise(iterable):
    a, b = tee(iterable)   # 复制成两个独立分支
    next(b, None)          # b 先往后挪一步
    return zip(a, b)       # a[i] 和 b[i]（也就是 a[i+1]）配对

# 3. 一次取 n 个项（跳过 islice 每次写 list(... ) 的麻烦）
def take(n, iterable):
    return list(islice(iterable, n))
```

`grouper` 里那行 `[iter(iterable)] * n` 是 itertools 圈子里有名的"黑魔法"——三个一模一样的迭代器引用，被 `zip_longest` 同时 next，每轮正好取走 n 个元素。初看像 bug，其实是对"迭代器是有状态的、共享同一个游标"这一事实的巧妙利用。能看懂这一行，说明 L2 你真的掌握了。

### 一段八卦：这个模块的作者和它的"代数"野心

itertools 主要由 Raymond Hettinger（Python 社区最能写的核心开发者之一，也是 `dict` 排序、`enumerate`、`collections` 一大堆东西的作者）在 2003 年前后贡献，灵感直接来自 Haskell 等纯函数式语言里的 list 处理库。最早版本里甚至还有 `ifilter`、`imap`、`izip` 这些带 `i` 前缀的名字——`i` 就是 iterator。Python 3 把内置的 `filter`/`map`/`zip` 改成直接返回迭代器之后，那些带 `i` 的版本就没必要存在了，被合并进内置函数。

但有个函数的命运很能说明"小而专注"的原则被执行得多彻底：`accumulate`。它在 2012 年（Python 3.2）才加入，做的是"前缀和"——`accumulate([1,2,3,4])` 给出 `1, 3, 6, 10`。有人质疑："这不是一行 `reduce` 就能写吗？"答案是：`reduce` 只给最终结果，而 `accumulate` 给每一步中间结果，而且是流式的、不毁数据的。在 itertools 的世界里，**"给中间结果"和"给最终结果"是两种不同的基础操作，值得分成两个函数。**

## 动手环节

### 实验 1：亲手验证"O(1) 空间 + 可处理无限流"

```python
from itertools import count, takewhile

# takewhile 会一直取，直到谓词第一次为 False 就停
result = takewhile(lambda x: x < 1000, count(1))
print(list(result))
```

跑一下，它会瞬间打出 1 到 999。现在试着把 `count(1)` 想成一个能产生"所有正整数"的东西——它没有尽头，但 `takewhile` 一旦看到 `>= 1000` 就喊停，整条管线立刻收尾。**整个过程中内存里只有当前那个数字，从来没有过一个长度为 999 的列表**（只有最后 `list(...)` 才把结果收进列表）。

再做一个对比：如果 `count` 真的返回一个列表会怎样？在脑中跑一下 `list(count(1))`——它会在产生任何输出之前就试图建立一个无穷大的列表，然后你的机器开始疯狂 swap，最后 OOM。这就是为什么迭代器不是性能优化，而是**能处理无限数据的唯一办法**。

### 实验 2：先预测，再运行

下面这段代码会输出什么？注意 `data` 是一个生成器表达式，而且被用了两次：

```python
from itertools import chain

data = (x for x in range(5))
pipeline = chain(data, data, data)
print(list(pipeline))
```

<details><summary>点我看答案</summary>

输出 `[0, 1, 2, 3, 4]`，而不是 `[0,1,2,3,4, 0,1,2,3,4, 0,1,2,3,4]`。

原因：`data` 是**一个**生成器，被传给 `chain` 三次，传的是同一个对象的引用。`chain` 第一次迭代它，一路 next 到耗尽；轮到第二、第三个"来源"时，它 next 的还是那个已经耗尽的迭代器，什么也拿不到，立刻结束。

这个坑完美对应 L2 那个"链式管线"心智模型：**迭代器是流，不是容器，同一条河水流过一次就没了。** 想重复三次，得写成 `chain(range(5), range(5), range(5))`，或者用 `itertools.tee` 复制出三个独立分支。

</details>

### 自测问题

1. 用 `product`、`permutations`、`combinations` 分别举例说明它们的差别，并说出：从 `{A,B,C}` 中选 2 个、**允许同一个元素出现两次**（比如 `('A','A')`）、不考虑顺序，应该用哪个？需要什么参数？
2. 写一个 `running_average(iterable)`，用 `accumulate` 和 `count`（或 `enumerate`）配合，输出每个前缀的平均值。提示：`accumulate` 可以给个函数作为第二参数。
3. 解释为什么下面两段代码行为不同：
   ```python
   list(islice(count(), 5))          # 写法 A
   list(islice(range(10**9), 5))     # 写法 B
   ```
   它们输出一样，但哪一个在构建时立刻吃掉大量内存？为什么？把这个区别讲清楚，就说明你理解了"迭代器是流"。

---

## 收尾地图

**itertools 速查表（按"你想做什么"组织）：**

| 你想… | 用什么 |
|---|---|
| 把多个迭代器首尾相接 | `chain(it1, it2, ...)`；嵌套的用 `chain.from_iterable` |
| 给迭代器做切片 | `islice(it, stop)` 或 `islice(it, start, stop, step)` |
| 按条件截取/丢弃 | `takewhile` / `dropwhile` |
| 留下/剔除满足条件的 | `filter`（内置）/ `filterfalse` |
| 把多个迭代器按位置打包 | `zip`（内置，按最短）；`zip_longest`（按最长） |
| 笛卡尔积 / 排列 / 组合 | `product` / `permutations` / `combinations`（去重）/ `combinations_with_replacement` |
| 相邻两个一组 | `pairwise`（3.10+） |
| 按 n 个一组 | `batched`（3.12+） |
| 前缀累积 | `accumulate`（默认求和，可给函数） |
| 复制一个迭代器成多个独立分支 | `tee` |
| 无限计数 / 无限循环 / 重复一个值 | `count` / `cycle` / `repeat` |
| 对一组元组解包后调用函数 | `starmap(fn, it)`（相当于 `fn(*x) for x in it`） |
| 把参数锁死再 map | 内置 `functools.partial` + `map` |

**下一站去哪：**

- 想真正理解 itertools 为什么这么设计，回头读[迭代器](/2026/08/17/python-iterator.html)和[生成器](/2026/08/17/python-generator.html)——itertools 里几乎每个函数都是某种生成器的 C 实现。
- `more-itertools` 是个第三方库，收录了官方 recipes 的绝大部分，加上几十个更高阶的工具（`chunked`、`flatten`、`windowed`、`unique` 等），看完标准库意犹未尽可以翻它的源码学。
- 想感受"组合式数据流"的真正威力，可以把 [pandas](https://pandas.pydata.org/) 的向量化操作或 [Apache Spark](https://spark.apache.org/) 的 RDD 看成是"对大数据集的 itertools"——它们的设计哲学同根同源：声明"做什么"，让运行时决定怎么遍历。
- 如果你喜欢"小函数拼大逻辑"的风格，再去翻 `operator` 模块（`itemgetter`、`attrgetter`、`add`、`methodcaller`），它和 itertools 是天作之合——比如 `sorted(rows, key=itemgetter('age'))`。

itertools 之所以被有经验的 Python 程序员当成"秘密武器"，不是因为它函数多，而是因为它给了你一套统一的语言来描述遍历：**拼接、切片、过滤、组合、累积**——而你要处理的任何数据流，拆解到最后，几乎都落在这几个动词里。下次你准备写一个带状态变量、嵌套三层、维护 `prev` 和 `current` 的 `for` 循环时，停一下，先问自己一句："这是不是 itertools 已经有的某个动词？"十次里有七次，答案是肯定的。
