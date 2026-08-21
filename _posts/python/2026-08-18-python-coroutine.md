---
title: Python 协程：从生成器到 async/await，一条线讲透
tags: [python]
categories: [Python]
last_modified_at: 2026-08-18 08:21:25 +0800
---

先看一段你大概写过、也困惑过的代码：

```python
import asyncio, time

async def fetch(name, delay):
    print(f"{name} 开始")
    await asyncio.sleep(delay)
    print(f"{name} 结束")
    return name

async def main():
    t0 = time.time()
    await asyncio.gather(fetch("A", 1), fetch("B", 1), fetch("C", 1))
    print(f"耗时 {time.time()-t0:.1f}s")      # 耗时 1.0s，不是 3.0s

asyncio.run(main())
```

三个"函数"各睡 1 秒，总耗时却是 1 秒。很多人第一次看到这个输出时脑子里冒出的第一个问题是：**它们到底是怎么"同时"跑的？是开了三个线程吗？**

不是。整个程序就一个线程，你可以在 `fetch` 里任何位置加一句 `print(threading.current_thread())`，三个全是同一个线程。也没有什么魔法——`asyncio.sleep` 在睡的时候，主动把 CPU 让了出去，让别人先跑；它睡醒了，再被叫回来接着跑。

"一个函数能在半路暂停、稍后从原地继续"——这听起来耳熟吗？对，这就是我们在[《Python 生成器》](/2026/08/17/python-generator.html)里讲过的东西。这篇文章要讲清楚一件事：**协程不是什么新发明，它就是那个"能暂停的函数"被事件循环接管之后的样子。** 你已经在 `yield` 里见过它的雏形，在 [`yield from`](/2026/08/18/python-yield-from.html) 里见过它的骨架；今天我们把它的最后一层窗户纸捅破。

<!--more-->

## L1：先让它跑起来——你已经会用协程了

你现在在第一层。目标只有一个：写出能并发的代码，不纠结原理。

协程的最小用法有三条规则：

1. 用 `async def` 定义一个协程函数（而不是普通 `def`）。
2. 在它内部用 `await` 等待另一个协程（或任何"可等待对象"）。
3. 用 `asyncio.run(...)` 启动最外层那个。

```python
import asyncio

async def say(msg, delay):
    await asyncio.sleep(delay)
    print(msg)

async def main():
    await say("你好", 1)       # 顺序等：花 1 秒
    await say("世界", 1)       # 再等 1 秒

asyncio.run(main())           # 总共 2 秒
```

但这么写没并发——`await` 就是"等它做完"，两个 `await` 串起来就是串行。要并发，得把协程**包成 Task 一起跑**：

```python
async def main():
    await asyncio.gather(
        say("你好", 1),
        say("世界", 1),
    )                         # 两个同时睡，总共 1 秒
```

`asyncio.gather` 干的事是：把这几个协程全部"安排上"，然后在它们之间反复切换——一个在 `await asyncio.sleep` 处暂停等时间时，就去跑另一个；等时间到了再切回来。一个线程，靠"你等的时候我跑别人"把等待时间全抹平。

到这里你已经能写异步代码了。但这个解释会留下一个让人睡不着觉的问题：`await` 那一行，函数到底"暂停"到哪去了？谁在叫醒它？为什么普通函数做不到？

## L2：底层发生了什么——协程就是被循环驱动的生成器

要回答这个问题，我们先把 `async/await` 全部忘掉，用**纯生成器**手写一个能跑的事件循环。你会发现协程的全部秘密就摆在台面上。

### 一个用生成器写的"协程"

回忆[上一篇](/2026/08/18/python-yield-from.html)讲过的：`yield` 能让函数暂停，`g.send()` 能把它叫醒。这就够了。我们约定：协程想"等一会儿"时，就 `yield` 一个请假条对象，告诉调度器"我要等多久"：

