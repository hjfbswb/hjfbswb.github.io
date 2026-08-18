---
title: Python 的 module 和 package：import 到底导入了什么
tags: [python]
categories: [Python]
---

先做一个实验。新建一个 `greet.py`：

```python
# greet.py
print("模块被加载了")

def hello():
    print("hello")
```

然后在同一个目录打开解释器：

```python
>>> import greet
模块被加载了
>>> import greet      # 再来一次
>>>                   # 什么都没打印
```

`print` 只执行了一次。这说明 `import` 不是"把文件内容粘贴到当前位置"——否则两次 import 应该打印两次。那它到底做了什么？被"导入"的 `greet` 又是什么东西？很多人写了好几年 Python，能分清 `from x import y` 和 `import x`，却说不清 `__init__.py` 为什么有时必须有时可以没有、相对导入为什么一运行就报错、以及循环导入为什么有时候能成有时候炸。这篇文章把这些一次性讲透。

<!--more-->

## 第一层：会用——两个词，三种写法

你现在在第一层，目标是先把术语对齐。

- **模块（module）**：一个 `.py` 文件就是一个模块。文件名去掉 `.py` 就是模块名，`greet.py` → 模块名 `greet`。
- **包（package）**：一个**装着模块的目录**。包里可以有子包，形成 `a.b.c` 这样的层级。

三种导入写法，差别只在"把什么名字绑到当前作用域"：

```python
import greet                       # greet 这个名字 -> 整个模块对象
greet.hello()

from greet import hello            # hello 这个名字 -> 模块里的那个函数
hello()

from greet import *                # 把模块里的名字一股脑搬进来
```

到这里你已经会用了。但有一个问题绕不开：Python 怎么知道一个目录是"包"而不是"随便放了几个 py 的文件夹"？答案是那个著名的 `__init__.py`：

```
myapp/
├── __init__.py        # 有它，myapp 才是一个（常规）包
├── auth.py
└── views/
    ├── __init__.py
    └── home.py
```

`__init__.py` 可以是空文件，它的存在本身就是标记。你可以在里面写初始化代码，包被导入时这些代码会执行——这是后话。

**先预测再看答案**：既然 `__init__.py` 是包的标志，那么在 Python 3.3 以后，如果一个目录里**没有** `__init__.py`，`import` 它会怎样？

- A. 必然报 `ModuleNotFoundError`
- B. 居然能导入
- C. 看人品

答案是 B。先记住这个反直觉的事实，第三层会解释为什么。

**台阶句**：你已经知道怎么写 import、怎么摆目录了，但这解释不了开头那个现象——第二次 `import greet` 为什么不重新执行文件？要回答它，必须钻进 import 的内部看一眼。

## 第二层：原理——import 是"执行并缓存"，不是"粘贴"

你现在在第二层。这一层回答：`import greet` 这一行，解释器到底干了什么。

它大致分三步：

**第一步：找。** 解释器去 `sys.path` 列出的一串目录里挨个找，看有没有叫 `greet` 的模块或包。`sys.path[0]` 通常是当前脚本所在目录（或交互模式下的空字符串，表示当前目录），所以同目录的文件能直接 import。

**第二步：执行。** 找到 `greet.py` 后，解释器**从头到尾执行一遍这个文件**——顶层的 `print`、变量赋值、`def` 函数定义，全部真的跑一遍。执行过程中产生的名字（变量、函数、类）被收进一个新建的**模块对象**的命名空间里。

**第三步：缓存并绑定。** 这个模块对象被存进全局字典 `sys.modules`，键是模块名 `"greet"`；然后把名字 `greet` 绑到你当前的作用域。

关键就在第二步和第三步之间。下一次再 `import greet`，解释器先查 `sys.modules`，发现 `"greet"` 已经在了，**直接拿缓存的对象，根本不重新执行文件**。开头实验里第二行 `print` 没输出，原因就在这：模块是**单例**，一个进程里每个模块只加载一次。

你可以亲眼验证：

```python
>>> import greet, sys
模块被加载了
>>> greet is sys.modules["greet"]
True
>>> greet.hello
<function hello at 0x...>
```

`greet` 不是什么特殊的语法产物，它就是 `sys.modules` 里那个普通对象的一个引用。

### 包的导入：`__init__.py` 就是包本身

导入一个包时，机制完全一样，只不过"执行文件"这步执行的是该目录下的 `__init__.py`：

