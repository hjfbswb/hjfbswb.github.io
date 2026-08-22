---
title: Node 程序的架构：模块是单例，依赖是棵树
tags: [nodejs, javascript, 模块, require, 依赖注入, 架构]
categories: [JavaScript]
last_modified_at: 2026-08-22 18:18:36 +0800
---

先准备三个小文件，跑之前在心里预测输出：

```js
// db.js —— 一个计数器
let count = 0;
module.exports = { next: () => ++count };
```

```js
// a.js —— 调用了两次 next
const db = require("./db");
db.next();
db.next();
```

```js
// b.js —— 也 require 了 db，然后打印
const db = require("./db");
console.log(db.next());   // 你猜是几？
```

再加一条命令把它们串起来：

```sh
node -e "require('./a'); require('./b')"
```

很多人预测输出 1：b.js 不是"又 require 了一次 db.js"吗？新文件新气象，计数器应该从 0 开始。实际输出是：

```
3
```

同样的 `require("./db")`，第二次没有重新执行——a.js 和 b.js 拿到的是**同一个**对象，计数器在它们之间是共享的，零新拷贝。

这看起来是个实现细节。但它其实是 Node 程序整个"架构"的地基。

"架构"这个词听着很大：目录怎么分、要不要分层、上不上微服务。但在 Node 里，最底层的架构就一句话——**从入口文件开始，require 一路织下去，织出一棵树**。谁先被创建、谁向谁要东西，全都定在这棵树里。这篇文章就沿着这棵树往上爬。

<!--more-->

## L1：先会用——从一段脚本到一个有骨架的程序

你在第一层，目标是把一个"散装"脚本拆成有骨架的程序，并且让它跑起来。

如果你的项目只有几个文件、几百行，你大概没感觉到"架构"的存在——一切都在 server.js 里摊着，路由、业务、数据混成一锅。等它涨到几千行，加一个功能要滚三屏，改一处数据格式要牵连五处，这时候"分层的骨架"就是刚需了。

最小的骨架长这样，箭头只有一个方向：

```
app.js                    入口：组装各层，启动服务器
  └─ routes/todoRoutes.js       路由：把 HTTP 请求翻译成业务调用
       └─ services/todoService.js    服务：业务规则
            └─ data/todoStore.js         数据：只管存和取
```

每层一句话职责，上层 require 下层，**下层永远不知道上层的存在**。一个能跑的待办清单 API，四个文件加起来不到 40 行：

```js
// app.js —— 入口
const http = require("http");
const todoRoutes = require("./routes/todoRoutes");

http.createServer(todoRoutes.handle).listen(8080, () =>
  console.log("运行中：curl localhost:8080/todos")
);
```

```js
// routes/todoRoutes.js —— 路由：把 HTTP 翻译成业务调用
const todoService = require("../services/todoService");

function handle(req, res) {
  res.setHeader("Content-Type", "application/json; charset=utf-8");
  const parts = req.url.split("/");          // ["", "todos"] 或 ["", "add", "milk"]
  if (parts[1] === "todos") return res.end(JSON.stringify(todoService.list()));
  if (parts[1] === "add") {
    todoService.add(decodeURIComponent(parts[2]));
    return res.end("ok");
  }
  res.statusCode = 404;
  res.end("not found");
}
module.exports = { handle };
```

```js
// services/todoService.js —— 服务：业务规则（空任务不许加）
const todoStore = require("../data/todoStore");

function add(text) {
  if (!text) throw new Error("空任务是耍流氓");
  todoStore.add(text);
}
function list() { return todoStore.list(); }
module.exports = { add, list };
```

```js
// data/todoStore.js —— 数据：只管存和取
const items = [];
module.exports = {
  list: () => items,
  add: (text) => items.push(text),
};
```

`node app.js`，然后依次敲：

```sh
curl localhost:8080/todos
curl localhost:8080/add/milk
curl localhost:8080/todos
```

预期输出依次是 `[]`、`ok`、`["milk"]`。数据存在内存数组里，重启就没了——先别管，那是后话（将来接数据库时，只有 `todoStore.js` 需要改，这正是分层的红利）。

到这里你会拆了。**但这棵树还欠一个解释**：你说 store 被 service require，那如果明天路由层也想直接读 store，它拿到的是和 service 共享的那一份，还是全新的一份？模块和模块之间到底是怎么"连"起来的？开头那个"输出 3"的谜，答案就在下一步。

## L2：懂原理——require 的四步：查、装、包、跑、存

你在第二层，目标是看清 `require("./db")` 这一行到底干了什么。

一句话版本：**它把模块代码执行一遍，然后记住结果，下次直接还你同一个结果。**拆开是五步：

