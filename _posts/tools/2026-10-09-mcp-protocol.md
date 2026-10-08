---
title: 模型的手长在外面：MCP 协议深读
tags: [mcp, ai, agent, claude-code, 工具链]
categories: [AI 工具]
---

过去一年半，Anthropic 没有为"接上 GitHub"发过一次版。

这话值得多看两眼。你（或者你用的任何 AI 客户端）在这个期间陆续获得过操作 GitHub、查数据库、开浏览器的能力，每一样都不是客户端厂商内置的——它们全是第三方写的"MCP 服务器"，即插即用，客户端一行代码没改。客户端不认识这些工具中的任何一个，它只认识一张"插座"。模型的手为什么能做到长在外面、随时插拔？这张插座上刻着什么？

这正是欠了一年的一笔账。[上下文管理篇](/2026/09/12/agent-context-management.html)结尾原话："按需把说明书递给模型"，正是 MCP（Model Context Protocol，名字里就带着 Context）要解决的问题。它同时是插件系列埋下的另一笔债——两条线，下一篇合流再还。"而更早的[harness 篇](/2026/09/11/deepseek-agent-harness.html)结尾也点了名：工具表硬编码在 Python 里显然不体面——"工具的插拔协议"（MCP）是插件系列埋过的另一个伏笔，两篇伏笔，迟早要还。今天合流还债：MCP 是插件系列那句"够写三篇的话题"里 [插件开发篇](/2026/09/07/claude-code-plugin-dev.html)只推开一条缝的那扇门。

## L1：先跑起来——50 行，一个行业标准

你现在在第一层：不背概念，亲手造一个 MCP 服务器，再让客户端连上它。

动手前先说清它站在哪。MCP 不改变模型的行为——模型永远只做它唯一会的事：收到文本，续写文本（这是 harness 篇的地基）。MCP 活在模型外面，管的是 harness 与外部世界之间那一段：

```text
模型 ──(function calling：开单/收单)──> 你的 harness
                                          │
                                          ├──(MCP：发现/调用)──> MCP 服务器 A（天气）
                                          ├──(MCP)──────────> MCP 服务器 B（数据库）
                                          └──(MCP)──────────> MCP 服务器 C（浏览器）
```