```python
import time

class Sleep:
    """请假条：谁 yield 我，就表示'我要睡 delay 秒'。"""
    def __init__(self, delay):
        self.delay = delay

def sleep(delay):
    return Sleep(delay)

def coro(name, delay):
    print(f"  {name} 启动")
    yield sleep(delay)          # 请假：这期间去跑别人吧
    print(f"  {name} 回来了")
```

`coro` 是个普通的生成器函数。它启动时打印一句，然后 `yield` 出一个请假条并暂停；等调度器觉得时间到了，再 `send` 把它叫醒，它就打印第二句、结束。

### 事件循环：一个永不打烊的前台

现在写调度器。它的逻辑简单到朴素：把每个协程启动，记下它们的请假条和该回来的时间；谁的假到期了，就把谁叫醒。

```python
def run(generators):
    pending = []                            # (唤醒时间, 生成器)
    for g in generators:
        note = next(g)                      # 启动，拿到请假条
        pending.append((time.monotonic() + note.delay, g))

    while pending:
        pending.sort(key=lambda x: x[0])    # 谁最早醒，先处理谁
        wake_at, g = pending.pop(0)
        remaining = wake_at - time.monotonic()
        if remaining > 0:
            time.sleep(remaining)           # 真的没事干就睡（单线程）
        try:
            g.send(None)                    # 叫醒它，让它继续跑
        except StopIteration:
            pass                            # 协程正常结束
```

跑起来：

```python
run([coro("A", 0.3), coro("B", 0.1), coro("C", 0.2)])
```

输出顺序是：

```
  A 启动
  B 启动
  C 启动
  B 回来了
  C 回来了
  A 回来了
```

三个全部先启动，然后按睡醒时间依次回来。**这就是一个如假包换的协程运行时。** 没有线程，没有操作系统介入，没有任何魔法——就是一个 `while` 循环在一堆生成器之间反复 `send`，谁 `yield` 了请假条谁就靠边站，时间到了再叫它。

> **先预测，再看答案**：把上面 `run` 里的 `time.sleep(remaining)` 删掉，程序还能正确并发吗？总耗时会变吗？
>
> <details><summary>点我看答案</summary>
>
> 总耗时不会变，但 CPU 占用会飙到 100%。因为没有真等待时，`while` 循环会空转，反复检查"到点了没"。`time.sleep` 在这里就是"真的没人能跑时，把线程让给操作系统"。`asyncio` 真正的做法更聪明——它用操作系统提供的 `select/epoll` 一次性等到"最早的那个事件"到来，既不空转也不轮询。但**调度逻辑和这个手写版完全一致**：暂停、排队、到期唤醒。
>
> </details>

### async/await 做了什么？把 yield 换成 await

现在谜底可以揭开了。`async/await` 在做的事，和你刚看到的生成器版本**几乎一模一样**，只是把约定标准化了：

| 生成器版本 | async/await 版本 |
|---|---|
| `def coro(): yield ...` | `async def coro(): await ...` |
| `yield` 一个请假条 | `await` 一个可等待对象（Awaitable） |
| 你自己写 `run()` 循环 | `asyncio.run()` 提供事件循环 |
| `g.send(None)` 叫醒 | 事件循环内部 `send`/调度 |
| `StopIteration.value` 带回返回值 | 协程的返回值直接 `x = await ...` |
| `yield from sub()` 委派子协程 | `await sub()` 等待子协程 |

最关键的一行是最后一个对应关系。我们在[上一篇](/2026/08/18/python-yield-from.html)花了整篇文章讲：`yield from` 建立了一条调用者到子生成器的透明双向管道。**`await` 就是这条管道在协程世界里的直系后代**——它让一个协程能"调用"另一个协程并等它完成，期间自己暂停、把线程让给事件循环，子协程完事了再把返回值顺着管道送回来。事实上 Python 3.4 的 `asyncio` 最早就是用 `yield from` 写协程的，3.5 才加了 `async/await` 这套专用语法，本质是把"生成器被当协程用"这个惯例，升级成了一等公民。

