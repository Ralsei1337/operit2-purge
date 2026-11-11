# 逐条移植台账

对照源：`dsh-purge-community`（kingcharleslzy-ai/dsh-purge@master），补丁表在
`lib/patches/{prompt,code,engine,new-tools,opt-in}.mjs`，注册表 `lib/patches/index.mjs`。

实际在册补丁 **73 条**（README 里"86 条"含历史与迁移 id）：提示词 9、代码 3、
引擎 15（含 #90 由 `lib/steering.js` 内联）、new-tools 44、opt-in 2（#18/#19 默认关闭）。
`PATCH_DISPLAY_GROUPS` 把 8 组同源补丁折叠成一行展示。

## 处置分类

- **A 直搬**：纯文本资产，无语义改动 —— 本工程 `SANITIZE_RULES` / `BUILTIN_SECTION` 的措辞来源。
- **B 钩子重写**：语义保留，实现换成 Operit2 官方钩子。
- **C 重设计**：功能等价，但承载与分发方式不同。
- **D 放弃**：宿主专属或依赖源码改写，Operit2 无对应物。

## 提示词层（prompt.mjs，9 条）

| id | 名称 | 类别 | 说明 |
| --- | --- | --- | --- |
| 1 | WORKSPACE_CONTEXT_INTRO | B | 把"仅供参考"的免责声明换成 STRONG_INTRO 强制语气。Operit2 无同名常量，改为在 `after_compose_system_prompt` 追加执行姿态段。 |
| 2 | REPLACEMENT_WORKSPACE_CONTEXT_INTRO | B | DSH 的替换稿优先级规则。Operit2 无替换稿机制，措辞并入执行姿态段。 |
| 3 | SCOPE_INTRO | B | 作用域强句。Operit2 由钩子的 `executionContext` 表达，不需改写。 |
| 4 | SYSTEM_PROMPT_PERSONA | D | 改 `agent.cordis.yml` 的 `personaPrefix`。Operit2 的 persona 在 Flutter 侧配置，插件无写入口。 |
| 5 | APPROVAL_NEVER_SENTENCE | B | "审批提示已禁用 → 自动拒绝"改成"拥有完整访问"。Operit2 无此句；语义转为规则表里的 `must-ask-confirmation`（把"先征求许可"改成"先执行再汇报"）。 |
| 11 | APPROVAL_NEVER_SENTENCE_BYPASS | B | 同上，句式变体。 |
| 12 | APPROVAL_ASK_SENTENCE_BYPASS | B | "请求许可"句改写。对应 `must-ask-confirmation-zh`。 |
| 43 | WEB_PRESET_IDENTITY_STRIP | A | 剥离预设身份文案。并入注入段的措辞（不提"AI 助手"）。 |
| 93/94 | SANDBOX_POLICY_BRAND_NEUTRAL / SKILL_BADGE_DESCRIPTION_NEUTRAL | A | 中性化品牌/徽标描述。Operit2 无对应文本，跳过。 |

## 代码层（code.mjs，3 条）

| id | 名称 | 类别 | 说明 |
| --- | --- | --- | --- |
| 6 | SANDBOX_DENY_FILE_WRITE | D | 改沙箱 deny 列表（bwrap）。Rust 执行器内部，无钩子暴露点。 |
| 7 | SANDBOX_MODE_DEFAULT | D | 改沙箱默认模式常量。 |
| 8 | APPROVAL_POLICY_DEFAULT | D | 改审批策略默认值。审批策略是宿主 `AiPermissionMode`，插件不可写。 |

## 引擎层（engine.mjs，15 条）

