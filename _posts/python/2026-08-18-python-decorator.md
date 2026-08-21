---
title: Python 装饰器：为什么加了一行 @log，函数就不认识自己了
tags: [python]
categories: [Python]
last_modified_at: 2026-08-18 23:43:00 +0800
---

先看一段十行不到的代码：

```python
def log(fn):
    def wrapper(*args, **kwargs):
        print(f"调用 {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper

@log
def add(a, b):
    """两数相加"""
    return a + b

print(add.__name__)   # 你以为是 add
print(add.__doc__)    # 你以为是"两数相加"
```

实际跑一下：

```
wrapper
None
```

函数把自己的名字和文档全丢了。你只是在它头顶贴了一个 `@log`，什么都没改，它就"不认识自己"了。更怪的是，网上搜来的解法是在 `wrapper` 上面再加一行 `@functools.wraps(fn)`——**用一个装饰器去修另一个装饰器造成的问题**，像个绕口令。

这是装饰器给几乎所有人的第一记下马威。但只要你看穿一件事，这些怪现象会一次性全部消失：

> `@log` 根本不是什么"给函数打标签"，它是一句**赋值**的语法糖。它在函数定义完的瞬间，把这个函数换成了另一个东西。

<!--more-->

## L1：先写出一个能用的装饰器

你在第一层，目标只有一个：能给一个已有的函数"加一层行为"，而不改它的源码。

装饰器的定义朴素到让人失望：

> **一个吃进函数、吐出函数的函数。**

```python
def log(fn):
    def wrapper(*args, **kwargs):
        print(f"调用 {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper
```

`log` 接收一个函数 `fn`，返回一个新函数 `wrapper`。`wrapper` 干的事是：先打印一行，再把收到的所有参数原封不动转交给 `fn`，最后把 `fn` 的返回值透传出去。

而 `@log` 只是下面这句的缩写：

```python
add = log(add)
```

就这么简单。Python 执行到 `def add(...)` 时造出函数对象，紧接着做一次 `add = log(add)`——把 `log` 返回的那个 `wrapper` 重新绑回 `add` 这个名字。从此以后，你调用的 `add` 早已不是你写的那个 `add`，而是 `wrapper`。这就解释了开头为什么 `add.__name__` 是 `wrapper`：**因为它现在就是 wrapper**。

### 为什么非得套一层 wrapper

新手常写的第一个错误版本是这样的：

```python
def log(fn):
    print(f"调用 {fn.__name__}")   # 直接写在 log 里
    return fn
```

跑起来你会发现，那行 print 只在**模块被 import、函数被定义时**打印了一次，之后每次调用 `add()` 什么都不输出。

原因在于时机：`log(fn)` 这个函数体只在**定义那一刻**执行一次。要让"打印"发生在**将来每次调用**时，你必须把它放进一个会被反复调用的壳里——这个壳就是 `wrapper`。`wrapper` 通过闭包抓住 `fn`，等以后谁来调用，它才执行那段逻辑再转交。

> 所以 `*args, **kwargs` 和 `return fn(...)` 不是什么仪式感：前者保证不管原函数有什么参数都能转发，后者保证原函数的返回值不会被你吞掉。漏了 `return`，被装饰函数的返回值就会神秘地变成 `None`，这是另一个经典的坑。

### 用 @wraps 把名字要回来

知道了病因，药方就好理解了。我们想让对外的名字还是 `add`、文档还是"两数相加"，但又不想手动一个个拷贝属性。`functools.wraps` 就是干这个的：

```python
from functools import wraps

def log(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        print(f"调用 {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper
```

加上它，`add.__name__` 就又变回 `add` 了。它本质上就是把原函数的 `__name__`、`__doc__`、`__module__`、`__qualname__` 等属性抄到 `wrapper` 上，还顺手设了一个 `__wrapped__` 指向原始函数（很多调试工具靠它"剥开"装饰器）。

先别急着追问 `@wraps(fn)` 为什么长成这副怪样子——它本身就是下一层要讲的东西，你先记住"它用来修元数据"即可。

### 先预测，再看答案

```python
def deco(fn):
    print("A")
    return fn

@deco
def f():
    print("B")

print("C")
f()
```

<details><summary>点我看答案</summary>

输出顺序是 `A C B`，不是 `C A B`。

`A` 在 `def f` 那一行就打印了——装饰器在**定义时**执行，不是第一次调用时。这是理解装饰器最重要的时间点，后面所有"带参数装饰器""堆叠顺序"的困惑，根源都在这。

</details>