1. **查**：把 `"./db"` 解析成磁盘上的绝对路径。不带 `./` 的（比如 `require("express")`）沿着 `node_modules` 一级级向上找——这条地图在[包管理那篇](/2026/08/22/nodejs-package-install.html)里已经画过；
2. **装**：读出文件内容；
3. **包**：把源码包进一层函数：

   ```js
   function (exports, require, module, __filename, __dirname) {
     // 你的源码在这里
   }
   ```

   这一行很关键：模块里能直接用的 `require`、`module`、`__dirname` **不是魔法全局变量，而是这个函数的参数**。顺带解开一个经典坑：在模块里写 `exports = { ... }` 不生效，因为那只是把"参数 exports"换个指向，真正的 `module.exports` 纹丝没动——得写 `module.exports = ...`；
4. **跑**：执行这层函数，模块代码正式出生；
5. **存**：把 `module.exports` 的结果写进 `require.cache`，键是第一步解析出的绝对路径。**下一次 require 同一个模块，前四步全部跳过，直接从缓存里把上次的结果原样交给你。**

所以"输出 3"不是 bug，是第 5 步的必然。同样的 `require("./db")`，从第二次起你拿到的永远是第一份。动手验证，先预测再运行——下面这段，`loud.js` 顶层的日志会打印几次？

```js
// loud.js
console.log("loud.js 被执行了！");
module.exports = 42;
```

```js
// main.js
require("./loud");
require("./loud");
```

运行 `node main.js`，输出：

```
loud.js 被执行了！
```

只有一次。第一个 `require` 把模块跑了一遍并缓存；第二个 `require` 命中缓存，一行代码都没执行。你可以顺手看一眼那本缓存账：`console.log(require.cache)`——入口 main.js 自己也在里面占着一个位置，键都是绝对路径。

到这里"单例"已经成立了。**但还有一个更刁钻的推论**：模块只出生一次，那出生的**顺序**就决定了命运。依赖树从入口开始深度优先执行，谁先被 require 谁先出生。如果两个模块互相 require，顺序就会撞车。先预测再看答案——下面这个程序，b.js 打印出来的 `a.name` 是什么？

```js
// a.js
const b = require("./b");
module.exports = { name: "a" };
```

```js
// b.js
const a = require("./a");
console.log("b.js 看到 a.name =", a.name);
module.exports = { name: "b" };
```

```js
// main.js
require("./a");
```

运行 `node main.js`，输出：

```
b.js 看到 a.name = undefined
Warning: Accessing non-existent property 'name' of module exports inside circular dependency
```

b.js 看到的不是"没有 a"，而是一个**空对象**——a 还没执行到 `module.exports = ...` 那一行，b 就把它的半成品拿走了。那条 Warning 是 Node 在替你报警："这里是个环，你拿到的是还没完成的东西。"如果你觉得"两个模块互相 require 没什么大不了"，现在你知道代价了：顺序失控的那一刻，谁拿谁都可能是半成品。

到这里，"模块怎么连、顺序怎么定"都清楚了。**但一个更根本的问题还悬着**：缓存单例这个设计，是不是太糙了？每次 require 都重新执行一遍、各拿各的副本，不是更符合直觉吗？为什么非得是"只出生一次"？爬最后一层。

## L3：想得透——为什么必须是单例；架构的本质

你在第三层，最后一层。这一层不谈命令也不谈机制，谈"为什么非这样不可"。

认真对待刚才那个"改进方案"：require 不缓存，每次都重新执行。失败场景排着队来：

- **状态悄悄分裂**：a.js 和 b.js 各 require 一份 store，各拿一个数组。你在 a 里写入的数据，b 里永远看不到——你以为在写同一个库，其实在写两个库。而且没有任何报错，排查时连怀疑对象都没有；
- **初始化重复燃烧**：依赖树一深，"连数据库、读配置"这类顶层代码被反复执行几十遍，启动慢成灾难。缓存单例保证"一份初始化只跑一次"，反而是省事的设计。

所以"单例"不是偷懒，是刻意选择：**共享状态必须只有一份，否则"共享"这个词本身就不成立。**那句"输出 3"不是细节，是整棵树能成立的契约。

再往前追一层历史。为什么模块非要"包一层函数"这么绕？因为 Node 之前的浏览器时代，JS 根本没有模块系统——所有 `<script>` 标签共享同一个全局命名空间，脚本之间靠 `window` 上的全局变量打招呼，谁都能改谁的变量，加载顺序错一点全盘崩。2009 年 CommonJS（当时叫 ServerJS）诞生，核心发明正是 L2 那第 3 步：**给每个文件包一层私有作用域，只通过 exports 显式交出该交的东西。**你现在用着的一切 require 语法，都是当年给"全局混乱"建的一堵墙。

