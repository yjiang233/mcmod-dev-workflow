# mcmod-dev-workflow

> An Agent Skill that carries a Minecraft mod (NeoForge / Forge) development change end-to-end — and leaves no traps for the next run.
>
> 一套 **Agent Skill**：把 Minecraft 模组（NeoForge / Forge）的「一次开发改动」从头到尾走完，并且不留下会坑到下一次的尾巴。

这不是 API 教程，也不是工程脚手架。它编排的是**过程与纪律**：开始前读什么、每一步做到什么才算完成、哪些操作绝对不能自作主张、踩过的坑怎么按症状查回来。API 签名、项目结构、Gradle 命令由专门的模组开发技能或官方文档负责，本技能不重复收录。

> 本仓库是**发布版**：已去除本机绝对路径、代理端口与凭据。装到你的环境后，请按 [PORTABILITY.md](PORTABILITY.md) 做一次环境核对。

## 为什么需要它

AI 改 MOD 时最常见的四种翻车：

| 症状 | 根因 |
| --- | --- |
| 改了半天没效果 | 旧客户端进程还占着调试端口，新实例启动失败，请求全打到了旧进程 |
| 「编译通过」就宣布完成 | 没有验证游戏内实际效果，也没有可验证的完成判据 |
| 下次开工一脸茫然 | 只写了给 AI 读的隐藏记忆，没留下给人看的日志 |
| 工作区被推到远程 | 顺手执行了 `git commit` / `git push` |

这套技能把每一步写成**带判据的动作**，把这四种情况挡在流程外。

## 适用 / 不适用

| 适用 | 不适用 |
| --- | --- |
| 在 NeoForge / Forge 工程改 Java 代码、加方块／物品／GUI、修 bug | Blockbench 建模、贴图美术、粒子／动画制作 |
| 构建、单测、启动客户端验证、截图自查、查崩溃日志 | 纯资源包 / 数据包 / 命令与记分板 |
| 排查渲染黑屏、GUI 模糊、模型贴图不显示 | Paper / Bukkit / Spigot 插件 |
| 打包、发 Release、跨 MC 版本移植 | 纯 API 用法咨询（直接查官方文档） |
| 整理开发日志、同步知识库 | 视频剪辑、图片生成 |

即便任务提到 NeoForge / Forge，动作不属于「适用」列时也让路给更贴切的技能。

## 安装

复制**整个目录**到你的 Agent 的 skills 目录 —— 只拷 `SKILL.md` 会让 `references/` 指针全部断掉。

| Agent | 用户级 | 项目级 |
| --- | --- | --- |
| WorkBuddy | `~/.workbuddy/skills/mcmod-dev-workflow/` | `<workspace>/.workbuddy/skills/mcmod-dev-workflow/` |
| Claude Code | `~/.claude/skills/mcmod-dev-workflow/` | `.claude/skills/mcmod-dev-workflow/` |
| 其他 Agent | 以该平台当期文档为准；部分平台用 `AGENTS.md` / rules，需要转写格式 | 同左 |

```bash
# Claude Code 示例
git clone https://github.com/yjiang233/mcmod-dev-workflow.git ~/.claude/skills/mcmod-dev-workflow
rm -rf ~/.claude/skills/mcmod-dev-workflow/.git   # 可选：技能目录里不必再留一个仓库
```

Windows 下把 `~` 换成实际用户目录；克隆后按平台要求重启会话或启用技能。

**装完必做**：打开 [PORTABILITY.md](PORTABILITY.md) 的「必须替换的占位清单」，把记忆目录、知识库路径、双版本发布编排、端口与资源引用换成你环境里的实际值。跳过这一步，技能会「能加载、跑一半空转」。

## 快速验证

- 正例（应命中）：`按流程给这个 MOD 加个方块，改完记得同步日志`
- 负例（应让路）：`用 Blockbench 做个 16x16 的模型`

完整的 5 正 + 5 负与装／不装基线对比见 [examples/trigger-queries.md](examples/trigger-queries.md)。

## 工作流一览