到这里你已经能写出能用、也不会丢名字的装饰器了。但用着用着，你会撞上三个新问题：`@wraps(fn)` 后面凭什么还带个括号？我想让装饰器自己接收参数（比如 `@retry(3)`）该怎么写？多个 `@` 叠在一起谁先执行？

这三个问题是同一个答案：**`@` 那一刻到底发生了什么。**

## L2：把 @ 那一刻发生的事拆开看

把语法糖彻底展开，规则只有两行：

```python
@deco
def f(): ...

# 完全等价于：
def f(): ...
f = deco(f)
```

而带括号的版本：

```python
@deco(args)
def f(): ...

# 完全等价于：
def f(): ...
f = deco(args)(f)
```

注意那个 `deco(args)(f)`：Python 会**先**调用 `deco(args)` 拿到一个返回值，**再**把这个返回值当装饰器、用 `f` 去调用它。看穿这一点，所谓"带参数的装饰器"就一点都不神秘了——

> **带参数的装饰器根本不是装饰器，它是一个"装饰器工厂"：一个返回装饰器的函数。**

### 三层嵌套，每层只管一件事

```python
def retry(times):                      # 第 1 层：接收装饰器参数
    def decorator(fn):                 # 第 2 层：这才是真正的装饰器，接收被装饰函数
        def wrapper(*args, **kwargs):  # 第 3 层：每次调用时执行的壳
            for _ in range(times):
                try:
                    return fn(*args, **kwargs)
                except Exception:
                    pass
        return wrapper
    return decorator

@retry(3)
def call_api(): ...
```

展开后是 `call_api = retry(3)(call_api)`：

1. `retry(3)` 运行，`times = 3` 被闭包抓住，返回 `decorator`；
2. `decorator(call_api)` 运行，`fn` 被抓住，返回 `wrapper`；
3. 以后每次 `call_api()` 调的是 `wrapper`，它能看到 `times` 和 `fn` 两个自由变量。

三层各有各的职责，谁也替代不了谁。新手常犯的错是想在两层里搞定一切，结果发现要么拿不到 `times`，要么拿不到 `fn`——因为它们活在不同的作用域、活在不同的执行时刻。

> 现在回头看 `@wraps(fn)`：它就是一个**带参数的装饰器**。`wraps(fn)` 先返回一个装饰器，这个装饰器再被应用到 `wrapper` 上，把 `fn` 的属性抄过来。所以它写在 `wrapper` 头顶，而不是 `log` 头顶——因为它要装饰的是 `wrapper`。绕口令解开了。

### 多个装饰器叠在一起：洋葱模型

```python
@dec1
@dec2
def f(): ...
```

展开规则依然机械：从下往上包。

```python
f = dec1(dec2(f))
```

也就是 `dec2` 先贴身包住原函数，`dec1` 再包在最外面。但**调用时的顺序正好反过来**：最外面的 `dec1` 先开跑，像剥洋葱一样一层层往里钻，原函数在最中心。

### 先预测，再看答案

```python
def dec1(fn):
    def wrapper(*a, **k):
        print("1 in");  r = fn(*a, **k);  print("1 out");  return r
    return wrapper

def dec2(fn):
    def wrapper(*a, **k):
        print("2 in");  r = fn(*a, **k);  print("2 out");  return r
    return wrapper

@dec1
@dec2
def f():
    print("  f")

f()
```

<details><summary>点我看答案</summary>

```
1 in
2 in
  f
2 out
1 out
```

应用顺序从下到上（`dec2` 贴身），执行顺序从上到下（`dec1` 先进）。记一个口诀：**定义时向上爬，调用时向下钻**。中间一旦某个 wrapper 忘了 `return fn(...)`，洋葱芯子的返回值就在那一层断了。

</details>

### 一个更隐蔽的点：装饰器在 import 时就跑了

因为装饰发生在 `def` 语句执行时，而模块顶层的 `def` 在 **import 那一刻**就会执行，所以你的装饰器函数体（不是 wrapper，是 `deco` 本身）是在 import 时跑的。这意味着：

- 不要在装饰器函数体里做依赖运行时配置、数据库连接、读用户输入这类重活——那时候程序可能还没准备好；
- 想做的初始化，要么提前到模块加载前就准备好，要么放进 `wrapper` 里延迟到第一次调用。

把它和开头那个 `A C B` 的预测题联系起来，你就彻底建立了"装饰器函数体=定义时、wrapper=调用时"这个时间感。

## L3：去掉语法糖，装饰器到底是什么

到这里"它怎么工作"你全懂了。还差一个问题：**Python 为什么非要发明一个 `@` 语法？我直接在函数下面写 `f = log(f)` 不也一样吗？**

