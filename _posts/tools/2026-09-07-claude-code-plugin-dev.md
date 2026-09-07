---
title: 动手写一个 Claude Code 插件：从一条斜杠命令到事件钩子
tags: [claude-code, ai, 工具链, 提示工程]
categories: [AI 工具]
---

[上一篇](/2026/09/05/claude-code-plugins.html)我们把插件这个包裹拆开看了个遍：一个目录，一份 `plugin.json`，几个约定位置的子目录。结尾埋了三个伏笔——`hooks.json` 的事件表、子代理的 frontmatter 字段、MCP 协议。本篇来还债，方式不是继续拆别人写的插件，而是从空目录开始，把每一级组件亲手造一遍。

动手之前先给你一个反直觉的预期管理：**整个开发流程里没有"编译"这个步骤**。没有 `npm run build`，没有生成物，没有链接器。你写下的是 markdown 和 JSON，Claude Code 读的就是这些文件本身。整个工具链里最接近"编译器"的东西，是一条验证命令：

```shell
claude plugin validate ./你的插件目录
```

它不产出任何东西，只负责告诉你"写得对不对"。这种"源码即成品"的体验是插件系统设计的第一性选择，本文结尾会回到这一点。现在，先让它跑起来。

## 第零步：一个空目录和一份清单

任意位置建目录，放进唯一必需的文件：

```text
demo-plugin/
└── .claude-plugin/
    └── plugin.json
```

```json
{
  "name": "demo-plugin",
  "description": "博客示例：从零造一个插件",
  "version": "0.1.0"
}
```

跑一下验证：

```text
$ claude plugin validate ./demo-plugin

Validating plugin manifest: ./demo-plugin/.claude-plugin/plugin.json

⚠ Found 1 warning:
  ❯ author: No author information provided. Consider adding author details for plugin attribution

✔ Validation passed with warnings
```

唯一必填字段是 `name`，`author` 只是被建议补上（加上 `"author": {"name": "你的名字"}` 即可消掉警告）。注意 `plugin.json` 在 `.claude-plugin/` 里面，其余所有组件目录在插件根上——上篇说过，把 `skills/` 挪进 `.claude-plugin/` 是官方文档列出的头号常见错误，目录位置就是协议。

此时这个插件还什么都不会。接下来逐级加东西，每一级都**实际跑通**——本文所有命令和输出都在本机 Claude Code 上真实执行过。

## 第一级：一条会说话的命令

加一个文件：

```text
demo-plugin/
├── .claude-plugin/plugin.json
└── commands/
    └── hello.md
```

`commands/hello.md` 的内容：

```markdown
---
description: 向指定的人问好
allowed-tools: Bash
argument-hint: [名字]
---

请用中文热情地向 $ARGUMENTS 问好，并报出当前时间（用 date 命令）。
```

不安装、不发布，直接加载：

```shell
claude --plugin-dir ./demo-plugin
```

会话里敲 `/demo-plugin:hello 世界`，真实输出：

> 你好，**世界**！🌍✨
>
> 现在是 **2026年9月7日 星期一 中午 12:11 (CST)** —— 正是午饭好时光 🍜

别小看这两行输出，它证明了三件事：`$ARGUMENTS` 把"世界"塞进了提示词；`allowed-tools: Bash` 放行了 `date` 命令的执行；命名空间前缀 `/demo-plugin:` 生效了。一条命令 = 一段预写好的提示词 + 一份 frontmatter 配置，这就是全部。

frontmatter 常用字段一张表：

| 字段 | 作用 | 取值示例 |
|---|---|---|
| `description` | `/help` 里的一句话说明 | `Review code for security issues` |
| `allowed-tools` | 本次命令可用的工具白名单 | `Read, Grep, Bash(git:*)` |
| `argument-hint` | 自动补全里的参数提示 | `[pr-number] [priority]` |
| `model` | 执行用的模型档位 | `haiku`（便宜快）、`sonnet`、`opus` |
| `disable-model-invocation` | 禁止模型自己调用，只许人敲 | `true` |

`$ARGUMENTS` 是"所有参数拼成一串"；需要拆开用 `$1`、`$2` 取位置参数。`Bash(git:*)` 这种带括号的写法是"只允许 git 开头的 Bash 命令"——白名单可以细到命令前缀。

到这里你已经有了最小可用插件。但这条命令每次说的一模一样，而真正有用的提示词往往需要"当下的事实"——改了哪些文件、当前在哪个分支。这就是下一级要解决的问题。

## 第二级：让命令带着上下文出发

命令文件里可以做两种"注入"，都写在正文里：

