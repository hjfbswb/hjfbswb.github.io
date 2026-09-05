---
title: Claude Code 插件：一份清单、一个目录、和不需要服务器的应用商店
tags: [claude-code, ai, 工具链, 提示工程]
categories: [AI 工具]
---

你花了一个周末给 Claude Code 调教出一套顺手的斜杠命令：`/review` 按团队规范审代码，`/deploy` 走发布清单，还有一个 subagent 专门写 commit message。然后同事说"也给我一份"——最直接的办法是把 `.claude/` 目录打包发过去。过两周你改了 `/review` 的提示词，再发一遍；同事那边自己也有改动，合并靠肉眼。

更惨的是另一个方向：你听说有人给 Claude Code 接了 Jira、接了 Figma、还让它在每次编辑文件后自动跑 clang-format。你去问怎么弄的，对方回了一句：`/plugin install`。

插件系统（plugins）就是为淘汰"打包发 `.claude/`"这个动作而生的。本篇把它读薄成一句话：**插件 = 把"你是怎么教 Claude 的"从聊天记录和散装配置里抽出来，做成文件、钉进版本库、变成可以像 npm 包一样安装的交付物**。装完你会发现，它没有魔法——一个目录，一份清单，仅此而已。

## L1：先用起来——五分钟装上第一个插件

你在第一层。目标：会装、会用、会卸载。

入口是 `/plugin`，一个终端里的分页面板（Tab 切换：Discover 浏览、Installed 管理、Marketplaces 商店、Errors 报错）。第一次交互式启动 Claude Code 时，官方市场 `claude-plugins-official` 已经自动注册好了，所以最短的路径是：

```shell
/plugin install github@claude-plugins-official
```

回车，选安装范围，然后你多了个 GitHub 集成——本质是一组预配好的 MCP 服务器，Claude 从此能直接操作 issue 和 PR。装完注意安装摘要：如果写着 `Run /reload-plugins to activate.`，就跑一下这条命令，新组件才会挂进当前会话。

想逛社区市场，先手动注册再装：

```shell
/plugin marketplace add anthropics/claude-plugins-community
/plugin install <插件名>@claude-community
```

装好怎么用？关键规则一条：**插件的所有技能都带命名空间**。`commit-commands` 这个插件提供的提交技能叫 `/commit-commands:commit`，而不是 `/commit`。这是防撞名的设计——十个插件都可以有 `review` 技能，互不覆盖。

卸载和停用是对偶的两件事：

```shell
/plugin disable github@claude-plugins-official   # 留着，先不用
/plugin uninstall github@claude-plugins-official  # 彻底删掉
```

安装时还有个"范围"（scope）概念要交代，它决定装进哪个配置文件、谁能用：

| 范围 | 写进的文件 | 效果 |
|---|---|---|
| user | `~/.claude/settings.json` | 你自己所有项目可用（默认） |
| project | `.claude/settings.json` | 进版本库，全团队可用 |
| local | `.claude/settings.local.json` | 只有你、只有这个仓库 |

三条命令之内你已经会装会用会卸了。但"装"这个动作背后发生了什么——它往你机器上放了什么文件、凭什么能"自动生效"——一个字都还没提。往下拆。

## L2：拆开看——插件其实只是一个目录

到这一层，问题变了：装一个插件，到底改变了什么？

答案反直觉地朴素：**一个插件就是一个目录，外加一份可选的 JSON 清单**。没有二进制，没有安装脚本，没有注册表。标准长这样：

```text
my-plugin/
├── .claude-plugin/
│   └── plugin.json      # 清单：名字、版本、作者（唯一必填字段是 name）
├── skills/             # 技能：markdown 写的规程，模型按需调用
│   └── code-review/
│       └── SKILL.md
├── agents/              # 子代理：带独立系统提示词的"分身"
│   └── security-reviewer.md
├── hooks/
│   └── hooks.json       # 事件钩子：工具调用前后触发的 shell 命令
├── .mcp.json            # MCP 服务器：接外部系统的工具
├── .lsp.json            # LSP 服务器：让 Claude 获得跳转定义/查引用
├── bin/                 # 可执行文件，启用期间进 PATH
└── settings.json        # 插件自带的默认设置
```