一样，完全一样。`@` 没有带来任何做不到的新能力。它带来的是一个**位置**：

> **装饰器是挂在 `def`（和 `class`）上的"出厂拦截器"——一个函数或类刚被造出来，Python 就顺手把它交给你指定的加工厂，你可以原样放行、包一层、甚至换成一个完全不相干的东西，再贴上原来的名字。**

去掉所有术语，装饰器的本质是一句话：

> **在函数诞生的那一刻，把它悄悄换掉。**

这个"诞生时拦截"的位置，才是它价值的全部。它让你能把"一大批函数都该有的同一种行为"——记日志、算耗时、做缓存、失败重试、权限校验、注册路由——从函数体里抽出来，集中写一次，然后用一个 `@` 贴到任何函数头上。函数自己只关心核心逻辑，调用方也完全无感知，却自动获得了新行为。

这就是面向对象里常说的**开放封闭原则**：对扩展开放，对修改封闭。你给一百个函数加日志，一行函数体都不用改。

### 为什么不用别的办法

你当然可以用别的机制实现类似效果，但各有各的别扭：

| 替代方案 | 问题 |
|---|---|
| 定义后手动 `f = log(f)` | 能用，但和定义分离了：要滚到函数底部才知道它被改造过，容易漏，多个改造时顺序还得自己数 |
| 继承 / 子类化 | 那是类层面永久的"is-a"关系，重，而且根本加不到一个普通函数头上 |
| 猴子补丁（事后替换模块里的名字） | 分散、隐式、顺序敏感，出了 bug 你都找不到是谁在什么时候改的这个函数 |

`@` 把"我被谁装饰"钉在函数头顶第一行，**显式、就近、在定义点一次性完成**，这正是手动赋值做不到的工程体验。

### 代价：没有白吃的午餐

把逻辑从函数体里抽出来不是免费的：

1. **调用栈变深、报错变丑**：包了五层装饰器的函数，traceback 里是一长串 `wrapper`，定位问题要在洋葱里来回翻。
2. **签名被遮蔽**：`help(add)` 看到的是 `wrapper(*args, **kwargs)`，类型检查器也会犯迷糊——`@wraps` 能补元数据，但复杂签名还得靠 `inspect` 或第三方库。
3. **import 时就执行**：前面说过，装饰器顶层代码在 import 时跑一次，放错东西会有意外的副作用。
4. **换上去的对象得"像"原来的**：你可以把函数换成任意对象，但如果调用方期望它是函数、有某些属性，你就得保证返回的东西伪装得足够像——这就是 `@wraps` 存在的根本原因。

### 一段八卦：@ 这个符号是吵出来的

装饰器在 **PEP 318**（Python 2.4，2005 年）才加入。在那之前，`staticmethod`、`classmethod`、`property` 已经存在，但用起来非常难看：

```python
class Foo:
    def bar(cls):
        ...
    bar = classmethod(bar)   # 定义完了还得回来赋值一次
```

为了让这种"定义后转换"有个体面的写法，PEP 318 应运而生。但**用哪个符号**引发了 Python 社区史上有名的大争论：有人提 `|`，有人提 `...`，有人主张干脆加个 `decorate` 关键字。Guido 最终拍板用 `@`，理由很实在：它在当时的 Python 里几乎是个闲置符号。

故事到这没完。十年后，数值计算社区实在想要一个矩阵乘法运算符，PEP 465（Python 3.5，2015 年）又把 `@` 征用去当 `__matmul__` 了。于是今天的 `@` 身兼两职：**出现在 `def` 上面是装饰器，出现在表达式中间是矩阵乘法**，全靠位置区分。一个符号打两份工，也算是 Python 语法里一段奇闻。

> 顺带一提：能装饰**类**还要更晚，是 PEP 3129（Python 2.6 / 3.0）才补上的。所以你今天看到的 `@dataclass`、`@total_ordering` 这类类装饰器，在 2005 年是写不出来的。

## 动手环节

### 实验 1：写一个记忆化装饰器，看闭包怎么当代词用

```python
def memoize(fn):
    cache = {}
    def wrapper(*args):
        if args not in cache:
            print(f"缓存未命中，计算 {fn.__name__}{args}")
            cache[args] = fn(*args)
        return cache[args]
    return wrapper

@memoize
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(10))
```

跑一下，留意"缓存未命中"打印了多少次——每个 `n` 只算一次。这里的 `cache` 和 `fn` 都是被 `wrapper` 抓住的自由变量（活在 cell 里），装饰器没有发明任何新机制，它就是给[闭包](/2026/08/18/python-closure.html)的一种固定用法起了个名字、安了个位置。看懂上一篇闭包的，看这段应该像呼吸一样自然。

