---
title: Python 闭包：为什么三个函数都返回了 2
tags: [python]
categories: [Python]
last_modified_at: 2026-08-18 23:29:59 +0800
---

先看一段十行不到的代码：

```python
funcs = []
for i in range(3):
    funcs.append(lambda: i)

print([f() for f in funcs])
```

你猜输出是什么？按"常识"，三次循环时 `i` 分别是 0、1、2，每个 lambda 抓住"当时"的 `i`，那应该是 `[0, 1, 2]`。

实际跑一下：

```
[2, 2, 2]
```

全是 2。这是 Python 里最有名的"反直觉"之一，很多人栽在这里，骂一句"Python 闭包有 bug"，然后从 Stack Overflow 抄一个 `lambda i=i: i` 的写法，再不敢深究。

但这不是 bug。要真正看懂为什么是 2，你得先搞懂两件事：**Python 是怎么找一个变量的（作用域）**，以及**一个函数被"带走"时到底带走了什么（闭包）**。这两件事其实是同一件事的两面。

<!--more-->

## L1：先搞清楚 Python 去哪儿找变量

你现在在第一层，目标只有一个：看到一个名字，能准确说出 Python 去哪层找它。

Python 用的是**词法作用域（lexical scope）**——一个名字能不能被看见，在你写下函数、按下回车的那一刻就定了，和它在哪儿被调用无关。查找顺序是经典的 **LEGB** 四层，由内到外：

| 层 | 名字 | 是什么 |
|---|---|---|
| L | Local | 当前函数内部 |
| E | Enclosing | 外层（任意一层）函数的局部 |
| G | Global | 当前模块（文件）的全局 |
| B | Built-in | `builtins` 模块，`len`、`print` 这些住的地方 |

```python
x = "global"                # G

def outer():
    x = "enclosing"         # E
    def inner():
        x = "local"         # L
        print(x)
    inner()

outer()                     # local
```

把 `inner` 里的 `x = "local"` 删掉，它就会找到 E 层的 `"enclosing"`；再把 `outer` 里的也删掉，就落到 G 层的 `"global"`；三个都没有，才去 B 层碰运气——这也是为什么你写 `print` 不用 import：它在最外层兜底。

> **一个隐藏的坑**：Python 判定一个名字是不是局部变量，看的是**整个函数里有没有对它赋值**，而不是执行顺序。于是有了著名的 `UnboundLocalError`：
>
> ```python
> x = 1
> def f():
>     print(x)       # 这时候 x 明明已经"存在"了啊？
>     x = 2          # 就因为这一行，x 被判定为整个 f 的局部变量
> f()              # UnboundLocalError: local variable 'x' referenced before assignment
> ```
>
> 你以为先 `print` 再赋值能蹭到全局的 `x`，但编译器在函数定义时就扫到了 `x = 2`，于是 `x` 在整个 `f` 里都是局部的，第一行还没赋值就引用，报错。想在函数里改全局变量，得显式 `global x`；想改外层函数的变量，得 `nonlocal x`——这两个声明后面会讲。

### 先预测，再看答案

```python
a = 10
def f():
    a = 20
    def g():
        print(a)
    return g

f()()
```

<details><summary>点我看答案</summary>

输出 `20`。`g` 被调用时，虽然 `f` 已经执行完、它的栈帧都销毁了，但 `g` 里的 `a` 仍然找到了 `f` 里那个 `20`。这就是闭包在起作用——也是下一层要拆的东西。

</details>

## L2：闭包到底"闭"住了什么

到这里你会查作用域了，但开头那个谜题还没解。关键问题是：**`f` 都 return 了、栈帧都没了，为什么 `g` 还能读到 `f` 的局部变量？**

普通函数的局部变量确实跟着栈帧一起销毁。但如果一个内部函数**引用了外层函数的变量**，事情就不一样了。这个被引用的外层变量，在 Python 里有个专门的名字叫**自由变量（free variable）**；编译器会做一个特殊处理：

> 它在内存里造一个叫 **cell（格子）** 的小对象存放这个变量的值，让外层函数和内层函数**都指向同一个 cell**。

内层函数被 return 走的时候，它的 `__closure__` 属性里攥着这些 cell，cell 又攥着真实的值——外层栈帧没了无所谓，cell 还活着。

你可以亲眼看一下：