所以，当你写下：

```python
async def main():
    data = await fetch("http://example.com")
```

发生的事情翻译成生成器语言就是：`main` 暂停，把"我要等这个 IO"的请假条交给事件循环；循环转头去跑别的协程；等网卡说数据到了，循环再 `send` 回 `main`，把数据作为 `await` 表达式的值交给它，`main` 从下一行继续。**没有任何线程被阻塞，被阻塞的只是 `main` 这一个协程——而它只是一个被挂起的栈帧对象，跟一个暂停的生成器没有任何区别。**

### 一个反直觉时刻：await 不是"启动并发"

很多新手以为 `await` 像 `threading.Thread.start()` 一样"把任务发出去"。完全不是。

```python
await fetch("A", 1)           # 这行只是：启动它，然后**当前协程暂停等它完成**
await fetch("B", 1)           # 这行要等上面完全结束后才执行
```

`await` 的语义是"我等它"，不是"让它跑"。要让多个协程并发跑，必须先把它们**全部交给事件循环**（`gather`/`create_task`），让循环把它们都启动、各自在第一个 `await` 处暂停排队，它们才有机会交错执行。单独一个 `await` 永远是串行的——就像你单独 `next(g)` 一个生成器，它也不会自己动起来。

## 台阶句：为什么非要有事件循环？线程不行吗？

到这里你懂了协程**怎么**被驱动，但最该追问的问题还没回答：既然操作系统早就提供了线程，线程也能在 IO 等待时切换，Python 为什么还要另起炉灶搞一套协程？

答案藏在"谁来决定切换时机"里。

### 协作式 vs 抢占式

线程是**抢占式**的：操作系统可以在任何两条指令之间把线程切走，你完全无法预知。这带来两个代价：

1. **要加锁**。因为任何一行代码执行到一半都可能被切走，两个线程操作同一个字典时，你必须用锁把"读-改-写"包起来，否则就是数据竞争。锁一旦用错，死锁、竞态全来了。
2. **切换有成本**。线程切换要进内核、保存恢复寄存器、刷缓存，一个线程还占着独立的栈内存（默认 8MB 量级）。一台机器开几千个线程就开始吃力。

协程是**协作式**的：切换只发生在 `await` 那一个点上。协程不主动 `await`，谁也不能把它切走。这意味着：

- **你的代码在两次 `await` 之间是原子的**，没人能半路插进来，共享状态不用加锁。
- 协程就是个普通对象，栈帧挂在它身上，一个协程占几 KB，开十万个都不稀奇。
- 切换全程在用户态，就是一次函数调用 + 保存恢复栈帧的代价，比线程切换轻一个数量级。

打个比方：线程像马路上每个人都在抢道，谁都可能随时变线，所以必须有红绿灯（锁）和交警（内核调度器）；协程像环岛，大家到了路口自觉停下让行——没有红绿灯，但因为每个人都在约定的点主动让，车流反而顺畅。**协程的快，不是因为它能"同时跑"，恰恰是因为它知道自己什么时候可以安全地停下来。**

### 但协作式有个致命弱点

代价是：一个协程如果**不 await、闷头算**，就会把整个事件循环堵住，其他所有协程全得干等。

```python
async def bad():
    total = 0
    for i in range(10**10):     # CPU 密集，一个 await 都没有
        total += i              # 这一跑，事件循环完全卡死，别的协程别想动
```

这不是 bug，是协作式的本质：让行是自愿的，没人能抢你的方向盘。所以协程**只适合 IO 密集**（大量时间在等网络/磁盘，等的时候自然会 await），不适合 CPU 密集——后者该用多进程或真线程。网上那句"协程比线程快"是有前提的，前提是你的任务大部分时间在**等**。

## L3：本质——用户态里的、协作让出的、可组合的并发

去掉所有术语，协程到底是什么？