### 实验 2：故意踩一次"定义时 vs 调用时"

下面这个计时装饰器有 bug，计时结果会是个完全不对的数字（甚至接近 0）。先别运行，找出问题，再把它修好：

```python
import time
def timed(fn):
    start = time.time()          # ← 问题在这一行
    def wrapper(*args, **kwargs):
        result = fn(*args, **kwargs)
        print(f"{fn.__name__} 耗时 {time.time() - start:.4f}s")
        return result
    return wrapper
```

<details><summary>点我看答案</summary>

`start` 写在 `timed` 的函数体里，它在**装饰器被应用时（import/定义时）**就求值了，之后每次调用都拿"程序刚启动时"当起点，算出来当然是错的。

把它挪进 `wrapper`，让它在**每次调用时**重新计时：

```python
def timed(fn):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = fn(*args, **kwargs)
        print(f"{fn.__name__} 耗时 {time.time() - start:.4f}s")
        return result
    return wrapper
```

这就是 L1 强调的那个时机问题的真身：**装饰器函数体里的代码在定义时跑一次，wrapper 里的代码每次调用都跑。** 分不清这两者，装饰器会变出各种幽灵 bug。

</details>

### 自测问题

1. 写一个 `@repeat(n)` 装饰器，让被装饰的函数每次被调用时实际执行 `n` 次，返回**最后一次**的结果。（提示：它是带参数的装饰器，需要三层。）
2. 不运行，说出下面两种写法里，"真正的装饰器函数"收到的第一个参数分别是什么：
   ```python
   @deco        # 写法一
   def f(): ...

   @deco()      # 写法二
   def g(): ...
   ```
   （答案指向 L2 那句"带参数的装饰器是装饰器工厂"。）
3. 用类实现一个装饰器：写一个 `CountCalls`，让 `@CountCalls` 装饰过的函数每次被调用时打印"这是第 N 次调用"。需要实现哪个特殊方法？被装饰函数本身存在哪个实例属性上？

---

## 收尾地图

**装饰器速查表：**

| 你想… | 写法 |
|---|---|
| 给函数套一层行为 | `def deco(fn): def wrapper(*a, **k): ...; return wrapper` |
| 保住函数名和文档 | `from functools import wraps`，在 `wrapper` 头顶加 `@wraps(fn)` |
| 让装饰器接收参数 | 再加一层：`def deco(arg): def inner(fn): def wrapper(...): ...; return wrapper; return inner; return deco` |
| 给类加装饰 | 同样的形式，参数 `fn` 换成 `cls`，在里面改完再返回类 |
| 用类写装饰器 | 实现 `__init__(self, fn)` 存函数、`__call__(self, *a, **k)` 定义调用行为 |
| 临时绕开装饰器 | 调用 `被装饰函数.__wrapped__(...)`（`@wraps` 会设上这个指向原函数的属性） |

**下一站去哪：**

- 看懂了函数装饰器，去翻 `@dataclass`、`@functools.total_ordering`、`@enum.unique`——它们全是**类装饰器**，在类诞生时改类。
- 再往深走一步是**描述符协议**（`__get__` / `__set__`）：`@property`、`@classmethod`、`@staticmethod` 表面是装饰器，真正的魔法其实发生在描述符里。装饰器只是把它"装"到了函数头上。
- `contextlib.contextmanager` 展示了 `@` 的另一种想象力：把"进入/退出"这种成对的逻辑写成一个生成器，贴一下就变成上下文管理器。
- 嫌三层嵌套样板烦，可以看 `wrapt` 和 `decorator` 这两个库，它们帮你处理签名保留和带参数的胶水代码。
- 回头读[《Python 闭包：为什么三个函数都返回了 2》](/2026/08/18/python-closure.html)：你会发现 `wrapper` 就是最典型的闭包，`fn`、`cache`、`times` 全是活在 cell 里的自由变量。装饰器没有创造任何新机制——**它只是承认了"闭包最常见的一种用法值得有个名字"，并把这个名字安放到了函数诞生的地方。**

装饰器听起来花哨，拆穿了只有一句话：**它让你在函数定义完的瞬间，用另一个东西替换它。** 你之所以会被 `@retry(3)` 的三层、会被丢失的 `__name__`、会被洋葱般的堆叠顺序绕晕，都是因为忘了这个最朴素的事实。一旦你把 `@deco` 自动翻译成 `f = deco(f)`，装饰器就再也没有任何秘密。