```python
def outer():
    x = 0
    def inner():
        return x
    return inner

f = outer()
print(f.__code__.co_freevars)    # ('x',)        —— 哪些名字是"抓"来的
print(f.__closure__)             # (<cell at ...: int object at ...>,)
print(f.__closure__[0].cell_contents)   # 0      —— cell 里此刻装的值
```

这就是闭包（closure）：

> **一个函数，加上它"出生"时周围那些被它引用的变量所对应的 cell，一起打包带走。**

注意这个"打包"打的不是值的快照，而是**装值的那个格子**。格子里的内容是会变的——记住这句话，开头的谜题马上就破了。

### 为什么三个 lambda 都返回 2

回头看开头：

```python
funcs = []
for i in range(3):
    funcs.append(lambda: i)
```

关键在于：**整个模块层级只有一个名叫 `i` 的变量，所有 lambda 共享同一个 cell。** 循环每走一圈，不是"新建一个 i"，而是把同一个 `i` 重新赋值成 0、1、2。三个 lambda 的 `__closure__` 全都指向那同一个 cell，而等你真正调用它们时，循环早就结束了，cell 里躺着的是最后的 `2`——于是全是 2。

画成图就是：

```
lambda0 ─┐
lambda1 ─┼──▶ cell(i) ──▶ 2
lambda2 ─┘
```

它们记住的**不是"我被造出来时 i 等于几"，而是"叫 i 的那个变量现在是几"**。这就是所谓的**迟绑定（late binding）**：闭包查的是调用时刻变量的值，不是定义时刻。

### 三种修法

既然病因是"共享一个 cell"，药方就是"给每个函数一个自己的 cell"。

**方法一：工厂函数，每调用一次造一层新作用域**

```python
def make_func(i):
    return lambda: i        # 这里的 i 是 make_func 的局部，每次调用都是新 cell

funcs = [make_func(i) for i in range(3)]
print([f() for f in funcs])    # [0, 1, 2]
```

`make_func` 每被调用一次，就有一个全新的栈帧、一个全新的 `i`、一个全新的 cell。三个 lambda 各抓各的，互不干扰。

**方法二：默认参数，在定义瞬间把值"抄"进来**

```python
funcs = [lambda i=i: i for i in range(3)]
```

默认参数在函数**定义时**就求值（而且是按值拷进去的），于是每个 lambda 的默认参数 `i` 各自记下了当时的 0、1、2。调用时 `f()` 不传参，这个默认值就派上用场。这是 Stack Overflow 上最常见的答案，但它其实是**绕开了闭包机制**——`i` 在这里已经是 lambda 自己的局部参数，不再是自由变量，`__closure__` 是空的。

**方法三：用 `functools.partial`**

```python
from functools import partial
funcs = [partial(lambda i: i, i) for i in range(3)]
```

道理类似：把当时的 `i` 作为参数绑定死，返回一个新的可调用对象。

> 对比一下：JavaScript 的 `let` 在 `for` 循环里每次迭代会新建一个绑定，所以 JS 里写 `funcs.push(() => i)` 是能得到 0/1/2 的；Python 没有这个机制，循环变量在整个循环期间就是同一个。这不是谁对谁错，是语言设计的不同选择——下一层会说 Python 为什么不抄这个作业。

## L3：本质——闭包到底是什么

去掉所有术语，闭包到底是什么？

> **闭包 = 一个函数，连同它"出生现场"里它在乎的那些变量，打包成一个还能继续调用的对象。**

一句话：**函数不再是一段光秃秃的代码，而是"代码 + 它诞生时的环境"**。普通函数像一张印着公式的纸，谁捡起来都一样；闭包像一个把公式和当时的草稿纸一起封进信封的东西，拆开时草稿纸上的数字还在。

这层"代码带着环境走"的能力，让一个原本只能"算完就走"的函数，**拥有了会记忆的状态**。看个最朴素的例子：

```python
def make_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

c = make_counter()
print(c())   # 1
print(c())   # 2
print(c())   # 3
```

没有类、没有实例属性、没有 `self`，一个函数就记住了自己被调用了多少次。秘密全在那个 cell 里：`counter` 和 `make_counter` 共享 `count` 的 cell，`make_counter` 退场后 cell 留给了 `counter`，于是状态被函数"私有"地持有了。这就是为什么有人说**闭包是"穷人的对象"**——它用一个函数加一组私有变量，干了类的一部分活。

### 为什么是迟绑定，而不是定义时拍照