> **协程 = 一个可以在指定位置主动暂停、把控制权交给一个调度器、稍后再从原地继续的函数；并发不靠操作系统分配 CPU，而靠这些函数在等待时彼此让行。**

这句话里有三个词缺一不可。**"指定位置"**决定了它不用加锁（切换点你看得见、数得清）；**"交给调度器"**说明它自己不调度，它只负责"我要等 X"，由事件循环决定等的期间去跑谁；**"彼此让行"**点出它的性能来源——不是并行，是把等待时间填上别人的活。

这里值得追问一层设计权衡：**为什么是"函数自己 yield"而不是"调度器强制切换"？** 后者就是线程啊，已经有了。协程的回答是：在 IO 密集这个特定场景下，99% 的等待都是"我在等网络/磁盘，这段时间我根本没有有用的事可做"——这种等待点天然就是切换点，让函数自己举手报告"我现在没事干了"，比让内核每隔几毫秒粗暴打断一次要高效得多，也简单得多。代价（CPU 密集会卡死）是真实的，但用对场景根本碰不到。**协程不是线程的替代品，是为"等待远多于计算"这一类 workload 量身做的更窄、更快的工具。**

而让协程真正能长成大程序的，是它的**可组合性**。这正是 `yield from`/`await` 那条透明管道的价值：

```python
async def get_user(user_id):
    resp = await http_get(f"/users/{user_id}")
    return resp.json()

async def get_posts(user_id):
    posts = await get_user(user_id)          # 等子协程
    return await http_get(f"/posts?author={posts['id']}")
```

`get_posts` 可以像普通函数调函数一样 `await get_user(...)`，中间这层在子协程等待时会透明地把控制权让给事件循环，子协程的返回值原路返回——这一切不需要你手写任何管道代码。**协程能"调"协程，它才能像函数一样层层封装、搭出大型系统；如果每调一个子协程都要手写那四五十行转发（就是上一篇 PEP 380 展开的那坨），异步代码永远只能停留在玩具阶段。** 从 `yield`（能暂停）到 `yield from`（能委派）再到 `async/await`（一等语法 + 标准事件循环），是一条逻辑环环相扣的进化链，缺了哪一环都长不成今天的样子。

### 一段历史：从痛骂到标配

Python 的异步故事开局并不光彩。2012 年左右，Python 3.3 加了 `yield from`，Guido 看中它能做协程，亲自带队搞了个代号 "Tulip" 的项目，就是后来的 `asyncio`，3.4 落地。那时写协程得用装饰器 `@asyncio.coroutine` 包一个生成器，里面 `yield from asyncio.sleep(1)`，又丑又容易和普通生成器搞混。社区怨声载道，很多人觉得"Python 要被 Node.js 带歪了"。

Guido 力排众议，推动 3.5（2015 年）加了 `async/await` 专用语法，把协程从"伪装的生成器"变成真正的一等类型。之后的几年里，aiohttp、httpx、FastAPI 依次把异步生态撑了起来。等到 2020 年 FastAPI 爆火，大家回头发现：那个被骂了快十年的设计，已经成了 Python 做高并发网络服务的默认答案。Guido 在一次访谈里说过，`asyncio` 是他在 BDFL 任期最后推动的几个大特性之一，争议大到他一度要在邮件列表里亲自下场吵架——但他认定 IO 并发是 Python 绕不过去的坎。

## 动手环节

### 实验 1：亲眼看见"阻塞事件循环"

把下面这段存成 `block.py`。先预测两个 fetch 的完成时间差，再运行：

```python
import asyncio, time

async def fetch(name, delay):
    print(f"{name} 开始 {time.time()-t0:.2f}s")
    await asyncio.sleep(delay)
    print(f"{name} 结束 {time.time()-t0:.2f}s")

async def cpu_hog():
    print(f"hog 开始 {time.time()-t0:.2f}s")
    total = sum(range(10_000_000))      # 闷头算，没有 await
    print(f"hog 结束 {time.time()-t0:.2f}s")

async def main():
    global t0
    t0 = time.time()
    await asyncio.gather(fetch("A", 0.1), cpu_hog(), fetch("B", 0.1))

asyncio.run(main())
```

