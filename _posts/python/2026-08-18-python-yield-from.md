---
title: Python yield from：它不是 for-yield 的缩写
tags: [python]
categories: [Python]
last_modified_at: 2026-08-18 22:56:09 +0800
---

先看一段几乎每个 Python 程序员都写过的代码。假设你有一个生成器，想把它产出的值"转手"再吐出去：

```python
def sub():
    yield 1
    yield 2

def wrapper():
    for x in sub():
        yield x

print(list(wrapper()))   # [1, 2]
```

能用。于是很多人得出一个结论：`yield from sub()` 不过就是上面这个 `for` 循环的缩写，纯粹省两行字。

这个结论是**错的**，而且错得会让你在某一天踩一个大坑。看下面这个把 `yield` 当双向通道用的例子（上一篇[《Python 生成器》](/2026/08/17/python-generator.html)讲过 `.send()`）：

```python
def sub():
    while True:
        x = yield
        print("子生成器收到:", x)

def wrapper_for():          # 用 for-yield 转发
    for x in sub():
        yield x

def wrapper_from():
    yield from sub()
```

两个看起来只是写法不同，但用 `.send(10)` 往 `wrapper_for` 里塞值，你会发现子生成器**永远收不到**那个 10；而 `wrapper_from` 能收到。一个字符的写法之差，行为天差地别。

