---
title: Python 对象的背包与紧身衣：从 __dict__ 到 __slots__
tags: [python]
categories: [Python]
---

凌晨两点，你写了两个小时的数据处理脚本，终于把一百万个用户对象加载进内存，准备跑分析。结果还没到第一个循环，内存就飙到 4GB，进程被系统 OOM 杀掉了。

更气人的是，对象本身就两个字段——`user_id` 和 `timestamp`。两个 int，一百万次，怎么算也就几十 MB。剩下的内存去哪了？

把锅甩给"Python 慢"之前，先问一个更具体的问题：**一个 Python 实例，到底把它的属性放在哪？**

答案藏在一个叫 `__dict__` 的字典里。而它的"反义词" `__slots__`，正是治这毛病的药。这篇文章会带你从"会用 slots"一路爬到"看懂 Python 对象模型的核心矛盾"，中间还有几个可以亲手跑的小实验。

<!--more-->

## L1：先让代码跑起来——`__slots__` 的最小用法

先看没有 `__slots__` 的世界：

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
p.z = 3              # 居然不报错？
p.x_coordinate = 1   # 拼错了也不报错，只是悄悄多了个属性
```

Python 不会拦你。只要你愿意，一个实例可以在运行时长出任意数量的新属性。加上 `__slots__` 之后：

```python
class Point:
    __slots__ = ('x', 'y')

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
p.z = 3              # AttributeError: 'Point' object has no attribute 'z'
p.x_coordinate = 1   # 拼写错误当场被抓
```

两个立刻能看到的好处：

1. **省内存**——每个实例不再背一个字典；
2. **防手滑**——拼错属性名立刻报错，而不是制造一个静默 bug。

到这里你已经会用了。但一个问题会自然冒出来：**为什么默认情况下，Python 要给每个实例配一个字典？这不是浪费吗？**

要回答这个问题，得先把 `__dict__` 看清楚。

## 台阶：`__dict__` 是什么？

普通实例的属性，其实都存在一个叫 `__dict__` 的字典里：

```python
class Person:
    def __init__(self):
        self.name = 'lxd'
        self.age = 25

p = Person()
print(p.__dict__)          # {'name': 'lxd', 'age': 25}
p.__dict__['city'] = 'Beijing'
print(p.city)              # Beijing
p.__dict__.pop('age')
print(hasattr(p, 'age'))   # False
```

`p.x = 1` 本质上就是 `p.__dict__['x'] = 1` 的语法糖（严格说是走 `type(p).__setattr__`，默认实现写入这个字典）。

不止实例有 `__dict__`，**类和模块也有**：

```python
class Dog:
    species = 'canine'
    def bark(self): return 'woof'

print(Dog.__dict__)
# mappingproxy({'species': 'canine', 'bark': <function ...>, ...})

import math
print(math.__dict__['pi'])  # 3.14159...
```

注意类的 `__dict__` 是个 `mappingproxy`——只读视图，防止你绕过类型系统乱改类内部结构。而模块的 `__dict__` 就是全局命名空间，`globals()` 返回的就是它。

### 属性查找的顺序

理解 `p.x` 到底按什么顺序找，是看懂 Python 对象模型的关键：

1. 先在 `type(p)` 的 MRO（方法解析顺序）里找**数据描述符**（有 `__set__` 的描述符，比如 `property`）；数据描述符优先于一切；
2. 再查 `p.__dict__`；
3. 再沿 MRO 查各个类的 `__dict__`（类属性、方法就在这）；
4. 都没有，调用 `__getattr__`（如果定义了），否则抛 `AttributeError`。

这能解释一个经典现象：

```python
class A:
    x = 'class'

a = A()
a.x = 'instance'
print(a.x)    # 'instance' —— 实例字典遮蔽了类属性
del a.x
print(a.x)    # 'class' —— 遮蔽移除，类属性重新可见
```

但 `property` 是数据描述符，**永远赢过实例字典**：

```python
class B:
    @property
    def x(self):
        return 'property'

b = B()
b.__dict__['x'] = 'instance'  # 你以为能塞进去？
print(b.x)                    # 仍然是 'property'
```

把 `__dict__` 想成每个 Python 实例随身背的一个**双肩包**：包里有什么属性完全自由，想塞什么塞什么——代价是包本身有重量。

## L2：`__slots__` 对背包做了什么

`__slots__` 等于告诉 Python："别给我发背包了，我就带这两样东西。"

```python
class Point:
    __slots__ = ('x', 'y')
