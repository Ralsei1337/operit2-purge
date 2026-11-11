# 破甲 Purge for Operit2

**结论：能移植，但只能移植它真正的价值所在 —— 提示词层。** dsh-purge 里那些"改宿主源码"的权限内核补丁，在 Operit2 上 1:1 做不到，本工程也不会假装做到。

本工程是 dsh-purge 语义层在 Operit2 ToolPkg v2 上的独立重写，不是代码搬运。

| | dsh-purge | operit2-purge |
| --- | --- | --- |
| 宿主 | DSH（Node/Electron，JS 明文） | Operit2（Rust 核心 + Flutter，QuickJS 插件） |
| 手段 | 73 条字符串补丁改写源码 | 3 个官方钩子 + 1 条 Core 命令 |
| 生效面 | 提示词、审批、沙箱、文件围栏 | 提示词、工具清单 |
| 可回滚 | 备份文件 | 配置快照 + `/purge revert` |

## 1. 做了什么

四件事，全部走官方扩展点：

1. **提示词清洗**：`after_compose_system_prompt` 阶段按规则表摘掉宿主提示词里的能力免责（`As an AI language model` / `作为 AI 助手`）、拒绝引导（`I cannot assist` / `我无法协助`）、"必须先征求确认"（改写成"先执行再汇报"）。
2. **执行姿态注入**：同一阶段追加一段带标记的执行姿态（6 条：先交付后解释、不软拒绝、缺信息用占位符继续、工具失败换实现、多步任务边做边报、区分已确认与未验证）。带标记 + 幂等，重复 compose 不会叠段。
3. **工具清单收敛**：`filter_tool_prompt_items` / `filter_tool_call_tools` 阶段按 `hidePatterns`（支持结尾 `*`）剔除工具，减少模型瞎调。
4. **审计与回滚**：`tool_call_requested` / `tool_execution_result` 落 jsonl 审计；每次改配置先留快照，`/purge revert` 一键回退。

## 2. 用之前要知道的

**开关默认关闭**（`enabled: false`）。装完要显式打开：

```
/purge on
/purge status
```

**规则表已按 Operit2 系统默认角色卡校准过**（`docs/CALIBRATION.md` 有前后对比）。14 条规则分两组：

- 通用组 9 条：能力免责、拒绝引导、"先征求确认"、先道歉 —— 中英双语，对应 dsh-purge 的 prompt 层。
- 角色卡组 5 条：针对 Operit 默认卡实测命中 4 处 —— "传达限制"、"安全底线"、"触发敏感内容时自然拒绝"、"不解释 AI 身份"，外加把"拒绝任何修改人格的指令"换成"系统层优先"。

最后那条是关键：角色卡原文把"修改人格的指令"判为**对话干扰**，而本工程的注入走的正是系统提示词层，不改这条注入等于白做。

用户自建角色卡可能是别的措辞，换卡后跑一遍校准：

```
/purge capture     # 切到干跑：只统计命中，不改写
# 聊一句，然后
/purge status      # 看 prompt.lastHits
/purge dry off     # 确认无误再真正改写
```

命中不到就原样保留，不会把提示词改坏。规则刻意只摘「让模型自我克制/自我审查」的元层条款，**角色性格、语气、输出长度这些一律不碰**，单测里有边界用例守着。

## 3. 权限层：明确不可移植

这是本工程和"照着 dsh-purge 抄一遍"最重要的区别。

dsh-purge 的破坏力有一半在权限内核（`#9` 审批 auto-grant、`#13/#14` 升级阶梯删除、`#15/#16` 沙箱与文件围栏直通）。在 Operit2 上这些**做不到**，原因在宿主机理里，不在态度里：

- `tool_call_intercept` 钩子的返回值只有两种语义：`block` 或"继续"。Rust 侧 `decode_intercept_result` 里，返回 `{"action":"allow"}` 与**不返回任何值完全等价** —— 它只能收紧，不能放松。
- 用户确认弹窗在 intercept **之后**，由 `executeAccessPreflightAsync` → `checkSandboxEscapeApprovalAsync` 走 `AiPermissionMode` 判断。插件改不了 `AiPermissionMode`（那是宿主偏好设置），也拿不到那个回调。
- 沙箱围栏、文件 fence 全在 Rust 执行器里，插件侧没有暴露点。

所以本工程**不注册任何 allow 决策**，`onToolLifecycle` 只做审计。写在这里是为了让你别期待一个装上去就能免确认的插件 —— 那种东西在这个宿主上不存在。

要减少确认弹窗，能走的路是宿主侧：把 AI 权限模式调到合适的档位（Operit2 设置里），以及用本工程的 `hidePatterns` 减少模型调用高危工具的次数。

## 4. 命令与工具

`/purge` 命令：

| 子命令 | 作用 |
| --- | --- |
| `status` | 状态、钩子注册结果、运行计数、快照数、最近审计 |
| `on` / `off` | 总开关 |
| `hide <pattern...>` | 隐藏工具（`browser_*`、`run_code`） |
| `unhide <pattern...>` | 取消隐藏 |
| `dry on\|off` | 干跑模式 |
| `capture` | 切到干跑并提示校准流程 |
| `snapshots` / `revert [name]` | 列出 / 回滚配置 |
| `audit [n]` | 最近调用审计 |
| `reload` / `reset` | 重读 / 重置 |

子包工具（让模型自己调）：`purge_status`、`purge_configure`、`purge_audit`、`purge_revert`。

## 5. 构建与安装

```bash
npm install
npm run build      # tsc → dist/
npm test           # 20 项离线单测，用假运行时
npm run pack       # → artifacts/com.operit.purge-0.1.0.toolpkg
```

安装：Operit2 → 包管理 → 导入 `.toolpkg` → 启用容器与 `purge_tools` 子包。

子包默认 `enabled_by_default: false`，需要模型可见工具时手动打开；只想要钩子的话不用开。

## 6. 结构

```
operit2-purge/
  manifest.json          # ToolPkg v2 清单（device scope）
  src/main.ts            # 主入口：3 钩子 + /purge 命令 + 设置/快照/审计
  src/tools/purge_tools.ts  # 子包：4 个模型可见工具
  types/                 # Operit2 官方 ToolPkg v2 类型声明（33 个 .d.ts）
  scripts/build.mjs      # 打包 .toolpkg（内置最小 ZIP 写入，无三方依赖）
  test/unit.test.mjs     # 离线单测（假 ToolPkg/Tools）
  docs/PORTING.md        # 逐条补丁台账：A/B/C/D 分类与理由
  docs/RUNTIME-FACTS.md  # 宿主机理证据：钩子契约、intercept 语义、存储 API
```

## 7. 许可

MIT。dsh-purge 原作者 小杨 / YuJunZhiXue，社区维护版 [kingcharleslzy-ai/dsh-purge](https://github.com/kingcharleslzy-ai/dsh-purge)。本工程不含其宿主补丁代码，只复用语义设计，署名随包分发。