现在追问那层设计权衡：既然迟绑定惹出了 `[2,2,2]` 这种坑，Python 为什么不干脆在函数定义时把自由变量的**当前值**拍个照存下来？

因为一旦按值快照，闭包最有价值的能力就没了：

1. **多个内部函数无法共享一个会变的状态。** 前面的计数器靠的就是 `count` 是同一个活的 cell，递增才看得见；如果定义时就把 `0` 拍成快照，`count += 1` 改的是函数自己的局部副本，别的函数看不到，计数器直接失效。
2. **闭包之间无法协作。** 一个典型模式是返回"一对函数"操作同一份隐藏状态：

   ```python
   def make_account():
       balance = 0
       def deposit(n):
           nonlocal balance
           balance += n
       def get_balance():
           return balance
       return deposit, get_balance
   ```

   `deposit` 存进去的钱，`get_balance` 必须看得到——这要求它们引用的是**同一个会变化的东西**，而不是各自拍下的 `0`。
3. **配置/依赖无法延迟生效。** 很多工厂函数先造好函数、稍后才配置环境变量，定义时拍照会把"还没配好"的值冻住。

换句话说，**按引用捕获（cell 共享）换来的是"函数之间能通过私有变量通信、持有可变状态"，代价是循环里那个反直觉的坑。** Python 觉得这笔买卖划算——毕竟循环里的坑有一行默认参数就能绕开，而快照方案丢掉的能力没法补。

> 顺带一段历史：词法嵌套作用域和闭包是 **PEP 227**（Python 2.1，2001 年）才加进来的。在那之前 Python 只有 L、G、B 三层——函数里不能直接引用外层函数的变量，想共享状态得把所有东西塞进类或默认参数里。而想在外层函数**修改**（不是读取）这个变量，得再等八年：`nonlocal` 是 **PEP 3104** 进 Python 3.0 的。原因也很朴素：Python 之父 Guido 早年担心"随意改外层变量"会让代码变得难追踪，一直拖到大家写装饰器和回调时实在绕不过去才松口。

### nonlocal 和 global：一句话分清

赋值在 Python 里默认意味着"新建/重建一个局部变量"，所以光读得到外层变量还不够，想**改**它必须显式声明：

- `nonlocal x`：**向外层函数找** `x`，找不到就报错（绝不顺手套到全局）。
- `global x`：**直接去模块全局**找 `x`，跳过所有嵌套函数层。

```python
x = "global"
def outer():
    x = "enclosing"
    def inner():
        nonlocal x       # 指向 outer 的 x
        x = "changed"
    inner()
    print(x)             # changed
outer()
```

一个判断诀窍：你想改的是**另一个函数里**的变量，用 `nonlocal`；你想改的是**文件顶层**的变量，用 `global`。

## 闭包最常见的三个用武之地

理解了机制，再回头看平时写的代码，闭包其实无处不在：

**1. 工厂函数：把配置"焊"进函数里**

```python
def make_printer(prefix):
    def printer(msg):
        print(f"[{prefix}] {msg}")
    return printer

log_info = make_printer("INFO")
log_err  = make_printer("ERROR")
log_info("started")     # [INFO] started
log_err("boom")         # [ERROR] boom
```

`prefix` 在 `make_printer` 返回后本该消失，但每个 `printer` 都用自己的 cell 留住了它。比起写个 `Printer` 类，这种写法轻得像没写代码。

**2. 带状态的装饰器**

装饰器本质上就是"吃一个函数、吐一个函数"的工厂，经常需要记住点什么（调用了几次、缓存了什么）：

```python
def count_calls(fn):
    n = 0
    def wrapper(*args, **kwargs):
        nonlocal n
        n += 1
        print(f"第 {n} 次调用 {fn.__name__}")
        return fn(*args, **kwargs)
    return wrapper
```

没有闭包，这种"包一层还能自己计数"的装饰器根本写不出来——你得退化成写类、把 `n` 放到 `self` 上。

**3. 回调：把"上下文"和"要做的事"一起传出去**

GUI 按钮、定时任务、`map`/`filter` 的 key 函数……当你把一个函数交给别人稍后调用时，闭包让它能"自带干粮"——把它需要的上下文变量一起带走，而调用方完全不用知道。这也是为什么早期 JavaScript 的事件绑定到处是闭包。

## 动手环节

### 实验 1：亲眼看到 cell 会变