`yield from` 到底做了什么？为什么一个看似语法糖的东西，会让 Greg Ewing 专门写一份 [PEP 380](https://peps.python.org/pep-0380/)、让 Python 3.3 推迟发布来等它、甚至直接催生了后来的 `asyncio`？这篇文章把它讲透。

<!--more-->

## L1：先让它跑起来——把一个生成器"转手"出去

你现在在第一层，目标是会用 `yield from`。

它最直接的用法，是在一个生成器里把另一个可迭代对象的值**原样**产出：

```python
def chain(*iterables):
    for it in iterables:
        yield from it

print(list(chain([1, 2], (3, 4), "ab")))   # [1, 2, 3, 4, 'a', 'b']
```

这比手写 `for x in it: yield x` 干净，而且任意可迭代对象（列表、元组、字符串、生成器）都能接。它最常见的用武之地是**递归展开嵌套结构**：

```python
def flatten(tree):
    for node in tree:
        if isinstance(node, list):
            yield from flatten(node)   # 递归把子树里的所有值交出去
        else:
            yield node

data = [1, [2, 3, [4, 5]], 6]
print(list(flatten(data)))   # [1, 2, 3, 4, 5, 6]
```

到这里，`yield from` 看起来确实像个便利写法。但记住这个直觉，我们马上就要拆掉它。

## L2：底层发生了什么——它建了一条透明的双向管道

到这里你已经会用了，但有个问题绕不过去：如果 `yield from` 只是缩写，开头那个 `.send(10)` 的例子为什么行为不同？

答案是：**`for x in sub(): yield x` 只转发了"值往外流"这一个方向；`yield from` 建的是一条双向、透明的管道，把调用者和子生成器直接连了起来。**

什么叫"双向"？回忆一下，生成器的 `yield` 是一扇双向门：值从门里递出去（`yield x`），调用者下次敲门也能塞值进来（`g.send(v)`，`v` 成为 `yield` 表达式的值）。除了值，还有三样东西能穿过这扇门：

| 操作 | 含义 |
|---|---|
| `next(g)` / `g.send(v)` | 让生成器往前跑一步，可顺带塞一个值 |
| `g.throw(exc)` | 从生成器当前暂停的 `yield` 处抛出异常 |
| `g.close()` | 从暂停处抛出 `GeneratorExit`，要求生成器结束 |
| `return value` | 生成器结束时带一个返回值（藏在 `StopIteration.value` 里） |

`for x in sub(): yield x` 只处理了第一行里"往外吐值"的那一半：

```python
def wrapper_for():
    g = sub()
    next(g)                 # 启动子生成器（如果它是协程式的）
    for x in g:
        received = yield x  # 调用者 send 进来的值进到这里……
        # ……然后 received 就被扔进垃圾桶了，没人把它交给 g！
```

调用者 `wrapper.send(10)`，10 赋给了 `received`，但这层包装根本没调 `g.send(received)`，于是子生成器那侧的 `x = yield` 永远等不到任何东西。`throw`、`close` 同理——全被中间这层吃掉了。

而 `yield from` 做的事，是把这**五样东西全部原样、正确地转发**给子生成器。它相当于在中间这层放了一个翻译官：外面喊什么，它一字不漏地递给里面；里面递什么出来，它也一字不漏地递出去。

我们直接对比验证：

```python
def sub():
    while True:
        x = yield
        print("子生成器收到:", x)

def wrapper_from():
    yield from sub()        # 透明管道

g = wrapper_from()
next(g)
g.send(10)                  # 子生成器收到: 10
g.send(20)                  # 子生成器收到: 20
```

换成手写的 `for` 循环，`send` 进来的 10、20 全部石沉大海。**这不是省两行字的区别，是"正确"和"错误"的区别。**

### 子生成器的 return 值，是 yield from 表达式的值

双向管道还有一个被很多人忽略的关键能力：它能把子生成器 `return` 回来的值带回来。

普通生成器 `return x` 时，`x` 不会被当作业界产出的值，而是塞进 `StopIteration` 异常的 `.value` 属性里。平时 `for` 循环会把这个值丢掉，但 `yield from` 会把它捞出来，作为整个 `yield from` 表达式的**返回值**：

```python
def averager():
    total = 0
    count = 0
    while True:
        x = yield
        if x is None:            # 收到 None 表示结算
            return total / count if count else 0
        total += x
        count += 1

def runner():
    avg = yield from averager()  # 子生成器 return 的值，赋给 avg
    print("平均值是:", avg)

def run():
    g = runner()
    next(g)            # 启动
    g.send(10)
    g.send(20)
    g.send(None)       # 触发子生成器 return

try:
    run()
except StopIteration:
    pass               # runner 跑完自然结束，忽略
# 打印：平均值是: 15.0
```

> 最后那个 `StopIteration` 是外层 `runner` 正常结束抛出的（任何生成器跑完都会抛，`for` 循环帮你吃掉了而已），这里手动捕获只是为了让输出干净。重点看 `avg` 是怎么拿到 `15.0` 的。

注意 `avg = yield from averager()` 这一行：左边的 `avg` 拿到的不是子生成器 `yield` 出来的东西（它根本没 yield 任何值），而是它最终 `return` 的那个 15.0。手写 `for` 循环要做到这件事，得自己去捕获 `StopIteration` 并读取 `.value`，又脏又容易错。

> **先预测，再看答案**：下面这段打印什么？
>
> ```python
> def gen():
>     yield 1
>     return "done"
>
> def main():
>     result = yield from gen()
>     print("result =", result)
>
> list(main())
> ```
>
> <details><summary>点我看答案</summary>
>
> 先打印 `result = done`，然后 `list` 拿到的是 `[1]`。
>
> `1` 是子生成器 `yield` 出来的，作为产出值被外层 `list` 收走；`"done"` 是 `return` 出来的，被 `yield from` 捞出来赋给 `result`。**两件东西走两条路**：yield 出去的值给"迭代我的人"，return 回来的值给"包含我的那个 yield from 表达式"。理解了这一点，`yield from` 就懂了一大半。
>
> </details>

## 台阶句：为什么不直接让 Python 自动展开？

到这里你懂了 `yield from` **怎么**工作，但最该问的问题还没回答：Python 为什么不干脆让一个生成器"自动"把对它的调用展开，非要发明一个新关键字？或者说，调用子生成器这件事，普通函数调用 `sub()` 不就好了，跟 `yield from` 有什么本质区别？

区别在于**栈帧**。我们在上一篇讲过：生成器的精髓是它能暂停，暂停时整个栈帧（局部变量、执行到哪行）被挂在生成器对象上。当外层 `wrapper` 执行到普通的 `sub()` 调用时，会**新建**一个栈帧，`wrapper` 的栈帧在下面等着；`sub` 跑完返回，它的栈帧销毁。这是普通的函数调用栈，一层压一层。

但如果 `wrapper` 想在 `sub` 暂停的**整个期间**保持暂停、让调用者直接和 `sub` 通信，普通函数调用做不到——因为调用者手里只拿着 `wrapper` 这一个生成器，它每 `send` 一次，控制权先进 `wrapper` 的栈帧，再想办法转进 `sub`。如果没有 `yield from`，每一次转发都要 `wrapper` 醒过来、手动调一次 `sub.send()`、再把结果 `yield` 出去——三层栈帧来回弹跳，又慢又啰嗦，还得处理上面说的 send/throw/close/return 全部细节。

`yield from` 的做法是：**在子生成器暂停期间，外层生成器直接"短路"掉自己，让调用者和子生成器的栈帧直接对接。** 外层不再每次都醒过来当传话筒，它像一根透明的导线接在两者之间。等子生成器真正 `return`，外层才醒过来，拿到返回值继续往下跑。

这正是 PEP 380 里那句被引用烂了的话的核心思想——

> "A generator can now delegate part of its operations to another generator."（生成器现在可以把自己的一部分操作**委派**给另一个生成器。）

注意是 **delegate（委派）**，不是 "缩写" 或 "便利写法"。委派的含义是：我把这段期间的控制权整个交出去，期间发生什么我不掺和，等你办完了把结果还给我。

### 想看展开后的样子？PEP 380 给了

`yield from EXPR` 在语义上等价于下面这一大坨（节选自 PEP 380，实际展开还要处理边界和异常，比这更长）：

```python
_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    while 1:
        try:
            _s = yield _y              # 把 _y 递出去，等着接收 send 进来的值
        except StopIteration as _e:
            _r = _e.value
            break
        try:
            if _s is None:
                _y = next(_i)
            else:
                _y = _i.send(_s)      # 把收到的值转发给子生成器
        except StopIteration as _e:
            _r = _e.value
            break
RESULT = _r
```

这还没算 `throw` 和 `close` 的转发——那部分再加二十多行。**`yield from` 一个关键字，替你正确地写了这四五十行充满边界条件的代码。** 现在你能体会为什么它不是语法糖了：它封装的是一整套"子生成器生命周期与异常协议"，手写几乎必错。

## L3：本质——把两个可暂停的执行片段焊在一起

去掉所有术语，`yield from` 到底是什么？

> **`yield from` = 透明地把一个生成器"串接"到另一个生成器后面，让调用者和最内层的生成器直接通信，中间各层既不丢消息也不掺和，直到最内层办完，把结果顺着管道原路送回来。**

它解决的核心问题是：**生成器的重构**。没有它之前，你一旦把一个大生成器拆成几个小函数，`.send()`、`.throw()` 这些能力就全断了——因为普通函数调用建不起"暂停期间的透明通道"。有了它，你可以放心地把一个生成器拆成多层，对外行为完全不变，就像内联写在一起一样。这是普通函数早就享有的"提取子函数"自由，生成器晚了十年才拿到。

这里值得追问一层设计权衡：**为什么不用普通函数调用，而要发明新语法？** 乍一看，让生成器调另一个生成器，直接 `sub()` 然后自动转发不就行了？

不行。关键冲突在于：普通函数调用是**栈式、嵌套**的——调用者阻塞等待子函数返回，期间它自己是"睡着"的，外部无法和它通信。但生成器的全部意义就在于"暂停时还能接收消息"。如果 `wrapper` 用普通方式调用 `sub()`，那在 `sub` 暂停期间，`wrapper` 也被压在栈底动不了，调用者 `send` 给 `wrapper` 的消息根本送不进去——它得先经过 `wrapper` 的栈帧，而那帧正在等 `sub` 返回，死锁。

`yield from` 给出的答案是**打破严格的栈式调用**：在子生成器活跃期间，外层生成器把自己从"必经之路"上挪开，让调用者的消息直达内层。这在普通的函数调用语义里是做不到的，所以它必须是一个新的原语，而不是哪个已有机制的自动行为。这个"打破栈"的决定影响深远——它让生成器第一次能**组合**，而能组合，就能往更高的抽象爬。

### 一段历史：它真正的使命是协程

`yield from` 在 2009 年由 Greg Ewing 在 PEP 380 中提出，2012 年随 Python 3.3 落地。表面理由是"方便生成器重构"，但 Ewing 和 Guido 心里都装着更大的东西：**协程**。

协程的核心需求是"一个协程能调用另一个协程，并等待它完成"，同时不阻塞整个线程。这件事用纯生成器能模拟（上一篇的 `.send()`），但只要协程想调用子协程，就会撞上前面说的"消息转发"地狱——你得手写那四五十行管道代码，层层重复。`yield from` 一出，这个问题瞬间消失：协程 `yield from` 另一个协程，调用者的消息直达最内层，返回值原路返回。

Guido 本人在 2012 年前后的[演讲](https://pyvideo.org/pycon-us-2013/)里多次直言：tulip（也就是后来的 `asyncio`）的设计**依赖** `yield from`，他甚至推动把这个特性从 Python 3.4 提前挪到 3.3，好给 asyncio 铺路。后来的故事你知道了：Python 3.5 加了 `async/await` 语法糖，其中 `await x` 在底层几乎就是 `yield from x` 的直系后代（只是限定了 awaitable 对象）。**你今天写的每一行 `await`，底下都站着 2009 年那个 `yield from`。**

一个"便利写法"，不会让一种语言的并发模型改道。`yield from` 做到了，因为它从来不是便利写法。

## 动手环节

### 实验 1：亲手验证 for-yield 会吞掉 send

把下面这段存成 `pipe.py`。先在心里预测输出，再运行：

```python
def sub():
    while True:
        x = yield
        print("  sub 收到:", x)

def wrapper_for():
    g = sub()
    next(g)
    for v in g:
        s = yield v
        print("  wrapper 收到 send:", s, "（但没转给 sub）")

def wrapper_from():
    yield from sub()

print("=== for-yield ===")
w = wrapper_for()
next(w)
w.send(10)
w.send(20)
w.close()

print("=== yield from ===")
w = wrapper_from()
next(w)
w.send(10)
w.send(20)
w.close()
```

你会看到 `for-yield` 版本里 wrapper 收到了 10、20 却没转给 sub——sub 只打印出一堆 `None`（那是 `for v in g` 自己用 `next` 驱动迭代时塞进去的），从来没收到用户 send 的 10、20；而 `yield from` 版本里 10、20 直达 sub。**"消息能不能穿透中间层"，就是这两种写法的根本分野。**

### 实验 2：用 yield from 递归遍历目录

写一个生成器，递归产出某个目录下所有文件的路径。体会"递归 + yield from"怎么把一棵树自然地铺平：

```python
import os

def all_files(path):
    for name in os.listdir(path):
        full = os.path.join(path, name)
        if os.path.isdir(full):
            yield from all_files(full)   # 子目录里的文件，全部交出去
        else:
            yield full

for f in all_files("."):
    print(f)
```

试着把 `yield from all_files(full)` 改写成 `for x in all_files(full): yield x`——功能在这里确实等价（因为你没有用 send/throw），但请记住：一旦哪天你想让这个生成器支持 `.send()` 或 `.throw()`，缩写版就会坏掉，而 `yield from` 版本天然正确。**默认用 `yield from`，别手写 for 循环。**

### 自测问题

答得上来，说明这篇你真的读进去了：

1. `for x in gen: yield x` 和 `yield from gen` 在什么场景下行为**不一致**？请至少说出两个。（提示：send、throw、return value。）
2. 下面代码打印什么？为什么 `result` 不是 `[1, 2]`？
   ```python
   def gen():
       yield 1
       yield 2
       return 99
   def main():
       result = yield from gen()
       print("result =", result)
   list(main())
   ```
3. 为什么说"普通函数调用无法实现生成器之间的委派"？（提示：想想栈帧和暂停期间谁在收消息。）

---

## 收尾地图

**什么时候用 yield from：**

| 场景 | 做法 |
|---|---|
| 在生成器里产出另一个可迭代对象的所有值 | `yield from it`，别手写 for 循环 |
| 递归/嵌套结构的展开 | `yield from recursive(...)` |
| 子协程/子生成器需要接收 `.send()`、`.throw()` | **必须**用 `yield from`，手写 for 会丢消息 |
| 需要拿子生成器的 `return` 值 | `result = yield from gen()` |
| 只是想拼接几个列表 | 普通函数里用 `list(chain(*iters))` 或 `extend` 即可，不必非用生成器 |

**下一站去哪：**

- 看懂了"透明管道"和"委派"，去读 [PEP 380](https://peps.python.org/pep-0380/) 本身，尤其是末尾那段"正式语义"展开——现在再看那几十行状态机，你会发现每一行都对应文中讲的一种转发。
- 接着去看 **`asyncio`** 和 `async/await`：把这篇里的"生成器"换成"协程"，把"`next/send`"换成"事件循环的调度"，你会发现 `await` 就是限定了对象类型的 `yield from`。从 `yield` → `yield from` → `await`，一条线串起来。
- 想玩点花的，看标准库 **`contextlib`** 里的 `@contextmanager` 是怎么用 `yield` 把"进入/退出"拼成上下文管理器的——那是生成器作为"可暂停片段"的另一种用法，和 `yield from` 的组合思想一脉相承。

`yield from` 教给人的不只是一个语法，而是一个关于**组合**的道理：一个抽象能不能长大，取决于它能不能透明地嵌套。函数之所以强大，是因为函数能调函数；生成器之所以能长成协程，是因为有了 `yield from` 之后，生成器终于能"调"生成器。**在那之前，它只是个能暂停的函数；在那之后，它成了能搭建并发大厦的砖块。**