```

底层发生了什么？Python 在**类创建时**为每个槽位生成一个叫 `member_descriptor` 的描述符：

```python
print(type(Point.x))         # <class 'member_descriptor'>
print(Point.x.__name__)      # 'x' —— 槽位名
print(Point.x.__objclass__)  # <class 'Point'> —— 声明它的类
```

访问 `p.x` 时，Python 通过这个描述符，直接从实例内存里该槽位的固定偏移位置读写——**没有字典查找，也没有字典本身的开销**（偏移量记录在描述符对应的 C 结构体里，Python 层没有直接暴露）。

可以直接验证：

```python
p = Point(1, 2)
print(hasattr(p, '__dict__'))   # False，背包没了
print(vars(Point))              # 但类字典里有 x 和 y 两个描述符
```

这时候内存差距有多大？`sys.getsizeof` 对带 dict 的实例不包含字典本身，真实差距用 `tracemalloc` 看更直观：

```python
import tracemalloc

class WithDict:
    def __init__(self):
        self.x = 1
        self.y = 2

class WithSlots:
    __slots__ = ('x', 'y')
    def __init__(self):
        self.x = 1
        self.y = 2

tracemalloc.start()
nodes = [WithSlots() for _ in range(1_000_000)]
current, peak = tracemalloc.get_traced_memory()
print(f"slots: {current / 1024 / 1024:.1f} MB")

tracemalloc.stop()
tracemalloc.start()
nodes = [WithDict() for _ in range(1_000_000)]
current, peak = tracemalloc.get_traced_memory()
print(f"dict:  {current / 1024 / 1024:.1f} MB")
```

在 CPython 3.11/3.13 上实测，上面这个两字段的例子，带 dict 的版本大约是 slots 版本的 **1.7 倍**（约 92 MB vs 54 MB）。这个数字随字段个数和 CPython 版本浮动——早年没有 key-sharing 优化时能接近 3 倍，新版本字典更紧凑后已明显缩小，但 slots 始终是最省的那个。这就是开头那个 OOM 故事的答案。

> **先猜再看**：如果我在类创建之后写 `Point.__slots__ = ('x', 'y', 'z')`，新属性 `z` 会生效吗？
>
> <details markdown="1"><summary>答案</summary>
>
> 不会。`__slots__` 必须在**类创建时**存在，CPython 在那个时刻就根据它固定了实例的内存布局。事后赋值只是加了个普通类属性，实例照样没有 `z` 槽位，访问仍然 `AttributeError`。这也是为什么 `@dataclass(slots=True)` 要"重新创建一个类"而不是简单地加个属性——它没法在类已存在后补布局。
>
> </details>

## 一个反直觉的细节：Key-Sharing Dict

你可能会想：每个实例都背一个字典，那 CPython 难道不会优化吗？

事实上 CPython 3.3+ 确实做了一个叫 **key-sharing dictionary** 的优化：同一个类产生的实例，**共享键表**，各自只存值。

```python
class P:
    def __init__(self):
        self.x = 1
        self.y = 2

a, b = P(), P()
# 内部：键表（x, y 的布局）a 和 b 共享，值数组各自持有
```

这把"普通类多实例"的内存开销压下来不少。所以 `__slots__` 的优势比早年小了——但它仍然是最省的方案，因为它连值数组的间接层都省了。

**坑点**：以下操作会打破共享，让实例退化成独立完整字典：

- 实例的属性插入顺序和兄弟实例不同（条件分支里才设置某属性）；
- 运行中删除属性（`del p.x`）；
- 直接替换 `p.__dict__`。

这是个很生活化的设计：快递站里，同小区的包裹用统一格式的面单（共享键表），但只要有一个包裹写了个奇奇怪怪的字段，就得换一张全新面单（退化）。

## L3：本质——这是一笔交易，不是魔法

去掉所有术语，一句话：

> **`__slots__` 和 `__dict__` 是 Python 在"灵活性"和"性能"之间的两个档位。`__dict__` 给你一个自由但有重量的背包，`__slots__` 给你一件缝死口袋的紧身衣。**

这个设计不是拍脑袋的。我们来追问一层：**为什么 Python 不直接给所有类都用 slots 那种紧凑布局，需要动态性时再特殊处理？**

反过来想就明白了：

- 如果默认紧凑布局，那"给实例动态加方法"、"猴子补丁"、"ORM 延迟加载字段"、装饰器给对象挂状态——这些 Python 程序员习以为常的能力全部要特判。
- 字典是一种"通用答案"：不管你将来有多少属性、什么类型、什么时候加，它都能装。代价是固定的开销。
- 槽位是一种"提前承诺"：你告诉解释器这辈子就这几个字段，它用 C 结构体的方式给你布局，省下来的是字典的哈希表开销和指针间接访问。

换句话说，**Python 默认选了"对未知开放"，把"我知道我要什么"的优化留给显式声明的人**。这是一门主打"开发者时间比 CPU 时间贵"的语言该有的取舍。

### 被否决的方案：混合模式

也许你会想，能不能让类先紧凑布局，真要动态加属性时再偷偷给它挂个字典？

事实上可以——把 `'__dict__'` 加进 `__slots__`：

```python
class Hybrid:
    __slots__ = ('x', 'y', '__dict__')  # x、y 走槽位，其他走字典