```markdown
---
description: 审查本次改动
allowed-tools: Read, Bash(git:*)
---

变更文件清单：!`git diff --name-only main...HEAD`

用 Read 读取上面清单中的每个文件，逐个审查：

- 代码质量与风格
- 潜在 bug
- 测试覆盖
```

两个语法各干一件事：

- **`` !`命令` ``**（反引号前加感叹号）：命令执行结果**在提示词送进模型之前**被展开。于是 Claude 看到的不是"请审查改动"，而是一份已经填好的文件清单。适合注入 git 状态、环境变量这类"此刻的事实"。
- **`@路径`**：文件引用，路径指向的内容会被读进上下文，省去 Claude 自己先去找文件。配合参数用是常客：`审查 @$1 里的安全问题`——用户敲 `/audit src/api/users.ts`，Claude 拿到的提示词里已经躺着文件内容。

插件命令还独享一个环境变量 `${CLAUDE_PLUGIN_ROOT}`，指向插件自身的安装路径。引用插件自带的脚本、模板、配置时用它，别写死绝对路径——你不知道用户的机器长什么样：

```markdown
运行分析脚本：!`node ${CLAUDE_PLUGIN_ROOT}/scripts/analyze.js $1`
```

顺带兑现一个上篇悬着的分辨题：**`commands/` 和 `skills/` 都是斜杠命令，区别在哪？** 答案是触发方式的分工——`commands/` 是**用户显式触发**的（你敲 `/demo-plugin:hello`）；`skills/` 下的 `SKILL.md` 除了同样能敲，还带一个 `description` 触发器，**模型读到匹配的场景时会自己想起并调用它**。官方文档已把 `.claude/commands/` 目录标记为 legacy 格式，推荐新写法直接用 `skills/<名字>/SKILL.md`——两者加载方式完全相同，只是文件布局不同。经验法则：给人敲的用命令，给模型"条件反射"的用技能。

静态提示词和动态上下文都有了，你可能会开始把越写越长的提示词塞进一个文件。但很快会撞到一堵墙：这些提示词全在**主对话的上下文里**执行——审查一个 5000 行的 diff，读进去的代码会把上下文撑得很大，而且"审查者"和"写作者"用的是同一份对话记忆。下一级解决这个。

## 第三级：子代理——一个带独立记忆的分身

加一个文件，插件就多了一个"可以派活出去"的分身：

```text
demo-plugin/
├── .claude-plugin/plugin.json
├── commands/hello.md
└── agents/
    └── reviewer.md
```

```markdown
---
name: reviewer
description: |
  Use this agent when the user asks to "review my code", "check my changes",
  or mentions code review. Typical triggers include pre-commit review,
  reviewing a pull request, and auditing recent changes.
model: inherit
color: cyan
tools: ["Read", "Grep", "Glob"]
---

You are a code reviewer specializing in pragmatic, high-signal feedback.

**Your Core Responsibilities:**
1. Find real bugs and risky changes, not style nits
2. Check test coverage for changed behavior
3. Flag security issues (injection, auth, path traversal)

**Output Format:**
- 按严重程度排序的发现列表，每条带文件名和行号
- 一句总体结论：可合 / 需改
```

frontmatter 关键字段：`name`（小写字母、数字、连字符，3–50 字符）和 `description` 是必需的；`model` 可以是 `inherit`（跟随主会话）、`sonnet`、`opus` 或 `haiku`；`tools` 不写就是全工具，写了就是最小权限——比如这个 reviewer 只需要读，连 `Bash` 都不给。

**`description` 是整个文件最重要的字段，没有之一。** 它会被常驻加载进上下文（这就是插件 token 账单的一部分），模型完全靠它判断"什么时候派这个分身出去"。写得含糊，该触发时不触发；写得啰嗦，白烧上下文。官方模板的推荐结构是：一句话触发条件（"Use this agent when..."）+ 两三个典型场景 + 明确什么情况**不要**用。注意上面的示例里触发描述是英文——这不是审美，是实测经验：触发器要匹配用户输入和模型内部检索的语言，双语团队就两个都写。

子代理跑在**独立的上下文**里：主对话派活时把任务交给它，它自己读文件、自己思考，只把结论带回来。5000 行 diff 的阅读成本发生在分身的上下文里，用完即弃。代价是多一层调度开销——小事别用。

命令、技能、子代理，到目前都是"有人在说话"——用户敲命令，或模型决定调用。它们都是被动的。而插件里唯一**主动执行**的组件，是下一级。这也是上篇埋得最深的伏笔。

## 第四级：钩子——事件驱动的自动化