```python
import myapp          # 执行 myapp/__init__.py，得到模块对象 myapp
```

换句话说，**`myapp` 这个名字指向的模块对象，就是 `myapp/__init__.py` 执行后的产物**。`__init__.py` 里定义的名字，能通过 `myapp.xxx` 访问；它没暴露的子模块，你得显式 `import myapp.auth` 才能用。

一个常用技巧：在 `__init__.py` 里把子模块的常用名字"抬"到包一层，用户写起来更短：

```python
# myapp/__init__.py
from .auth import login, logout      # 相对导入，下一节讲
```

```python
>>> import myapp
>>> myapp.login        # 不用写 myapp.auth.login
```

### `from x import *` 和 `__all__`

`from greet import *` 默认会把模块里所有不以下划线开头的名字搬进来。这很容易污染命名空间（你不知道自己到底搬了什么进来），所以模块可以用 `__all__` 显式声明"对外公开哪些"：

```python
# greet.py
__all__ = ["hello"]     # 只公开 hello

def hello(): ...
def _internal(): ...    # 即使没写进 __all__，下划线开头本就不被 * 导入
helper = 42             # 不在 __all__ 里，import * 时不会被搬进来
```

`__all__` 只影响 `import *`，不影响显式的 `from greet import helper`。

### 相对导入：那个总报错的点

在包里，你可以用点号引用同包或父包的模块：

```python
# myapp/views/home.py
from . import sidebar         # 一个点 = 当前包 (myapp.views)，引同目录的 sidebar.py
from .. import auth           # 两个点 = 父包 (myapp)
from ..auth import login      # 从父包的 auth 模块里取 login
```

相对导入有一个新手必踩的坑：**直接把模块当脚本跑就会炸**：

```sh
$ python myapp/views/home.py
ImportError: attempted relative import with no known parent package
```

原因是：当你用 `python home.py` 直接运行它时，这个文件被当成**顶层脚本**（`__main__`），它"不知道"自己属于 `myapp.views` 这个包，自然找不到"父包"在哪。正确的跑法是以模块方式运行，让解释器从包的视角定位它：

```sh
$ python -m myapp.views.home
```

记住这条准则：**包内用相对导入，跨包用绝对导入；想用相对导入就用 `-m` 跑，别直接跑文件。**

到这里机制基本通了。**但还有一个更根本的问题没回答：既然 module 是 `sys.modules` 里的一个对象，那它和普通对象到底有什么不一样？package 又特殊在哪？以及——第一层留的悬念——没有 `__init__.py` 的目录为什么也能导入？**这是最后一层。

## 第三层：本质——module 是一等对象，package 是带 `__path__` 的 module

你现在在第三层。先建立全文最关键的一个心智模型：

> **模块不是文件。模块是一个运行时对象（`types.ModuleType` 的实例），文件只是它最初的来源。`import` 做的事是：找到来源、执行出一个命名空间、把这个命名空间对象装进 `sys.modules`。**

去掉术语跟同事解释：`.py` 文件是菜谱，模块对象是按照菜谱做出来的那道菜；`import` 是按菜谱做菜并端上桌，第二次 `import` 只是指着桌上那盘菜说"我也要一份"——不会重做。

因为模块是对象，你可以像操作任何对象一样操作它：

```python
>>> import greet
>>> greet.custom = "我给模块加了个属性"     # 运行时动态挂属性
>>> type(greet)
<class 'module'>
>>> import types
>>> isinstance(greet, types.ModuleType)
True
```

这也解释了循环导入为什么"时好时坏"。假设 `a.py` 里 `import b`，`b.py` 里 `import a`：

1. 执行 `import a`，开始跑 `a.py`；
2. 跑到 `import b`，开始跑 `b.py`；
3. `b.py` 跑到 `import a`——此时 `sys.modules` 里**已经有一个半成品的 `a`**（执行到一半，后面的名字还没定义），于是直接拿到它；
4. 如果 `b` 此时访问 `a` 里**还没定义到的名字**，就炸 `AttributeError`；如果它只在函数内部引用 `a.xxx`（要等函数被调用时才查），那时 `a` 早已执行完，就没事。

所以循环导入不是必然报错，而是"看你在什么时机访问什么"。最干净的解法还是重构掉循环依赖，但理解了半成品模块的存在，你就能看懂那些"明明循环引用了却能跑"的代码。