`plugin.json` 长这样：

```json
{
  "name": "my-plugin",
  "description": "团队的代码质量工具集",
  "version": "1.0.0",
  "author": { "name": "Dev Team" }
}
```

就这么多。所谓"安装"，是 Claude Code 扫描这些约定目录，把组件注册进会话：`skills/` 里的每个 `SKILL.md` 变成一条带命名空间的技能，`agents/` 里的每个 markdown 变成一个可 @ 的子代理，`hooks.json` 挂到工具调用等事件上，`.mcp.json` 里的服务器随插件启停。**全部是声明式的文本**——你可以打开任何一个插件逐行审读，这在后文的安全话题里是命门级的事实。

那"应用商店"呢？更朴素：**marketplace 就是一个带 `.claude-plugin/marketplace.json` 的 git 仓库**。这份清单只做一件事——列菜单：

```json
{
  "name": "my-plugins",
  "owner": { "name": "Dev Team" },
  "plugins": [
    {
      "name": "code-formatter",
      "source": "./plugins/formatter",
      "description": "保存时自动格式化",
      "version": "2.1.0"
    }
  ]
}
```

`source` 可以是仓库内的相对路径，也可以指向另一个 GitHub 仓库。所以 `/plugin marketplace add anthropics/claude-code` 干的事，本质是 clone 一个仓库、读它的菜单。

安装还涉及一个容易忽略的动作：**拷贝进缓存**。从市场装的插件会被复制到 `~/.claude/plugins/cache`，每个版本一个目录，版本号就是缓存键——你改了代码但没 bump 版本，用户那边 `/plugin update` 会告诉你"已是最新"。同一插件的新旧版本目录会并存约 14 天（等还在用旧版的会话退出）再被清理。这条设计留着 L3 再审。

## 动手：十分钟造一个自己的插件

别满足于装，写一个。任意目录：

```bash
mkdir -p my-first-plugin/.claude-plugin
mkdir -p my-first-plugin/skills/hello
```

写 `my-first-plugin/.claude-plugin/plugin.json`：

```json
{
  "name": "my-first-plugin",
  "description": "学习用的问候插件",
  "version": "1.0.0"
}
```

写 `my-first-plugin/skills/hello/SKILL.md`：

```markdown
---
description: 用中文打个招呼
---

请热情地向 $ARGUMENTS 问好，并问一句今天能帮上什么忙。
```

不安装、不发布，直接加载：

```bash
claude --plugin-dir ./my-first-plugin
```

启动后在会话里敲 `/my-first-plugin:hello 老王`——Claude 会热络地和"老王"打招呼。改了 `SKILL.md` 想看效果？跑 `/reload-plugins`，不用重启。

**先预测再看答案**：如果你把 `skills/` 目录挪进 `.claude-plugin/` 里面——毕竟"元数据旁边放内容"看起来很合理——会发生什么？

答案是：插件照常加载，但**所有技能消失**，一个不剩。这是官方文档列为第一位的常见错误：`.claude-plugin/` 里只放 `plugin.json`，其余目录必须在插件根上。原因很简单——Claude Code 对约定目录的扫描是"从根开始找 `skills/`、`agents/`……"，没人告诉它去 `.claude-plugin/` 里面找。目录约定代替了配置，代价就是目录位置就是协议。

顺手再学一条诊断命令，它会替你算账：

```bash
claude plugin details my-first-plugin
```

输出里最值得看的是两行成本：`always-on`（这个插件每个会话常驻多少 token，比如技能的 description 就永远占着上下文）和 `on-invoke`（每个组件被触发时再花多少）。**插件不是免费的**——装十个插件，每个的技能描述都会挤进每一次对话的上下文。这行数字就是你的账单。

## L3：想得透——这套设计在防什么、图什么

到这里机制清楚了，但几个"为什么非这样设计"还悬着。这一层逐个回答，最后给本质。

**为什么一切是 markdown 和 JSON，而不是像 VS Code 扩展那样跑代码的包？** 因为插件的主体是"给 Claude 的说明书"——提示词、规程、工具声明。说明书的最佳载体是可 diff、可 review 的文本。团队评审一个插件，review 的是几段 markdown，而不是反编译一个 `.vsix`。这也直接决定了下一条。