前面所有组件回答的都是"怎么教 Claude 说话"；钩子回答的是另一个问题：**在 Claude 工作流程的某个时刻，要不要让一段代码自动跑起来**。这就是 `hooks/hooks.json`：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo \"[$(date '+%H:%M:%S')] session started\" >> /tmp/hook-log.txt"
          }
        ]
      }
    ]
  }
}
```

注意插件格式的 `hooks.json` 有一层 `"hooks": {...}` 包装（直接写在 `.claude/settings.json` 里的钩子没有这层包装）。加载这个插件、启动会话，日志文件里真的多了一行：

```text
$ cat /tmp/hook-log.txt
[12:18:47] session started
```

这就是钩子的全部心智模型：**事件 → 匹配器 → shell 命令**。事件是 Claude Code 工作流里的固定时刻：

| 事件 | 触发时机 | 典型用途 |
|---|---|---|
| `SessionStart` | 会话开始（含 compact/clear 后重启） | 注入项目上下文、加载环境 |
| `UserPromptSubmit` | 用户发出一条提示之后 | 给提示附加上下文 |
| `PreToolUse` | 工具调用**之前** | 拦截危险操作、改写参数 |
| `PostToolUse` | 工具调用之后 | 自动格式化、审计日志 |
| `Stop` | 主 agent 打算结束时 | 完成度检查 |

把整条工作流画出来，钩子的位置一目了然：

```text
 用户敲下提示
     │
     ▼
[UserPromptSubmit] ──→ 模型思考，决定调用某工具
     │
     ▼
[PreToolUse] ──→ ┃ 拦下 / 放行 / 改写参数 ┃ ──→ 工具真正执行
     │                                            │
     ▼                                            ▼
   （拒绝则打回给模型）                      [PostToolUse]
     │                                            │
     ▼                                            ▼
   模型继续思考 ◄──────────────── 模型继续思考
     │
     ▼
[Stop] ──→ ┃ 允许结束 / 打回继续干 ┃
```

最有力的是 `PreToolUse`，因为它是唯一能在坏事发生**之前**说不的地方。给上面的事件表加一段：

```json
"PreToolUse": [
  {
    "matcher": "Bash",
    "hooks": [
      {
        "type": "command",
        "command": "jq -r '.tool_input.command' | grep -q 'rm -rf' && echo '{\"hookSpecificOutput\":{\"permissionDecision\":\"deny\",\"reason\":\"团队策略：禁止 rm -rf\"}}' || true"
      }
    ]
  }
]
```

工作原理：钩子命令通过 **stdin 收到一份 JSON**，包含 `session_id`、`cwd`、`tool_name`、`tool_input` 等字段；它往 **stdout 回一份 JSON** 表达决定。把这段命令单独喂样例输入验证逻辑：

```text
$ echo '{"tool_name":"Bash","tool_input":{"command":"rm -rf /tmp/some-dir"}}' \
    | jq -r '.tool_input.command' | grep -q 'rm -rf' \
    && echo '{"hookSpecificOutput":{"permissionDecision":"deny","reason":"..."}}'