### package 的本质：一个有 `__path__` 的模块

那 package 和 module 到底什么关系？答案出乎意料——**包也是模块**，只不过它多了一个 `__path__` 属性：

```python
>>> import myapp
>>> myapp.__path__
['/.../myapp']
```

`__path__` 是一个目录列表，告诉解释器"这个包下面的子模块去这些目录里找"。普通模块没有 `__path__`，所以不能再往下包含子模块。

- **常规包（regular package）**：有 `__init__.py` 的目录，`__path__` 通常就一项，指向该目录。
- **命名空间包（namespace package）**：没有 `__init__.py` 的目录。这就是第一层问题的答案。

### 为什么允许没有 `__init__.py`？

这是 Python 3.3（PEP 420）引入的。设计动机来自一个现实痛点：**一个大型项目想把同一个包拆分到多个目录分发**。比如你装了两个库，它们各自提供 `zope.foo` 和 `zope.bar`，分布在 `site-packages` 下不同位置，但你希望它们共同组成一个 `zope` 命名空间。

如果要求每个目录都有 `zope/__init__.py`，两个库就会争抢同一个文件，谁后装覆盖谁。命名空间包的解法是：解释器扫 `sys.path` 时，把所有匹配 `zope/` 且没有 `__init__.py` 的目录**合并**成一个包，它的 `__path__` 同时包含这些目录，子模块可以分散在各处：

```python
>>> import zope
>>> zope.__path__
_NamespacePath(['/path1/zope', '/path2/zope'])   # 多个目录拼起来的
```

代价是：命名空间包本身没有 `__init__.py`，**不能写包级初始化代码**；而且解释器要扫描整条 `sys.path` 去拼目录，导入略慢。所以日常写应用，**你应该继续加 `__init__.py`**——它明确、可初始化、导入快。命名空间包是为分发包的大型组织准备的，不是日常默认。

**一段历史**：在 Python 最早的年代，模块只能是单个 `.py` 文件，没有"包"的概念。1998 年 Python 1.5 才引入包机制，当时需要一个标记区分"包目录"和"普通目录"，`__init__.py` 就这样诞生了——选它是因为它既能当标记，又能顺理成章地承载初始化代码。这个设计沿用了 15 年，直到 PEP 420 才补上"无 `__init__.py`"的缺口。所以你看到的两种写法，不是新老谁取代谁，而是为两种不同场景并存的工具。

## 收尾地图

一张判据表，遇到时直接查：

| 现象 / 问题 | 答案 |
|---|---|
| 第二次 import 不执行代码 | 模块是单例，缓存在 `sys.modules` |
| 一个模块在进程里加载几次 | 恰好一次（除非手动 `importlib.reload`） |
| 包和模块的关系 | 包是带 `__path__` 的模块 |
| 目录必须有 `__init__.py` 吗 | 常规包要；命名空间包不要（Python 3.3+） |
| `__all__` 影响什么 | 只影响 `from x import *`，不影响显式导入 |
| 相对导入报错怎么办 | 用 `python -m 包.模块` 运行，别直接跑文件 |
| 循环导入一定炸吗 | 不一定，看是否在模块执行中途访问尚未定义的名字 |
| 日常该用哪种包 | 常规包（加 `__init__.py`），命名空间包留给分发包 |

两道自测题（不回头看能答上，就是真懂了）：

1. 你在 `a.py` 顶部 `import b`，`b.py` 顶部 `import a`，在 `b` 里只在某个函数内部调用 `a.foo()`，程序却能正常跑。为什么？（答案在第三层"半成品模块"。）
2. 两个没有 `__init__.py` 的同名目录分处 `sys.path` 的不同位置，`import` 这个名字会发生什么？如果其中一个目录加了 `__init__.py`，结果又如何？（答案在第三层命名空间包与"合并"机制。）

**下一站**：理解了"import = 执行并缓存模块对象"，再去看 `importlib` 这个标准库就会觉得理所当然——它不过是把解释器内部那套"找、执行、缓存"的机制暴露成了 Python 代码，让你能动态导入、热重载、自定义导入器（finder/loader）。再往后是 `sys.path` 的初始化顺序（脚本目录、`PYTHONPATH`、site-packages）和虚拟环境如何改写它——搞懂这些，`ModuleNotFoundError` 对你就不再是玄学，而是一道可以顺着 `sys.path` 一步步排查的逻辑题。