你会看到 A 和 B 虽然各自只睡 0.1 秒，却都要等 `cpu_hog` 算完（约 0.3 秒）才结束。这就是"协作式"最直观的证据：hog 不主动让，别人谁也跑不了。把 `sum(range(...))` 换成 `await asyncio.sleep(0.3)` 再跑一次，A 和 B 会在 0.1 秒时准时结束——同一个线程，只因"让不让"的区别，行为天差地别。

### 实验 2：亲手写一个 30 行的事件循环

把 L2 里那个生成器事件循环亲手敲一遍（不超过 30 行），再加一个 `coro`，让它 `yield sleep(0.5)`。运行，观察启动顺序和回来顺序。然后试着把 `pending.sort(...)` 改成按插入顺序（不排序），看看会发生什么——你会直观地理解为什么事件循环必须维护一个按时间排序的调度队列。

等你能徒手写出这个循环，再回头看任何 `asyncio` 教程，会发现 `Future`、`Task`、`call_later` 这些概念，全是你这个 30 行小循环的工程化加强版。

### 自测问题

答得上来，说明这篇你真的读进去了：

1. 一个线程里跑着 1000 个 `asyncio` 协程，它们是"同时"在 CPU 上执行吗？如果不是，"并发"省下的时间到底从哪来？
2. 为什么下面两段代码的总耗时不同？`create_task` 在其中起了什么作用？
   ```python
   await fetch("A", 1); await fetch("B", 1)        # 2 秒
   task = asyncio.create_task(fetch("A", 1))
   await fetch("B", 1); await task                 # 1 秒
   ```
3. 协程在两次 `await` 之间操作一个共享列表，需要加锁吗？如果把这段逻辑放进两个**线程**里呢？为什么？
4. 用自己的话说说 `await` 和上一篇讲的 `yield from` 是什么关系。

---

## 收尾地图

**什么时候用什么：**

| 你的场景 | 选择 | 原因 |
|---|---|---|
| 大量网络/磁盘 IO，请求之间互相独立 | `asyncio` + 协程 | 等待时让行，单线程撑住海量并发，无需锁 |
| 大量 CPU 计算（数值、编解码、图像处理） | `multiprocessing` | 协程救不了你，必须真并行绕开 GIL |
| 调用一堆本身只有同步版本的阻塞库 | 线程池 (`run_in_executor`) | 那些库不会主动 `await`，放进线程里别堵事件循环 |
| 就两三件小事，代码简单 | 同步代码即可 | 并发不是免费的，别为了用而用 |

**下一站去哪：**

- 回头重读[《Python 生成器》](/2026/08/17/python-generator.html)和[《yield from》](/2026/08/18/python-yield-from.html)，现在你应该能看清那条完整的链：`yield`（可暂停的函数）→ `yield from`（可委派/组合的暂停片段）→ `async/await`（被事件循环标准化接管的协程）。
- 想真正吃透事件循环，读一遍 `asyncio` 标准库源码里的 `base_events.py`——从 `call_later` 和 `_run_once` 看起，你会发现它和你手写的 30 行循环是同一个骨架，只是多了 IO 多路复用、异常传播、任务取消这些工程细节。
- 用到生产环境时，去弄清 `TaskGroup`（3.11+，结构化并发）、`asyncio.timeout`、以及"永远不要在协程里调用阻塞函数"这三条铁律——它们决定了你的异步代码是稳健还是定时炸弹。

协程教给人的，不只是一个并发语法，而是一个更朴素的道理：**当你无法把一件事做得更快时，至少可以诚实地说出"我在等"，好让别人趁机干活。** 线程靠操作系统替你决定何时停下，协程把这个决定权交还给你——换来的是更轻的切换、更简单的共享状态，以及在等待堆积成山时，依然从容的那一个线程。