h = Hybrid()
h.x = 1           # 槽位
h.extra = 999     # 字典
```

但这就像给紧身衣再缝一个背包——省内存的好处主要来自那两个固定字段，而 `__dict__` 的开销只要用了就跑不掉。所以它只在"少量固定字段 + 偶发动态属性"这种特定场景下值得。

### 另一个被否决的方案：`NamedTuple` / `struct`

如果对象完全不可变、字段固定，比 `__slots__` 更省的方案是 `typing.NamedTuple`，甚至 `numpy` 结构化数组 / `array` 模块——它们把所有数据塞进一段连续内存，连对象头都省了。但代价是失去了所有 Python 对象的能力（不能随便加方法、不能继承行为）。

每一档都对应一种"你愿意放弃多少"。

## 踩坑清单：用了 `__slots__` 之后会撞到什么

### 1. 继承会"失效"

```python
class Base:
    __slots__ = ('a',)

class Child(Base):
    pass    # 没定义 __slots__

c = Child()
c.anything = 1   # 不报错！子类又长出 __dict__ 了
```

子类必须也显式定义 `__slots__`（哪怕是空元组），紧凑布局才会延续。

### 2. 多继承布局冲突

```python
class A:
    __slots__ = ('a',)
class B:
    __slots__ = ('b',)

class C(A, B):  # TypeError: multiple bases have instance lay-out conflict
    pass
```

两个父类都有非空槽位，CPython 不知道该怎么合并内存布局。解决方案是把不需要槽位的 mixin 写成 `__slots__ = ()`：

```python
class Mixin:
    __slots__ = ()   # 空槽，不占布局，不冲突
```

### 3. 弱引用失效

```python
import weakref

class Node:
    __slots__ = ('value',)

n = Node()
weakref.ref(n)  # TypeError: cannot create weak reference
```

需要弱引用时，显式加上 `'__weakref__'`：

```python
class Node:
    __slots__ = ('value', '__weakref__')
```

### 4. pickle / copy 行为变化

旧版 pickle 协议依赖 `__dict__` 还原状态。定义 `__getstate__` / `__setstate__` 最稳妥：

```python
class Point:
    __slots__ = ('x', 'y')

    def __getstate__(self):
        return (self.x, self.y)

    def __setstate__(self, state):
        self.x, self.y = state
```

### 5. 不能和类属性重名

```python
class Bad:
    __slots__ = ('x',)
    x = 1   # ValueError: 'x' in __slots__ conflicts with class variable
```

槽位本身在类上以描述符形式存在，再放一个同名类属性就打架了。

## 动手环节：三个可以立刻跑的实验

### 实验 1：预测输出

盖住答案，先猜下面这段打印什么：

```python
class A:
    __slots__ = ('x',)

a = A()
a.x = 1
print('A has __dict__?', hasattr(a, '__dict__'))

class B(A):
    __slots__ = ('y',)

b = B()
b.x = 10
b.y = 20
try:
    b.z = 30
except AttributeError as e:
    print('setting z failed:', e)
print('b.x, b.y =', b.x, b.y)
```

<details markdown="1"><summary>答案</summary>

输出：

```
A has __dict__? False
setting z failed: 'B' object has no attribute 'z'
b.x, b.y = 10 20
```

A 的实例没有 `__dict__`。B 继承自 A 并定义了自己的 `__slots__ = ('y',)`，同时**也继承了 A 的 x 槽位**，所以 `x`、`y` 都是槽位；而 `z` 没在任何 slots 里，赋值当场失败。

如果你下意识以为 `b.z = 30` 总能成功，说明潜意识里还觉得"子类总归能动态加属性"——这正是 `__slots__` 在帮你纠正的错觉。注意：只要整条继承链上**所有**类都老老实实写了 `__slots__`，背包就不会回来；任何一个祖先漏写，子类实例就会重新长出 `__dict__`。

</details>

### 实验 2：实测一百万对象的内存

保存为 `memtest.py`：

```python
import tracemalloc