协议本身基于 [JSON-RPC 2.0](https://www.jsonrpc.org/specification)——比 REST 还老的老协议，整个协议的全部消息就是三种形状：带 id 的请求、对着 id 回的响应、不带 id 的通知。下面这个服务器只用标准库，35 行，实现三个方法：

```python
# server.py —— 最小 MCP 服务器：零依赖，只讲协议本身
# 规范依据：stdio 传输 = stdin/stdout 上的换行分隔 JSON-RPC 2.0
import sys, json

WEATHER = {"北京": "晴，26℃，西北风二级，湿度低", "上海": "小雨，23℃，湿度高"}

def handle(req):
    """返回 (result, error)，二者必居其一。"""
    m = req["method"]
    if m == "initialize":
        return ({"protocolVersion": "2025-06-18",
                 "capabilities": {"tools": {"listChanged": False}},
                 "serverInfo": {"name": "weather", "version": "0.1"}}, None)
    if m == "tools/list":
        return ({"tools": [{
            "name": "get_weather",
            "description": "查询指定城市今天的天气，决定晒被子前先问它",
            "inputSchema": {
                "type": "object",
                "properties": {"city": {"type": "string",
                                        "description": "城市名，如 北京"}},
                "required": ["city"]}}]}, None)
    if m == "tools/call":
        city = req["params"]["arguments"]["city"]
        report = WEATHER.get(city, f"{city}：暂无数据")
        return ({"content": [{"type": "text", "text": report}],
                 "isError": False}, None)
    return (None, {"code": -32601, "message": f"Method not found: {m}"})

for line in sys.stdin:                      # 每行一条 JSON-RPC 消息
    req = json.loads(line)
    if "id" not in req:                     # 通知（notification）没有 id，
        continue                            # 按 JSON-RPC 约定不回复
    result, err = handle(req)
    out = {"jsonrpc": "2.0", "id": req.get("id")}
    if err is None:
        out["result"] = result
    else:
        out["error"] = err
    sys.stdout.write(json.dumps(out, ensure_ascii=False) + "\n")
    sys.stdout.flush()
```

三个方法各管一件事：`initialize` 报家门（协议版本、有什么能力），`tools/list` 交出工具说明书，`tools/call` 接单干活。传输层就是 stdio——stdin 进 stdout 出，一行一条 JSON。

客户端也手写，20 行，走完一次完整会话：

```python
# client.py —— 拉起服务器子进程，走完一次完整会话
import subprocess, json, itertools

p = subprocess.Popen(["python3", "server.py"], stdin=subprocess.PIPE,
                     stdout=subprocess.PIPE, text=True)
replies = (json.loads(p.stdout.readline()) for _ in itertools.count())
def call(rid, method, params=None):
    req = {"jsonrpc": "2.0", "id": rid, "method": method, "params": params or {}}
    p.stdin.write(json.dumps(req) + "\n"); p.stdin.flush()
    return next(replies)

print(call(1, "initialize"))                          # ① 握手
p.stdin.write('{"jsonrpc":"2.0","method":"notifications/initialized"}\n')
p.stdin.flush()                                       # ② 客户端通知，无 id 无回复
print(call(2, "tools/list"))                          # ③ 要工具说明书
print(call(3, "tools/call",
           {"name": "get_weather", "arguments": {"city": "北京"}}))
print(call(4, "tools/call",
           {"name": "get_weather", "arguments": {"city": "南京"}}))
p.stdin.close(); p.wait()                             # 关 stdin = 挂电话
```

**先做个预测再往下看**：客户端总共发了 5 条消息（2 请求 + 1 通知 + 2 请求），会收到几条回复？

真实运行的结果（原样照贴，Python 字典格式）：

```text
{'jsonrpc': '2.0', 'id': 1, 'result': {'protocolVersion': '2025-06-18', 'capabilities': {'tools': {'listChanged': False}}, 'serverInfo': {'name': 'weather', 'version': '0.1'}}}
{'jsonrpc': '2.0', 'id': 2, 'result': {'tools': [{'name': 'get_weather', 'description': '查询指定城市今天的天气，决定晒被子前先问它', 'inputSchema': {'type': 'object', 'properties': {'city': {'type': 'string', 'description': '城市名，如 北京'}}, 'required': ['city']}}]}}
{'jsonrpc': '2.0', 'id': 3, 'result': {'content': [{'type': 'text', 'text': '晴，26℃，西北风二级，湿度低'}], 'isError': False}}
{'jsonrpc': '2.0', 'id': 4, 'result': {'content': [{'type': 'text', 'text': '南京：暂无数据'}], 'isError': False}}
```

5 发 4 回。少的那条不是丢了，是通知本来就约定不回复——所以那个"挂电话"是 stdio 传输的正式关机流程（规范原话：stdin 读到 EOF 是"主要的、也是唯一可移植的优雅关机信号"）。

不瞒你说，这 4 行输出是我**第二遍**才跑出来的。第一遍的 `server.py` 没写那两行 `continue`，真实翻车实录如下：

```text
{'jsonrpc': '2.0', 'id': 1, 'result': {'protocolVersion': '2025-06-18', ...}}
{'jsonrpc': '2.0', 'id': None, 'error': {'code': -32601, 'message': 'Method not found: notifications/initialized'}}
{'jsonrpc': '2.0', 'id': 2, 'result': {'tools': [...]}}
{'jsonrpc': '2.0', 'id': 3, 'result': {'content': [{'type': 'text', 'text': '晴，26℃……'}], ...}}
```

服务器好心地给那条通知回了个错误。看起来无伤大雅——但客户端是按"收到一行 = 上一条请求的回执"来对账的，多出来的一行使**所有应答错位一格**：`tools/list` 拿到了别人的回执，最后一个 `tools/call` 永远等不到属于自己的那行，整个会话死锁。教训比错误本身值钱：**JSON-RPC 的通知不是"可以不回"，是"必须不回"**——协议里每一个"不出声"都是设计好的，你补的那声"收到"才是事故。

最后一步，把这个真服务器插进 Claude Code。项目根目录放一个 `.mcp.json`：

```json
{
  "mcpServers": {
    "weather": {
      "command": "python3",
      "args": ["/path/to/server.py"]
    }
  }
}
```

重启会话，你就能看到 [插件开发篇](/2026/09/07/claude-code-plugin-dev.html)预告过的那个名字出现：`mcp__weather__get_weather`——`mcp__服务器名__工具名`，服务器名来自 `.mcp.json` 的键，工具名来自 `tools/list` 的返回。当年埋的伏笔，对上了。

到这里你已经亲手实现了一个 MCP 服务器——不到 50 行，零依赖。这本身就该让你停下来想：一个三家人同时背书的行业标准，凭什么这么小？

## L2：拆开看——说明书搬了个家

第二层，拆协议的骨架。

第一个要看清的层次是**角色**。规范（[Architecture](https://modelcontextprotocol.io/specification/2026-07-28/architecture)）定义的是 host–client–server 三层：host 是那个 AI 应用本体（Claude Code 就是一个 host），它为**每一个** MCP 服务器创建一个专属的 client 实例，client 与 server 严格一对一。也就是说，上面图里画的"三根线"，在 host 内部其实是三个互相隔离的管道。

为什么要隔离得这么死？规范的设计原则第 3 条写得毫不客气：**"Servers should not be able to read the whole conversation, nor see into other servers"**——服务器不该读到完整对话，也不该看见别的服务器。每次调用，host 只把这一次需要的上下文递给对应的 server。你想给模型接一双手，但手不该长着眼睛。记住这条，本文最后谈安全时要用。

第二个要看的，是 `tools/list` 那行返回值里藏着的一笔旧账。把它和你 60 行 harness 里硬编码的那个 Python 列表摆在一起看：

```python
# harness 篇：工具说明书硬编码在源代码里
TOOLS = [{"type": "function",
          "function": {"name": "get_weather", "description": "…",
                       "parameters": {"type": "object", "properties": {"city": …}}}}]

# MCP：说明书长在服务器上，运行时拉取
{"tools": [{"name": "get_weather", "description": "…",
            "inputSchema": {"type": "object", "properties": {"city": …}}}]}
```

两个结构几乎逐字段同构——这不是巧合。function calling 解决的是"**模型怎么开单**"（单据的格式），但没有规定单子从哪来、谁接单、怎么知道有哪些单可开；MCP 补的正是这后半截。**MCP 没有发明任何新格式，它做的事情是把 harness 里那本硬编码的工具说明书，从源代码搬到了协议上**：加一个工具，从"改代码、重发版、重启进程"变成"服务器方自己上线，客户端下次 list 就有了"。"没发版"之谜的谜底就在这一搬家。

顺着说明书这个线索，把 MCP 服务器的三样原语摆上桌。规范（[Server Overview](https://modelcontextprotocol.io/specification/2026-07-28/server)）自己给的定性表：

| 原语 | 谁控制 | 是什么 | 例子 |
|---|---|---|---|
| Tools | **模型**控制 | 模型看情况决定调不调的函数 | 发 API 请求、写文件 |
| Resources | **应用**控制 | 应用决定往上下文里塞的数据 | 文件内容、git 历史 |
| Prompts | **用户**控制 | 用户主动点选的模板 | 斜杠命令、菜单 |

本篇只拆了 tools，但注意"谁控制"这一列——同一种数据，按"触发它的按钮在谁手里"分成三类，这是整个协议最锋利的一刀。你写过的 Claude Code 插件里，斜杠命令对应 prompts（用户点），`.mcp.json` 里的服务器对应 tools（模型点）。

到这里你已经看清了骨架。但这解释不了两件事：一是上下文管理篇欠的那笔账还没还——说明书从源代码搬到协议上，可它进了上下文，不还是要在驻留区常驻吃 token 吗？二是一个更根本的问题：这么小的协议，凭什么能当标准？

## L3：想得透——单子的格式是模型的，单子的来路是协议的

第三层，去掉所有术语。

> **MCP 是工具说明书的分发协议。function calling 定了"单据怎么开"，MCP 定了"单据的来路"：说明书从哪拉、拉到什么程度、单子开出来谁接。它把"给模型的一双手"从 harness 的源代码里拆出来，变成了可以插拔的外设接口。**

现在能读懂标题了。类比你大概率听过："MCP 是 AI 界的 USB-C"——[Anthropic 当年发布时](https://www.anthropic.com/news/model-context-protocol)就自称"USB-C port for AI applications"，这个类比准的部分是：**插座不提供任何能力，它只规定插头的形状**。MCP 服务器干不干活、干得好不好，协议一概不管；它只管三件事——怎么发现你（list/discover）、怎么调你（call）、怎么认识你（握手与版本）。类比失效的地方更值得说：USB-C 的电力是单向的、无对话的，而 MCP 管道里有双向消息（通知、进度、曾经的反向采样），所以它是"插座"但不是"墙上的插座"——更像是插线板，两头都能插。这个细节不是抬杠，它正是后面"协议做减法"的伏笔。

**为什么非得是这样设计？**拆三个被否决的替代方案，本质才立得住。

**方案一：把工具内置进客户端。**客户端厂商给每个主流系统写适配器。失败场景你已经能背出来：就是本文开头那个"没发版"之谜的反面——每接一个新系统发一次版，长尾系统永远排不上队，第三方生态彻底不存在。

**方案二：每个工具提供方自己开 HTTP 服务，各家客户端对着私有 REST API 去连。**失败场景在本地与信任两端：本地工具（读文件、跑 git）还得起网络服务、开端口、管防火墙，荒唐；而且"连"是主动伸手，授权、来源、谁能连，全要重新发明。MCP 的 stdio 传输选了相反的方向——**不是"你连我"，而是"我拉起你"**：客户端把服务器当子进程启动，stdin/stdout 对话，退出即回收。权限、生命周期、审计，全在拉起方手里。这不是工程偷懒，是信任模型的根本分岔，跟[插件篇](/2026/09/05/claude-code-plugins.html)的结论一脉相承——插件四件套全是声明式文本，审读即审计；stdio 服务器的信任链同样短到可以亲眼验证：配置里的一个 `command`。

**方案三：JSON-RPC 换成更新的协议。**为什么是这么个"古董"？因为 harness 与服务器之间需要的不只是"一问一答"：还有不需要回执的通知（你刚见识过）、以及同一条管道上无 schema 约束的任意消息。JSON-RPC 用"有没有 id"一个字节就把请求和通知分开了，并且跑在任何可靠字节流上——stdio 规范里明说：这份线格式原封不动地适用于 Unix 域套接字和 TCP。协议要的是"哪儿都能跑"，不是"哪儿都新"。

**然后是全文最漂亮的一处合流。**翻看 [2026-07-28 版规范](https://modelcontextprotocol.io/specification/2026-07-28/changelog)的变更清单，头两条读起来会有强烈的既视感：

> 1. Remove protocol-level sessions …… List endpoints no longer vary per-connection.（删除协议级会话……列表类端点不再随连接而变）
> 2. Make MCP stateless: remove the `initialize`/`notifications/initialized` handshake. Every request now carries its protocol version and client capabilities in `_meta`.（让 MCP 无状态：删除 initialize 握手。每个请求自带协议版本与客户端能力）

你亲手写的那个 `initialize` 握手——绝大多数现存教程的第一课——被规范自己删了，连同会话和 `Mcp-Session-Id` 头。为什么？[上下文管理篇](/2026/09/12/agent-context-management.html)否决"服务端做有状态"时论证过：无状态意味着任意一台机器能接任意一个请求，这是横向扩展的前提。协议作者们在一年半之后自己撞上了同一堵墙，做了同一个选择——**连接不再是状态，状态要么进每个请求的 `_meta`，要么变成服务器显式发下的句柄（handle）随单据流转**。连"聊天记录在客户端、服务器不存上下文"这条 MCP 从第一天就有的设计，也是同一个哲学。看到没有：无状态不是某家 API 的怪癖，是所有想被规模化使用的东西最终都要抵达的形状。

最后还账——上下文管理篇那道题："工具说明书住在驻留区里，一个回合都不能缺席。工具一多，光说明书就能吃掉几千 token。"MCP 给出的答案分两半。协议这半：说明书在协议上、在客户端手里，什么时候进上下文、进多少，由 host 决定——所以工具表可以按需刷新而不是一次全塞；2026-07-28 版甚至给列表结果加了 `ttlMs`/`cacheScope` 缓存提示，规范里写着这么一句[理由](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)：确定性排序"enables clients to reliably cache the tool list and improves LLM prompt cache hit rates"——协议自己都在操心你的 prompt cache 命中率（上篇讲的前缀稳定 = 钱，规范委员会同样在算这笔账）。host 这半：说明书可按需加载之后，还剩"模型怎么知道有工具可找"的问题——印象中 Claude Code 近期的做法是先给模型一个轻量的工具索引，用到哪类再把完整说明书递进去（ToolSearch 一类机制），细节以官方文档为准。两半合起来，"按需递说明书"从口号落成了机制。

## 收尾地图

一张选型速查表：

| 你的处境 | 先试什么 |
|---|---|
| 就一个固定工具，自己玩 | 硬编码 function calling 就够——60 行 harness 别急着升级 |
| 工具要多系统分发、第三方接入 | MCP：写个 stdio 服务器，50 行起步 |
| 一段"教模型干活的方法"，不碰外部系统 | 插件（skill/命令/subagent）——说明书在文件里，不需要协议 |
| 工具说明书撑爆上下文 | 按需注入：工具索引 + 用到才展开 + 工具表缓存 |

三个自测问题，能答上来说明真懂了：

1. 客户端发 5 条消息只收到 4 条回复，少的去了哪？如果你的服务器好心补了一条回复，具体会发生什么？（答案在 L1）
2. `tools/list` 的返回和你 60 行 harness 里的哪个变量几乎同构？MCP 到底把什么从源代码搬到了协议上？（答案在 L2）
3. 2026-07-28 版规范删掉了 initialize 握手。它和上下文管理篇否决"服务端有状态"的理由是哪一句话？（答案在 L3）

留一个我自己也没答案的真问题。规范里白纸黑字写着"clients MUST consider tool annotations to be untrusted"，设计原则里写着服务器不许读完整对话——协议把"说明书不可信"防得很好。但"该不该执行这次调用"的最终裁判，今天大多仍是一个确认弹窗：模型决定调什么，而模型恰恰是最容易被骗的那个——工具的返回值也是文本，也能注入。调用授权这件事，有没有比"人眼把关"更结构化的终态？我没想明白，评论区的高见我等着抄。

最后留一笔具体的账。这份协议的第一版里，藏着一个方向相反的能力：MCP 服务器可以反过来"借用"客户端的模型，让 host 替它跑一次续写（sampling）。到了 2026-07-28，它和 Roots、Logging 一起被正式标记为废弃——"让工具借模型的手"这条路，一年半就走到了头。**为什么？**一个协议，给"模型的手"立了标准，却亲手关掉了"借用模型"的门，这两件事背后的理由是同一句话。下一篇见分晓——顺便把远程 MCP 服务器（HTTP 传输）的信任链一起拆掉。

（补一句诚实的注脚：删掉握手的是最新规范，现实世界的服务器与 SDK 还在路上。新规范为此留了向后兼容的探针流程——先发 `server/discover`，对面答不上来就回退到 initialize 握手。你今天照本文写的服务器，短期内不会失效。）