{"hookSpecificOutput":{"permissionDecision":"deny","reason":"..."}}
```

 stdin 进、决定出，一个纯粹的函数。`matcher` 字段决定哪些工具触发这个钩子：`"Bash"` 精确匹配，`"Read|Write|Edit"` 多选，`"mcp__.*"` 正则匹配所有 MCP 工具。

**先预测再看答案**：如果把 `matcher` 写成小写的 `"bash"`，会发生什么？

答案是：钩子**一次都不会触发**，且没有任何报错。matcher 对大小写敏感，工具名是 `Bash` 不是 `bash`。一个字符的大小写，一段静默失效的防线——这类"静默不生效"是钩子调试的主要痛点，定位手段是二分法：先确认事件名和 matcher（用 `claude plugin validate` 查结构），再单独跑脚本喂样例 JSON（像上面那样），最后才怀疑语义。

两个容易踩的坑补在这里。其一，钩子命令的工作目录不一定是项目根，引用插件自己的文件一律用 `${CLAUDE_PLUGIN_ROOT}`——superpowers 插件的真实写法就是 `"command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start"`。其二，`SessionStart` 钩子可以把 `export KEY=value` 追加进 `$CLAUDE_ENV_FILE`，为整个会话注入环境变量——这是往会话里塞项目配置的正道。

除了 `type: "command"`，钩子还有一种 `type: "prompt"` 形态：不跑 shell，而是让一个小模型按你给的自然语言规则判断放行与否（`"prompt": "Evaluate if this tool use is appropriate..."`）。确定性策略用 command，模糊判断用 prompt——前者快且可预测，后者灵活但每次调用都要花 token。

## 第五级：一瞥 MCP

钩子管"事件"，MCP（Model Context Protocol）管"工具"——让 Claude 能操作 git 仓库之外的世界：数据库、issue 系统、浏览器。插件通过根目录的 `.mcp.json` 声明随插件启停的 MCP 服务器：

```json
{
  "filesystem": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"]
  }
}
```

三种形态：`stdio`（拉起本地子进程，如上例）、`type: "sse"` 和 `type: "http"`（连远程服务器，OAuth 自动处理）。配置里的 `${CLAUDE_PLUGIN_ROOT}` 和 `${DB_URL}` 这类占位符会被展开，后者取自用户环境——密钥不落盘。MCP 是个够写三篇的话题，本篇只把门推开一条缝：记住它是插件四件套之外唯一"重"的组件，装一次服务器，进来一组 `mcp__服务器名__工具名` 的工具。

## 本质：插件的源代码是写给模型读的

五级台阶走完，可以收口了。去掉所有术语：**Claude Code 插件开发，就是把你教 Claude 的方式文件化——你开口的时刻做成命令，它该想起来的时刻做成技能，要派出去干的活做成子代理，世界发生事件的时刻做成钩子**。四种文件对应协作里的四种时刻，仅此而已。

再追问一层设计权衡：为什么除了钩子和 MCP，"源代码"全是 markdown 而不是代码？因为**这些文件的读者是模型，而模型的原生接口就是自然语言**。命令、技能、子代理的"逻辑"本来就是提示词——它们此前活在你和 Claude 的聊天记录里，插件只是给这段提示词一个文件名、一个版本号。反过来，凡是需要**确定性**的地方——拦截危险命令、改写参数、连外部服务——markdown 就无能为力了，那里才轮到 shell 和真正的代码出场。这就是为什么整套 API 里只有钩子是"事件回调"的形状：它守着"必须每次都一样"的那条线。

类比着理解：钩子之于 Claude Code，很像 Express 之于 Node——每个请求过一遍中间件链，中间件可以放行、拒绝、改写。但类比到这里要拆：Express 中间件可以完全接管并改写响应流，钩子不能——它拦得住输入，注入得了消息，却从不"代替"模型说话。理解这条边界，你就理解了钩子是护栏而不是遥控器。

## 收尾：验证、成本与分发

开发循环的最后一环是质量关：

```shell
claude plugin validate ./demo-plugin    # 本地验证，CI 里加 --strict
claude plugin details demo-plugin       # 组件清单 + token 成本账单
```

`details` 输出的 `always-on` 成本值得每次发布前看一眼——技能和子代理的 description 是常驻税，钩子是被触发才付费。分发则复用上篇的结论：git 仓库 + `.claude-plugin/marketplace.json` 就是一个商店，`claude plugin tag ./demo-plugin` 可以打一个和清单版本号互相校验过的发布标签。

一张决策表收束全篇：

| 你想要的效果 | 用哪一级 |
|---|---|
| 把常用的提示词变成一条命令 | 第一级：`commands/` |
| 提示词需要当前的事实（git 状态等） | 第二级：`!`命令`` 内联注入 |
| 模型在特定场景自动想起某套规程 | 第二级：`skills/`（带触发器） |
| 任务很重，别污染主上下文 | 第三级：`agents/` 子代理 |
| "每次 X 发生时自动 Y" | 第四级：`hooks/` 事件钩子 |
| 让 Claude 操作外部系统 | 第五级：`.mcp.json` |

两条自测题，能答上来就是真懂了：

1. 团队禁令是"不允许直接 push 到 main"。你的钩子 matcher 写 `"Bash"`，脚本里 `grep 'git push'` 并输出 deny——但有人绕过了。最可能的漏洞在哪？（提示：想想 `git` 命令有多少种写法，以及 grep 匹配的是整个字符串还是子串。）
2. 同一个需求"每次写完文件自动跑 clang-format"，用 `PostToolUse` 钩子和用子代理各怎么做？为什么必须选钩子？（提示：谁是"事件发生时"被动触发的，谁需要有人派活？）

下一篇的伏笔埋在这里：你会发现本篇的所有验证都是"结构对不对"——`validate` 查文件布局，`details` 算 token 账，但没有任何一步验证"提示词写得**好**没有"。而本机 CLI 里还藏着一个 `claude plugin eval`：给插件定义一组测试用例和评分标准，跑一遍打分。怎么给一段自然语言的"代码"写测试？这是比 hooks 更深的问题，下篇展开。

——以及，本文开头那个 `demo-plugin` 目录还在 `/tmp` 里躺着。与其读到这里就关掉，不如现在就把它挪到你自己的项目里，改掉那个问候语。十分钟后你会发现：最难的一步从来不是语法，是决定把哪段提示词固化下来。