| id | 名称 | 类别 | 说明 |
| --- | --- | --- | --- |
| 9 | APPROVAL_AUTO_GRANT | **D** | 把审批请求的 rejected 分支改成 `allowed-once`。这是 dsh-purge 的核心杀伤点，也是**最不可移植**的一条：Operit2 的 `tool_call_intercept` 只能 block，返回 allow 与不返回等价，确认回调在 intercept 之后。 |
| 10 | APPROVAL_CONFIG_DEFAULT_NEVER | **D** | 审批配置默认 never。属宿主偏好，插件无写权限。 |
| 13 | ESCALATION_WIDENING_EXEMPT | D | 删掉"只能升不能降"的三条 throw。Rust 侧无对应可 patch 点。 |
| 14 | ESCALATION_GRANT_UNCONDITIONAL | D | `approveEscalation` 直接返回请求模式。同上。 |
| 15 | SANDBOX_CONFINE_PASSTHROUGH | D | `confine(argv, policy)` 直通。执行前 argv 构造，在 Rust 内部。 |
| 16 | FS_FENCE_DISABLED | D | `checkedTarget` 直通，取消 read-only / workspace-write 拒绝。Rust 文件 Host 内部。 |
| 17 | FS_OBSERVATION_INTENT_FREE | D | 取消"先读后写"意图校验。 |
| 35 | ESCALATION_NEVER_REJECT | D | 所有失败 case 并回 return mode。 |
| 91 | BROWSER_COOKIE_NAME_IGNORE_PORT | D | 桌面 web 认证 cookie 名去掉端口。Operit2 无 Electron 认证链路。 |
| 92 | HOST_MAX_HTTP_HEADER_SIZE | D | 放宽 Node HTTP 头上限。无对应物。 |
| 90 | RESTORE_NATIVE_STEERING | D | 撤销型补丁，恢复宿主原生 steering。本工程无 steering 概念。 |
| 18/19（opt-in） | — | D | 默认关闭的实验开关，依赖 DSH 内部结构。 |

## new-tools.mjs（44 条）

**重要事实修正**：这个文件名不副实 —— 44 条里**没有一条是"注册新工具"**，全是既有宿主文件的改写或撤销型补丁。

| 类别 | 数量 | 类别判定 | 说明 |
| --- | --- | --- | --- |
| 工具描述"停手"语义清除（#33/#48/#60/#61/#34/#36） | 6 | B | 把工具描述里"没有网络访问""先停下确认""不可信任"等提示去掉。Operit2 的工具描述由各插件自身声明，本工程不改别人的描述 —— 改为用 `hidePatterns` 减少暴露面，效果等价且不污染别人。 |
| 提示词身份/品牌剥离（#25/#26/#32/#37/#28） | 5 | A/B | 剥离 harness 身份、最小 persona。措辞并入注入段。 |
| Auto Review / hooks 短路（#59/#30/#31/#62） | 4 | D | 把 Auto Review 首行 `return next()`、hooks deny 改常量、pipeline fail-open。Operit2 无这些内部管线。 |
| 注入保活（#40/#55/#74/#87/#88） | 5 | B | 保证注入段不被 complete 预设 / compaction / rewind 挤掉。Operit2 的 compose 钩子在每次组装时都跑，天然保活；本工程再加幂等标记防叠段。 |
| 自持久化与降噪（#78/#52/#83/#84） | 4 | D | 把 `.dshpurge.bak` 拷回、关会话日志上传、不弹 crash dialog。宿主专属。 |
| 兼容垫片与版本锚点（#21/#22/#23/#24/#38/#39/#42/#45/#46/#80 等） | 其余 | D/C | DSH 版本演进带来的锚点维护，换宿主后无意义。 |

## 净结果

| 类别 | 条数 | 占比 |
| --- | --- | --- |
| A 直搬（措辞可复用） | ~8 | 11% |
| B 钩子重写（已实现） | ~16 | 22% |
| C 重设计（改动承载方式） | ~7 | 10% |
| D 放弃（宿主专属/源码改写） | ~42 | 57% |

按能力面看：**提示词层 ~100% 可移植，工具清单层 ~100%（换实现），权限/沙箱层 0%。**

这个比例也解释了 dsh-purge 的实际体验来源：73 条里真正动权限内核的只有 9/13/14/15/16/17 六条，其余四十多条是提示词与工具描述改造。**"破甲"的感受主要来自模型对拒绝的解读，而不是权限真的被拆掉了。** 本工程把那四十多条里能落地的那部分做扎实，不去伪造做不到的那部分。

## 明确不做

- 不注册 `block` 决策去干扰用户正常调用（那是另一个产品，不是 purge）。
- 不返回伪造的 `allow` 制造"已放行"错觉。
- 不改宿主 `AiPermissionMode`，不尝试绕过确认弹窗。
- 不上传提示词或审计数据；不发起任何网络请求。