```python
def outer():
    x = 0
    def inner():
        return x
    return inner

f = outer()
# f 抓住了 outer 的 x，从外面直接读那个 cell：
cell = f.__closure__[0]
print(cell.cell_contents)   # 0
```

再做一个更有意思的：让两个函数共享同一个 cell，从一个里面改、另一个里面看：

```python
def make_pair():
    value = 0
    def set_value(v):
        nonlocal value
        value = v
    def get_value():
        return value
    return set_value, get_value

s, g = make_pair()
print(g())          # 0
s(42)
print(g())          # 42 —— 改的是同一个 cell
print(g.__closure__[0] is s.__closure__[0])   # True，真的是同一个格子
```

### 实验 2：迟绑定，故意踩一次

把开头的代码改成"用默认参数立即绑定"和"用工厂函数"两个版本，各跑一遍，确认输出分别是什么。再试一个更隐蔽的变体：

```python
funcs = [lambda: n for n in range(3)]   # n 是模块级循环变量
print([f() for f in funcs])             # ?

funcs = [(lambda n: lambda: n)(n) for n in range(3)]  # 立即执行的工厂
print([f() for f in funcs])             # ?
```

<details><summary>点我看答案</summary>

第一段 `[2, 2, 2]`——老问题。第二段 `[0, 1, 2]`——那个 `(lambda n: lambda: n)(n)` 是个**立即调用的函数表达式**，每轮循环当场执行一次外层 lambda，参数 `n` 是该次调用的局部变量，于是内层 lambda 抓住的是三个独立的 cell。这就是方法一的紧凑写法。

</details>

### 自测问题

1. 下面这段为什么不报错？它打印什么？
   ```python
   def f():
       return g()
   def g():
       return 42
   print(f())
   ```
   （提示：词法作用域看的是定义时还是调用时？`g` 对 `f` 可见吗？）
2. 下面代码报错，问题出在哪？怎么用最小改动修好？
   ```python
   def make_adder():
       total = 0
       def add(x):
           total += x
           return total
       return add
   ```
3. 用闭包写一个 `make_stock_ticker()`，返回两个函数 `buy(n)` 和 `shares()`，分别买入和查询当前持股数。再用一个类实现同样的功能，对比两种写法各用了几行、状态藏在哪里。

---

## 收尾地图

**作用域与闭包速查：**

| 现象 | 原因 | 怎么办 |
|---|---|---|
| `UnboundLocalError` | 函数内有赋值，名字被判定为局部 | 赋值前别读；或 `global`/`nonlocal` |
| 循环造的函数全返回最后一个值 | 共享同一个 cell，迟绑定 | 工厂函数 / 默认参数 / `partial` |
| 函数里改不了外层变量 | 赋值默认创建局部变量 | 外层用 `nonlocal`，模块用 `global` |
| 外层函数返回后还能读到它的变量 | cell 被 `__closure__` 持有 | 闭包正常工作，不用管 |
| 想偷偷看闭包抓了啥 | 读 `__code__.co_freevars` 和 `__closure__` | 调试时偶尔用 |

**下一站去哪：**

- 闭包装状态、`wrapper(*args, **kwargs)` 转发参数——看懂这些，**装饰器**就只剩下语法了。去写一个 `@memoize` 缓存装饰器，你会发现它就是"用 dict 当 cell"。
- 理解了"代码 + 环境打包带走"，再去看 Python 的**函数式工具**：`functools.partial`、`operator` 模块里那些工厂，全在玩同一套。
- 想知道这一套在底层怎么实现的，去翻 CPython 的 `PyCellObject` 和字节码 `LOAD_DEREF` / `STORE_DEREF`——你会看到 `LOAD_FAST`（读局部）和 `LOAD_DEREF`（读自由变量）是两条不同的指令，这正是闭包存在的物理证据。
- 回头读[《Python 生成器：调用一个函数，函数体却一行都没执行》](/2026/08/17/python-generator.html)，你会发现生成器的栈帧里也挂着一份"被带走的环境"——它和闭包是同一种思想的两个分支：**让代码带着自己的现场，离开原地继续活着。**

闭包这个名字听起来玄乎，拆开来看一点都不神秘：**函数记住了它出生时周围的变量，而且记住的是变量本身，不是变量当时的值。** 一旦你把"记住变量，不记住值"这句话刻进脑子里，`[2,2,2]` 就不再是坑，而是必然。