`判定平台与分支 → 停掉运行中的客户端 → 改码 → 构建与单测 → 进游戏验证 → 同步日志 → 停手`

每一步都有完成判据，判据没过不进下一步；无截图能力时允许用日志／单测输出代替验证证据。完整步骤表在 [SKILL.md](SKILL.md)。

### 红线

- **不自动提交**：只有用户明确说「提交 / 上传 / 推送 / 打包 / 发布」才执行 git 写操作。
- **客户端运行时不构建**：覆盖 `build/classes` 会让正在跑的客户端抛 `NoClassDefFoundError`。
- **必须留下给人看的日志**：只写工作区隐藏记忆不算完成；日志要写清症状与根因。
- **凭据不落盘**：token 只从该环境的凭据存储读取，不写进技能、代码或任何会进仓库的文件。
- **发布不可逆**：执行前逐项核对 `references/delivery.md` 的清单，并再次确认用户指令。

## 目录结构

```text
mcmod-dev-workflow/
├── SKILL.md                       # 入口：触发边界、7 步工作流、红线、收尾清单
├── PORTABILITY.md                 # 跨 Agent / 跨工程迁移指南（装完先读）
├── README.md
├── LICENSE
├── references/
│   ├── environment.md             # 中文用户名编码、依赖网络、端口占用、调试端点
│   ├── rendering.md               # 矩阵、坐标系、缓冲分趟、GUI 与渲染器注册
│   ├── assets.md                  # 贴图复用、动画贴片、自动连接纹理、BlockItem
│   └── delivery.md                # 双版本发布、跨版本移植、知识库规范
└── examples/
    └── trigger-queries.md         # 触发验收集
```

`references/` 按需渐进读取：遇到哪类问题读哪篇，不用一次性全塞进上下文。

## 收录的踩坑（按症状查）

| 症状 | 根因 |
| --- | --- |
| 屏幕全黑、文字一个都不显示 | `new Matrix4f(...)` 是**列优先**，按行优先写基向量会把画布推到几十格外 |
| GUI 面板发虚、按钮文字看不清 | `Screen.render()` 内部会二次调用 `renderBackground()` |
| 渲染时崩 `Not building!` | 同一缓冲里交替画色块和文字，切 RenderType 会反复 `endBatch` |
| 方块实体渲染器完全不生效 | `RegisterRenderers` 要挂在 **MOD 总线** |
| 创造栏崩溃 `stack count must be 1 for air` | 用 `BLOCKS.register` 注册的方块没配 `BlockItem` |
| 注册自定义类型报 `Registry is already frozen` | 只能用 `DeferredRegister`，不能在 setup 阶段注册 |
| 修改后「改了没效果」 | 旧进程占着端口，调试请求打到了旧实例 |

## 跨 Agent / 跨工程

格式是通用的（`SKILL.md` + 仅 `name` / `description` 的 frontmatter + 相对路径 `references/`，无平台专属工具调用），**内容是按一个具体工程沉淀的**。换环境时按 `PORTABILITY.md` 处理：

1. 把记忆与知识库路径换成该 Agent 的实际机制；没有对应文件就跳过，不要凭空建目录。
2. 双版本发布编排、端口、代理、资源脚本都是实例值，目标工程不适用就删掉。
3. 版本号、MC / Java 版本、loader 一律**运行时从 `gradle.properties` / `build.gradle` 现场确认**，不沿用文档里的数字。
4. 缺截图能力、缺调试端点、缺知识库时按降级矩阵换等价证据，不能假装完成。

## 安全与状态

- 全包不含任何 token / PAT 或本机绝对路径；示例路径均用 `~` 或占位符表示。
- 提炼自一个 NeoForge + Forge 双版本模组工程的真实开发记录（2026-11）。技术坑可迁移，环境值不可迁移。
- **未实测项**：没有在 WorkBuddy 之外的 Agent 里真实加载并跑完整流程；目录映射与兼容性判断是格式规范 + 平台惯例的静态推演。正式使用前请按「快速验证」跑一次。

## 许可证

[MIT](LICENSE) · Copyright (c) 2026 yjiang233