但单例也有它的洞，而且正好打在测试上：**想换掉共享的东西，很难。**store 是全程序唯一的数组，你要测试 service 的"空任务不许加"规则，就得真的去操作那个唯一数组——想塞一个假的、带历史记录的 store，塞不进去，因为 service 已经通过 require 把它写死了。

补丁叫**依赖注入**：把"模块自己 require 依赖"改成"由调用方把依赖递给它"。最轻量的一种写法是工厂函数——模块导出一个函数，而不是一个对象：

```js
// store.js —— 导出的是"造 store 的函数"，不是 store 本身
module.exports = () => ({
  items: [],
  add(text) { this.items.push(text); },
  list() { return this.items; },
});
```

```js
// main.js
const makeStore = require("./store");
const s1 = makeStore();   // 每个调用方拿一份全新的
const s2 = makeStore();
s1.add("milk");
console.log(s1.list(), s2.list());   // [ 'milk' ] [] —— 互不干扰
```

`require` 缓存管不着工厂函数的返回值——模块本身仍是单例（`makeStore` 只有一份），但每次调用都给你**新的一份 store**。生产环境只调一次、全程序共享；测试时多调几次、各塞各的假数据。单例的"唯一性"你保住了，可替换性也拿回来了。

本质段来了。去掉所有术语，一个 Node 程序的架构到底是什么？

> **一个 Node 程序，就是一棵由 require 织出来的树：每个模块只出生一次，谁被谁 require，谁就站在谁的下游。架构不是画在文档里的一张图——它早就编码在这棵树里了：谁先被创建、数据往哪流，顺着 require 的箭头走一遍，就是全貌。**

检验标准：你能不借用 require、模块、依赖注入这些词，跟同事讲清"为什么测试时要塞一个假的 store"吗？可以这样说——树上每个节点只认下游递给它的东西，所以想换掉最底层的数据库，只需在"递东西"那一步换，其他地方纹丝不动。这就是分层和依赖注入同一件事的两面：**上层决定要什么，下层决定从哪来。**

一句话总结这层的权衡：**单例缓存用"换实现难一点"的小麻烦，换"状态全局唯一、初始化只跑一次"的大确定**；依赖注入再把"可替换"补回来，代价是多写一层传参。[入门那篇](/2026/08/22/nodejs-intro.html)讲的是运行时怎么扛住一万个连接，这篇讲的是程序自己怎么长——两件事，同一个"树"的思路：连接是事件循环里的树，代码是 require 织成的树。

## 收尾：一张决策表

什么时候拆模块、什么时候别拆、什么时候打破单例：

| 你的处境 | 做法 | 为什么 |
|---|---|---|
| 代码只有一两百行 | 别拆，一个文件写完 | 拆是成本，小项目撑不起结构 |
| 一个状态要被全程序共享 | 直接 require 单例 | 缓存保证全局只有一份，天然符合预期 |
| 测试时想换假实现（假数据库） | 依赖注入 / 工厂函数 | 把"自己找依赖"改成"别人递给你"，才换得掉 |
| 模块代码变长、职责变多 | 按"入口→路由→服务→数据"分层 | 上层要什么、下层从哪来，各管一段 |
| 两个模块互相 require | 重构：把公共部分抽到第三层 | 循环依赖是初始化顺序失控的警报 |
| 文件多到失控、想自动管理 | 看 npm workspaces / monorepo | require 树变大后，需要更显式的边界 |

三个自测问题，能答上说明真懂了：

1. 同一个模块 require 两次，第二次还会执行顶层代码吗？为什么？（答案在 L2 的"存"这一步：命中 `require.cache` 直接返回，代码不再执行。）
2. 为什么循环依赖里，后加载的一方拿到的是半成品？（答案在 L2 推论：依赖树深度优先执行，谁先被 require 谁先出生；对方还没执行完，你拿到的自然是没写完的。）
3. 测试时想给 service 塞一个假 store，直接 require 为什么换不掉？怎么改？（答案在 L3 的依赖注入：单例把依赖写死了，用工厂函数把依赖变成"递进来的参数"。）

下一站按顺序走会更顺：先用 `async/await` 给这篇的 todo 程序接数据库、处理错误——错误会顺着依赖树一层层往上抛，最后停在谁手里，就是下一篇的主角；然后上手 Express——你会发现它的"中间件"也是一棵树，一层套一层，长得很像你这棵 require 树。到时候再回头看"模块是单例，依赖是棵树"，就全串起来了。

```json
{
  "上一站": "Node.js 入门：单线程的服务器 / Node 的包管理",
  "下一站": "async/await 与错误处理 -> Express 中间件"
}
```