**为什么安装是拷贝进缓存，而不是原地引用？** 三个动机叠加：版本隔离（新旧版本目录并存，互不踩踏，正是上节那 14 天宽限期的来源）；路径封闭（组件路径不允许逃出插件根目录，`../shared-utils` 会被直接拒绝，符号链接指向插件外也会被跳过——拷贝一份封闭的快照，插件运行时碰不到你机器上的其他东西）；以及搜索卫生（被标记为孤儿版本的目录会被 Glob/Grep 跳过，Claude 不会翻到过期的插件代码）。

**为什么 marketplace 是个 git 仓库，而不是 Anthropic 运营的中心商店？** 这是整个设计里最聪明的一刀。npm 有 registry，App Store 有苹果——而 Claude Code 的插件市场**不需要任何中心服务器**：任何一个 GitHub 仓库、任何一个私有 GitLab、甚至本地一个目录，放一份 `marketplace.json` 就是商店。团队分发因此变成纯 git 问题：私有仓库 + 项目 `.claude/settings.json` 里一行 `extraKnownMarketplaces`，同事 clone 完信任目录即自动挂载商店，权限、审计、版本全部复用 git 已有的答案。信任模型也随之明确——你信任一个市场，等价于你信任那个仓库的维护者。

**为什么装完有时不生效，要 `/reload-plugins`？** 因为挂载新组件会往上下文里注入新内容，这会使已缓存的对话前缀失效。Claude Code 让你显式确认这笔"重读全文"的成本，而不是默默扣掉。一个让你_reload 的提示，背后是 prompt cache 的经济账。

**必须直面的反面：这是一套高信任模型。** hooks 本身就是 shell 命令，`.mcp.json` 会拉起子进程，`bin/` 直接进 PATH——插件可以以你的用户权限执行任意代码，官方文档原话是 "highly trusted components"。中心化审核是不存在的：官方市场是 Anthropic 自选，社区市场有自动校验，但两者都不等于安全背书。所以装第三方插件前的动作清单是：`claude plugin details` 看它到底贡献了哪些组件；打开仓库读 `hooks.json` 和 `.mcp.json`（好在全是可以直接读的文本）；企业侧则可以用 managed settings 白名单化允许的市场。

最后收口成本质。去掉所有术语：**插件是把"教 Claude 干活的方法"文件化之后，得到的一个可以放进 git、可以按版本安装、可以开关的包裹；应用商店则是一个可以由任何人、用任何 git 仓库充当的菜单。** 它选择的方案是"约定目录 + 文本清单 + git 分发"，被否掉的是三条路——中心化的商店（分发被平台卡脖子）、二进制包（不可审读，信任无处安放）、原地引用（无版本隔离，无路径封闭）。理解了"说明书文件化 + 分发 git 化"这两个动词，整个系统就没有再需要解释的部件。

## 收尾：一张决策表

| 你的处境 | 该用的做法 |
|---|---|
| 个人小工作流，图快 | 直接写 `.claude/`（standalone），不打包 |
| 团队共享、要跟代码一起走版本 | 做成插件，私有 git 仓库当 marketplace，`--scope project` 安装 |
| 想发给全网用户 | 提交到社区市场 `claude-plugins-community` |
| 只是想试试一个本地插件 | `claude --plugin-dir ./路径`，零安装 |
| 发布前检查 | `claude plugin validate ./路径`（CI 里加 `--strict`） |

两条自测题，能答上来就是真懂了（答案都在正文里）：

1. 同事说"插件装了，新命令却敲不出来"——按概率排前两位的原因是什么？（提示：一个看目录位置，一个看安装摘要。）
2. 为什么 Claude Code 不需要运营一个插件中心服务器，而 npm 需要？（提示：想想 marketplace.json 放在哪、由谁托管。）

下一站：把本篇的目录约定当跳板，去读三个更深的部件——`hooks.json` 的事件表（它是插件里唯一"主动执行"的东西，也是安全的重点）、子代理的 frontmatter 字段（`model`、`tools`、`maxTurns`），以及 MCP 协议本身。官方文档都在 [code.claude.com/docs](https://code.claude.com/docs/en/plugins)，而你现在带着地图了。