class WithDict:
    def __init__(self):
        self.a = 1; self.b = 2; self.c = 3

class WithSlots:
    __slots__ = ('a', 'b', 'c')
    def __init__(self):
        self.a = 1; self.b = 2; self.c = 3

for cls in (WithSlots, WithDict):
    tracemalloc.start()
    objs = [cls() for _ in range(1_000_000)]
    current, _ = tracemalloc.get_traced_memory()
    print(f"{cls.__name__:10s}: {current/1024/1024:6.1f} MB")
    tracemalloc.stop()
```

在 CPython 3.11/3.13（Linux）上的实测量级：

```text
WithSlots :   64.0 MB 左右
WithDict  :  100.0 MB 左右
```

具体数字随 CPython 版本和位数浮动（老版本的 dict 实例能到 200 MB 上下），但 slots 明显更省这个结论不变。跑一次，你对"省内存"这件事就有身体记忆了。

### 实验 3：打破 key-sharing

```python
class P:
    def __init__(self, mode):
        self.x = 1
        self.y = 2
        if mode:
            self.z = 3   # 只有一半实例有 z

a = P(False)
b = P(True)
# 思考：a 和 b 还共享键表吗？
```

<details markdown="1"><summary>怎么验证</summary>

CPython 没有暴露直接的查询 API，但可以通过观察内存变化间接验证：用 `tracemalloc` 对比"所有实例属性顺序一致"和"一半实例多一个字段"两组的总内存，后者会明显变大——因为它们退化回了各自独立的字典。

</details>

## 自测时间

能答上这三个问题，说明你真的懂了：

1. 一个定义了 `__slots__ = ('x', 'y')` 的类，它的实例有没有 `__dict__`？它的**类**有没有 `__dict__`？
2. 为什么 `@dataclass(slots=True)` 需要"重建一个类"，而不是直接给原类加 `__slots__`？
3. 子类定义 `__slots__ = ()`（空元组）有什么用？

<details markdown="1"><summary>参考答案</summary>

1. 实例没有 `__dict__`；但**类仍然有**——类的 `__dict__` 是个 mappingproxy，装着方法、类属性，以及 x/y 的 member_descriptor。两者不要混淆。
2. 因为 `__slots__` 只在类创建时被 CPython 读取，用来决定实例的内存布局；类创建完之后再赋 `__slots__` 只是个普通属性，不会改变布局。dataclass 装饰器只能"造一个新类"来注入它。
3. 空元组既表示"我自己不新增槽位"，又能阻止子类自动长出 `__dict__`。这是 mixin 的标准写法——它告诉 Python "我也是紧凑布局的一员，请别给我发背包"。

</details>

## 收尾地图：什么时候用什么

| 场景 | 推荐方案 | 理由 |
|---|---|---|
| 普通业务对象，字段会变 | 普通类 | 灵活性最重要，那点内存不值得省 |
| 数据量大、字段固定的数据对象 | `@dataclass(slots=True)` | 省内存，还白送 `__init__`/`__repr__`/`__eq__` |
| 不可变的值对象 / 记录 | `typing.NamedTuple` | 更省，还能解包 |
| 百万级数值记录，要做向量化运算 | `numpy` 结构化数组 / `array` | 连续内存，连对象头都省了 |
| 高性能序列化（API / IPC） | `msgspec.Struct`、`attrs` | slots + 自动序列化 |
| 需要动态加属性、猴子补丁 | 普通类或 `__slots__ = (..., '__dict__')` | 背包不能丢 |
| 写 mixin 给 slotted 类用 | `__slots__ = ()` | 不引发多继承布局冲突 |

**下一站**：搞懂 `__dict__` / `__slots__` 之后，下面这些概念会变得"原来如此"——

- **描述符协议**：`property`、`classmethod`、`member_descriptor` 都是同一套机制，理解了它你就理解了 slots 底层、ORM 字段、`@property` 的共同骨架；
- **MRO（方法解析顺序）**：类的 `__dict__` 是怎么沿着继承链被查找的，C3 线性化解决了什么；
- **弱引用与 `__weakref__`**：为什么 slotted 类要显式声明它，弱引用表到底存在哪；
- **CPython 对象内存布局**：`PyObject_HEAD`、`ob_size`、`tp_itemsize`——你会真的看懂"槽位偏移量"是什么意思。

把 `__dict__` 想成背包，`__slots__` 想成紧身衣，再看任何 Python 对象，你都能一眼判断它今天背了什么、放弃了什么。

这就是 Python 对象模型最朴素的矛盾：**自由是有重量的，而承诺可以换来速度。**
