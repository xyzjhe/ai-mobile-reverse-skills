---
name: ai-mobile-reverse-skills
description: 面向移动安全分析场景的 6 阶段总控 Skill。用于统一调度 APK 静态侦察、流量与代码对齐、SO/JNI 深度分析、加密与漏洞综合分析、验证设计与报告交付流程。支持 JADX MCP、Burp/Yakit MCP、IDA/Ghidra MCP，采用 AI 主导分析与阶段脚本组合的方式。
---

# Skill: AI Mobile Reverse Skills

## 定位

本 Skill 是整套移动安全分析体系的根入口，也是唯一的阶段调度入口。

它面向以下典型场景：

- Android App 静态逆向分析
- 抓包结果与源码实现联动分析
- SO / JNI / native 通信与加密逻辑分析
- 弱加密、认证授权、组件安全、业务逻辑等高风险问题筛查
- 授权环境下的最小验证方案设计
- 渗透测试报告与交付物生成

它不替代具体阶段做细节分析，而是负责：

1. 识别用户当前要执行的阶段。
2. 识别当前已连接的 MCP、已提供的输入材料和已有的中间产物。
3. 检查当前阶段的最低启动条件是否满足。
4. 将任务路由到正确的阶段 Agent。
5. 约束各阶段按统一输入输出契约推进。

进入某一步后，必须以对应阶段文件为直接执行标准。

## 运行方式

本 Skill 采用固定 6 阶段执行模式。所有模式都从 Phase 1 开始，区别只在于：

- 每一阶段结束后是否默认挂起，等待人工确认
- 自动化从哪一段开始接管后续流程

运行入口统一分为两大类，且允许用户在开始时显式选择：

- Phase 1：APK 静态侦察
- Phase 2：流量与代码对齐
- Phase 3：SO 与 JNI 深度分析
- Phase 4：弱加密与高风险漏洞筛查
- Phase 5：最小验证 POC 设计
- Phase 6：渗透报告汇总

### 模式选择规则

用户在开始时可以显式声明：

- `run_mode: step_by_step`
- `run_mode: auto_chain`

若 `run_mode = auto_chain`，则还可以继续声明：

- `auto_chain_mode: A`
- `auto_chain_mode: B`
- `auto_chain_mode: C`

总控执行原则：

1. 若用户已显式声明模式，则总控必须优先按该模式执行，不再自行猜测。
2. 若用户未声明模式：
   - 默认进入 `step_by_step`
   - 只有当用户明确表达“自动推进”“一条提示词跑完”“后面自动继续”等含义时，才进入 `auto_chain`
3. 若用户声明了 `auto_chain` 但未声明 `auto_chain_mode`，总控应根据当前最早可行切入点推断：
   - 若用户强调“第一阶段后人工处理抓包与前置准备，再自动推进后续”，优先按 `A`
   - 若用户强调“前 1-3 阶段人工控制，后 4-6 自动收口”，优先按 `B`
   - 若用户强调“从第一阶段起全流程自动推进”，优先按 `C`
4. 无论选择哪种模式，总控都必须先执行 Phase 1，不允许因为选择了自动链而跳过第一阶段。

### 类别一：逐阶段步进模式

对应值：

- `run_mode: step_by_step`

适用场景：

- 测试人员希望人工控制每一步推进节奏
- 每一阶段结束后先人工复核 JSON / Markdown 产物
- 只有确认当前阶段结论后，才进入下一阶段

运行机制：

1. 用户先声明运行模式。
2. 总控仍从 Phase 1 开始返回标准输入模板。
3. 用户补充当前阶段所需材料。
4. 总控路由到对应 Agent 执行。
5. 当前阶段完成后默认挂起，等待人工确认是否继续下一阶段。

人工介入点：

- 每个阶段结束后都可以人工停下复核
- 可人工调整下一阶段是否继续、继续到哪一步、是否更换输入材料

### 类别二：自动链

对应值：

- `run_mode: auto_chain`

自动链再细分为以下三条链路。三条链路都从 Phase 1 开始，只是自动化生效的起点不同。

#### 自动链 A：分析闭环

对应值：

- `run_mode: auto_chain`
- `auto_chain_mode: A`

适用场景：

- 希望整个流程从 Phase 1 开始
- 但第一阶段结束后，需要人工完成检测绕过、代理配置、抓包准备等前置操作
- 用户明确说明后续 `burp-mcp` / `yakit-mcp` 与 `ghidra-mcp` / `ida-mcp` 已接通
- 希望在这些前置动作完成后，由系统自动推进后续阶段

运行范围：

- Phase 1 人工确认
- Phase 2 -> Phase 3 -> Phase 4 -> Phase 5 -> Phase 6 自动推进

运行机制：

1. 总控先执行 Phase 1。
2. Phase 1 完成后默认挂起，等待人工完成抓包与其他前置准备。
3. 当用户确认“前置准备已完成”后，总控检查：
   - Phase 1 结果是否存在
   - 抓包材料是否真实可用或抓包 MCP 是否可用
   - Native 分析 MCP 或本地 native 材料是否可用
4. 即使用户已人工完成前置操作，本 Skill 仍必须先做前置检查，而不是直接跳过检测结论、抓包材料和 JNI / so 线索判断。
5. 若条件成立，则从 Phase 2 自动推进到 Phase 6。
6. 若关键条件不成立，则在缺失点暂停，并明确指出阻塞项。

人工介入点：

- 人工负责完成 Phase 1 之后的抓包与前置环境准备
- 一旦用户明确“前置准备已完成”，后续分析链默认自动推进

#### 自动链 B：验证闭环

对应值：

- `run_mode: auto_chain`
- `auto_chain_mode: B`

适用场景：

- 希望整个流程从 Phase 1 开始
- 前 1-3 阶段由人工主导并逐步确认
- 从 Phase 4 开始不再停下，而是自动推进到验证与报告阶段

运行范围：

- Phase 1 -> Phase 2 -> Phase 3 人工确认
- Phase 4 -> Phase 5 -> Phase 6 自动推进

运行机制：

1. 总控仍从 Phase 1 开始执行。
2. Phase 1、2、3 每一阶段结束后默认挂起，等待人工确认是否继续。
3. 进入 Phase 4 后，总控检查 `vuln_analysis.json`、`risk_matrix.json` 及相关前序材料是否存在。
4. 若条件成立，则从 Phase 4 自动进入 Phase 5，再自动进入 Phase 6。
5. 若验证设计或报告所需关键材料缺失，则暂停并提示缺失项。

人工介入点：

- 人工负责完成并确认前 1-3 阶段
- 人工可在进入 Phase 4 前取消自动链，改为全程逐步执行

#### 自动链 C：直通车

对应值：

- `run_mode: auto_chain`
- `auto_chain_mode: C`

适用场景：

- 测试人员在一开始就已经完成全部人工前置准备
- `jadx-mcp`、`burp-mcp` / `yakit-mcp`、`ghidra-mcp` / `ida-mcp` 已全部接通
- 抓包结果已经准备完毕
- 希望从 Phase 1 一路自动推进到 Phase 6

运行范围：

- Phase 1 -> Phase 2 -> Phase 3 -> Phase 4 -> Phase 5 -> Phase 6

运行机制：

1. 总控先执行 Phase 1，生成 APK 静态侦察结果。
2. 即使用户已经人工确认“抓包可行”，本 Skill 仍必须执行 Phase 1 的检测与入口识别，不能跳过样本画像和环境检测检查。
3. Phase 1 完成后，总控检查以下条件：
   - 抓包材料是否已存在或抓包 MCP 已可用
   - Native 分析 MCP 是否已接通
   - 是否具备进入 Phase 2 与 Phase 3 的最低输入
4. 若条件成立，则自动进入 `auto_chain`，从 Phase 2 连续推进到 Phase 6。
5. 若条件不成立，则在最早阻塞阶段暂停，并提示缺失条件。

人工介入点：

- 人工负责在启动前完成 MCP 接入、抓包准备与其他前置环境操作
- 一旦启动直通车模式，后续阶段默认自动推进，除非用户主动中断

### 自动衔接总原则

当用户明确表达”只输入一次，后续自动衔接””尽量连续推进到报告””从头跑到尾”时，总控应优先进入 `auto_chain` 模式。

在 `auto_chain` 模式下，总控必须：

1. 只在开始时收集一次最小必需输入。
2. 自动识别 `{output_dir}` 下已有中间产物，避免重复询问。
3. 在每个阶段结束后自动判断是否满足下一阶段最低条件。
4. 若满足条件，直接推进下一阶段。
5. 若不满足条件，只在真正缺失关键输入时暂停，并明确指出缺少什么。
6. 默认目标是从当前最早可行阶段推进到 Phase 6。

### auto_chain 质量门（Blackboard Hint）

在 `auto_chain` 模式下，每个 Phase 完成后，总控在推进下一 Phase 前执行以下检查：

1. 读取 `{output_dir}/session_blackboard.json`
2. 统计本 Phase 新写入的 Fact 数量（`from_phase = 当前 Phase` 且本次新增的条目）
3. 若新增 Fact 数量为 0，向 `hints` 数组追加一条 Hint 后继续推进：

```json
{
  “id”: “H001”,
  “type”: “stall_warning”,
  “from_phase”: “phase_N”,
  “target_phase”: “phase_N+1”,
  “message”: “上游 phase_N 未产生新的关键 Fact，下一阶段注意可能存在覆盖盲区”,
  “consumed”: false
}
```

4. **不阻断流程**，下一 Phase 启动时读取该 Hint 并在分析中注明”上游信息可能不完整”。
5. 若新增 Fact 数量 > 0，正常推进，无需写入 Hint。

默认原则如下：

- 默认只要求最基本输入项
- 不要求用户手动指定 `focus`
- 每个 Agent 默认完成本阶段的标准分析范围
- 已提供 `output_dir` 时，应优先落盘标准产物，而不是只输出对话摘要
- 若 `output_dir` 不存在，应先创建后再写入结果

补充说明：

- 当前仓库提供的是 Skill 总控规范、阶段 Agent 文档与本地索引脚本
- 当前并未内置独立 CLI、对话路由器或“一键跑完整 6 阶段”的程序入口
- 文档中提到的“总控返回模板并路由”，指的是上层 AI / Skill 按本文件执行，而不是仓库内已有单独命令行程序

## 模式启动模板

以下模板用于在任务开始时显式声明运行模式。

### 第一步：先声明模式

#### 模板 1：逐阶段步进模式

```text
run_mode: step_by_step
```

说明：

- 整个流程从第一步开始
- 每个阶段结束后默认挂起
- 等人工复核当前阶段产物后再继续

#### 模板 2：自动链 A（分析闭环）

```text
run_mode: auto_chain
auto_chain_mode: A
```

说明：

- 整个流程从第一步开始
- 第一阶段结束后，等待人工完成抓包准备、MCP 接通和其他前置动作
- 从第二阶段开始自动推进到第六阶段

#### 模板 3：自动链 B（验证闭环）

```text
run_mode: auto_chain
auto_chain_mode: B
```

说明：

- 整个流程从第一步开始
- 前 1-3 阶段默认人工确认
- 从第四阶段开始自动推进到第六阶段

#### 模板 4：自动链 C（直通车）

```text
run_mode: auto_chain
auto_chain_mode: C
```

说明：

- 整个流程从第一步开始
- 一开始就假定人工前置准备已全部完成
- 从第一阶段开始按自动链持续推进到第六阶段
- 直通车模式仍必须先执行 Phase 1 的样本侦察与检测确认

### 第二步：进入第一阶段

无论选择哪种模式，后续都应先进入第一阶段模板：

```text
step: 1
analysis_mode: local_source/jadx_mcp_session
target_dir: sample_target/decompiled
output_dir: analysis_runs/current_run
jadx_mcp: yes/no
```

补充说明：

- 若选择 `auto_chain_mode: A` 或 `auto_chain_mode: C`，可以在第一阶段模板之后继续补充抓包、Native、报告相关参数
- 若选择 `auto_chain_mode: B`，可先完成第一阶段，后续在进入第四阶段前补充验证与报告参数

## 使用时机

当用户要分析以下任一材料时，应启用本 Skill：

- 脱壳后、反编译后的 Android 目录
- Jadx 反编译结果
- Burp / Yakit / mitmproxy 抓包结果
- so 文件、JNI 线索、IDA / Ghidra 工程
- 前序阶段输出的 `raw_*.json`、`*_analysis.json`、`report.md`

## 总控执行铁律

1. 只允许按当前 6 个阶段执行，不得新增主阶段，不得修改顺序，不得偷换名称。
2. Phase 1 只分析**已经得到的脱壳后、反编译后的材料**，不得表述成“本 Skill 负责脱壳”。
3. 默认顺序推进；若用户明确要求“开始第 N 步”，且材料足够，可直接进入该步。
4. 默认采用“AI 主导分析 + 阶段脚本组合”：
   - 若当前 MCP 与材料足够，优先直接由 AI 执行阶段分析。
   - 当阶段走本地代码分析路径时，对应脚本默认执行，用于补齐批量索引与结构化原始命中。
   - 脚本不取代阶段分析结论。
5. 所有结论必须尽量绑定证据：
   - 代码路径
   - 行号或符号名
   - 抓包字段
   - JNI / so 伪代码片段
   - 或 `raw_*.json` 原始命中
6. 若当前阶段材料不足，只能指出缺失项和下一步需要准备什么，不能编造上下文。
7. 当前阶段完成后，应尽量产出可供下一阶段消费的标准产物。
8. Phase 1 在用户提供 `{target_dir}` 与 `{output_dir}` 时，默认应生成：
   - `file_inventory.json`
   - `tech_stack.json`
   - `entrypoints.json`
   - `env_guard_report.json`
   除非用户明确要求“只口头分析，不写文件”，否则不得只给对话总结而不落盘。
9. `{output_dir}` 一旦在第一次输入中确认，就应视为整个会话的统一输出根目录，不得在后续阶段随意变更，除非用户显式覆盖。
10. 统一输出根目录确认后，总控应默认采用按阶段分目录落盘的方式，而不是把所有产物平铺在根目录。

## 统一输入变量

总控优先识别并沿用以下变量：

- `{target_name}`：目标 App 名称
- `{apk_path}`：APK 路径，可选，仅用于补充样本元信息
- `{target_dir}`：脱壳后、反编译后的目录
- `{traffic_source}`：Burp / Yakit / mitmproxy 导出结果
- `{native_analysis_source}`：IDA / Ghidra 工程、伪代码或 so 样本路径
- `{output_dir}`：输出目录
- `raw_*.json`：脚本索引结果
- `*_analysis.json`：阶段分析结果
- `report.md` / `security_report.md`：最终报告类文档

## 会话级路径继承规则

一旦用户在当前任务中明确提供了以下路径或标识，总控必须把它们视为当前会话的默认上下文，后续阶段自动继承，除非用户显式覆盖：

- `{target_name}`
- `{apk_path}`
- `{target_dir}`
- `{traffic_source}`
- `{native_analysis_source}`
- `{output_dir}`

继承原则如下：

1. 用户第一次提供后，后续阶段不得重复要求再次填写相同路径。
2. 若用户只说“继续第二步”“继续第三步”“继续第四步”，总控应优先复用已记录的路径上下文。
3. 若用户启动 `4/5/6` 一体化模式，只需在开始时说明一次 `{output_dir}`，后续 Phase 5、Phase 6 自动继承。
4. 若用户重新提供了新的路径，则以最新值覆盖旧值。
5. 若当前阶段缺失某类路径，但可以从已有产物推断，例如已知 `{output_dir}` 中存在前序文件，则应优先从已知上下文推断，而不是再次要求用户手工填写。

## 统一输出目录规则

当用户第一次提供 `{output_dir}` 时，总控必须将其视为“统一输出根目录”，并在其下默认创建以下阶段子目录：

- `{output_dir}/step1`
- `{output_dir}/step2`
- `{output_dir}/step3`
- `{output_dir}/step4`
- `{output_dir}/step5`
- `{output_dir}/step6`

各阶段默认输出位置如下：

- Phase 1 -> `{output_dir}/step1/`
- Phase 2 -> `{output_dir}/step2/`
- Phase 3 -> `{output_dir}/step3/`
- Phase 4 -> `{output_dir}/step4/`
- Phase 5 -> `{output_dir}/step5/`
- Phase 6 -> `{output_dir}/step6/`

默认要求：

1. 第一阶段开始前，应先确保统一输出根目录存在。
2. 第一阶段执行时，应至少创建 `step1`，并建议同时预创建 `step2` 到 `step6`。
3. 后续阶段默认只向自己的阶段目录写入产物。
4. 后续阶段读取前序产物时，应优先从对应 `stepN/` 目录中读取，而不是要求用户重新指定文件路径。
5. 若仓库或用户已有旧版“平铺输出”结果，可兼容读取，但新的默认写入方式应为分阶段目录。

产物路径别名规则：

- 新流程默认将 Phase 1 产物解析为 `{output_dir}/step1/<artifact>`。
- Phase 2 产物默认解析为 `{output_dir}/step2/<artifact>`。
- Phase 3 产物默认解析为 `{output_dir}/step3/<artifact>`。
- Phase 4 产物默认解析为 `{output_dir}/step4/<artifact>`。
- Phase 5 产物默认解析为 `{output_dir}/step5/<artifact>`。
- Phase 6 产物默认解析为 `{output_dir}/step6/<artifact>`。
- 读取时可以兼容旧版根目录平铺产物，写入时默认不得再平铺到根目录。

推荐同时在根目录维护：

- `{output_dir}/analysis_state.json`

用于记录：

- 当前目标名称
- 会话级路径上下文
- 当前执行到的阶段
- 各阶段状态
- 各阶段产物路径

### `analysis_state.json` 角色说明

`analysis_state.json` 是整条 1-6 阶段工作流的状态记录文件，不负责保存具体分析结论，而负责记录当前任务的运行状态。

它的主要作用是：

- 记录当前任务的基本上下文，例如 `{target_name}`、`{target_dir}`、`{output_dir}`、运行模式和自动链模式；
- 记录当前执行到的阶段，以及每个阶段是 `pending`、`running`、`waiting_review`、`completed`、`blocked` 还是 `failed`；
- 为自动链 A / B / C 提供切换依据，判断当前是否允许自动进入下一阶段；
- 为中断恢复提供依据，使流程在挂起、阻塞或失败后能够从正确阶段继续；
- 索引各阶段输出目录，避免后续阶段重复询问前序产物路径。

职责划分如下：

- `agents/*.md` 负责执行各阶段分析；
- `analysis_state.json` 负责记录流程状态；
- 总控负责在阶段开始、结束、挂起、阻塞或失败时更新该文件。

最小字段建议包括：

- `target_name`
- `run_mode`
- `auto_chain_mode`
- `target_dir`
- `output_dir`
- `current_phase`
- `overall_status`
- `manual_ready`
- `phases`

### 阶段状态字段

总控与各阶段默认应统一使用以下状态值：

- `pending`：尚未开始
- `running`：正在执行
- `waiting_review`：当前阶段已执行完毕，等待人工确认
- `completed`：当前阶段已完成，可供下游消费
- `blocked`：缺少输入、条件不满足或被人工要求暂停
- `failed`：执行报错或结果不可用

默认规则如下：

- `run_mode = step_by_step` 时，阶段执行结束后默认写为 `waiting_review`
- 人工确认允许继续后，当前阶段再写为 `completed`
- `run_mode = auto_chain` 时，满足切换条件后，当前阶段写为 `completed` 并自动进入下一阶段
- 缺少关键材料时应写为 `blocked`，而不是假装继续推进
- 执行报错或关键产物无效时应写为 `failed`

### 自动链切换条件

自动链不是“时间到了自动跳转”，而是必须满足前置条件后才能进入下一阶段。

#### 链路 A

适用逻辑：

- Phase 1 正常执行并人工确认
- 人工完成抓包准备、MCP 接通和其他前置动作后
- 从 Phase 2 自动推进到 Phase 6

最小切换条件：

- `phase_1.status = completed`
- `manual_ready.traffic_ready = true`
- `manual_ready.mcp_burp_ready = true` 或 `{traffic_source}` 已提供
- `manual_ready.mcp_native_ready = true` 或 `{native_analysis_source}` 已提供

#### 链路 B

适用逻辑：

- Phase 1-3 由人工逐步确认
- 从 Phase 4 自动推进到 Phase 6

最小切换条件：

- `phase_1.status = completed`
- `phase_2.status = completed`
- `phase_3.status = completed`

#### 链路 C

适用逻辑：

- 从 Phase 1 开始执行
- 人工已在开始前完成全部环境、抓包与 MCP 准备
- 满足条件后持续自动推进至 Phase 6

最小切换条件：

- `analysis_mode = local_source` 或 `manual_ready.mcp_jadx_ready = true`
- `manual_ready.traffic_ready = true`
- `manual_ready.mcp_burp_ready = true` 或 `{traffic_source}` 已提供
- `manual_ready.mcp_native_ready = true` 或 `{native_analysis_source}` 已提供
- `phase_1.status = completed`

补充要求：

- 所有自动链都必须先执行 Phase 1，不得跳过样本侦察与检测确认
- 若任一条件不满足，应写入 `blocked` 并等待人工处理，而不是继续跳转
- 若产物存在但关键字段缺失，应优先进入 `blocked` 或 `failed`，不得视为 `completed`

## 主阶段文件

总控只能路由到以下文件：

1. `agents/agent-01-sample-recon.md`
2. `agents/agent-02-protocol-mapper.md`
3. `agents/agent-03-crypto-native-analyzer.md`
4. `agents/agent-04-crypto-vuln-analyzer.md`
5. `agents/agent-05-validation-designer.md`
6. `agents/agent-06-reporter.md`

## 阶段路由总表

| 阶段 | 用户常见说法 | 主工具方式 | 推荐 MCP | 最低输入要求 | 推荐产物 |
|---|---|---|---|---|---|
| Phase 1 | “开始第一步”“先做静态侦察”“我接好了 jadx-mcp” | `jadx-mcp` 会话分析或本地目录分析 | `jadx-mcp` 或无 MCP | `jadx_mcp=yes` 或 `{target_dir}` | 默认生成 `file_inventory.json` `tech_stack.json` `entrypoints.json` `env_guard_report.json` |
| Phase 2 | “开始第二步”“做流量与代码对齐”“我接好了 Burp MCP” | 抓包与代码映射 | Burp MCP / Yakit MCP | `{target_dir}` + 抓包 MCP 或 `{traffic_source}` | `api_endpoints.json` `protocol_map.json` `traffic_alignment.json` |
| Phase 3 | “开始第三步”“分析 so/JNI”“我接好了 ida-mcp”“我接好了 ghidra-mcp” | so / JNI 深挖 | `ida-mcp` / `ghidra-mcp` | `{native_analysis_source}` 或 so 相关材料 | `crypto_native_analysis.json` `jni_analysis.json` |
| Phase 4 | “开始第四步”“做漏洞筛查” | 汇总风险判断 | 无强制 MCP，必要时回调 `jadx-mcp` | 前 1-3 阶段结果至少一部分 | `vuln_analysis.json` `risk_matrix.json`，并尽量补齐 `secrets_report.json` `jsbridge_analysis.json` |
| Phase 5 | “开始第五步”“做最小验证”“设计 POC” | 验证方案设计与最小脚本模板生成 | Burp / Yakit MCP 可选 | Phase 4 结果 | `validation_cases.json` `test_plan.md` `repro_steps.md` `poc_scripts_index.json` |
| Phase 6 | “开始第六步”“汇总报告”“出报告” | 交付与汇总 | 无强制 MCP | 前 1-5 阶段结果 | `security_report.md` `findings.json` 等 |

说明：

- 表中的“推荐产物”默认应写入对应的 `stepN/` 目录。
- 若使用统一输出根目录，则不应再把这些文件直接平铺写到 `{output_dir}` 根下。

## 路由规则

### Rule 1：优先识别显式阶段

如果用户明确说了“第一步/第二步/第三步/第四步/第五步/第六步”，优先按用户指定阶段路由。

典型说法如下：

- “帮我进行第一步”
- “我现在连接上了 jadx-mcp，开始第一步”
- “我现在连接上了 Burp MCP，继续第二步”
- “我现在连接上了 ida-mcp，做第三步”
- “我现在连接上了 ghidra-mcp，做第三步”
- “帮我做第四步漏洞筛查”
- “帮我设计第五步最小验证 POC”
- “帮我汇总第六步报告”

若用户只说：

- “开始第一步”
- “开始第二步”
- “开始第三步”
- “开始第四步”
- “开始第五步”
- “开始第六步”

而没有提供足够输入材料，则总控**不得直接开始分析**，必须先返回该阶段的标准输入模板，等待用户补齐后再执行。

### Rule 2：没有显式阶段时，按材料推断最早可行阶段

若用户没有明确说第几步，则按以下顺序判断：

1. 若提供了 `{target_dir}`，优先以“本地反编译/解包源码目录模式”从 Phase 1 开始。
2. 若已连接 `jadx-mcp`，且目标样本已在 Jadx 中打开，优先以“`jadx-mcp` 会话模式”从 Phase 1 开始。
3. 若未提供 `{target_dir}`，且也未连接 `jadx-mcp`，必须明确说明当前仓库不负责自行脱壳或反编译，并要求用户补齐 `jadx-mcp` 或脱壳后、反编译后的目录。
4. 若只提供了抓包材料和代码目录，且 Phase 1 产物（`file_inventory.json` 或 `entrypoints.json`）已存在，则可从 Phase 2 开始；否则必须先补做 Phase 1。
5. 若只提供了 IDA / Ghidra / so 材料，且 Phase 1 产物已存在，则可从 Phase 3 开始；否则必须先补做 Phase 1。
6. 若已提供 `vuln_analysis.json`、`risk_matrix.json`，可从 Phase 5 或 Phase 6 开始。
7. 如存在歧义，优先选择更早的阶段，而不是擅自跳步。

注意：后续阶段的 Agent 会硬性检查前序阶段产物是否存在。若跳过 Phase 1 直接进入 Phase 2 / 3 / 4，Agent 会因缺少 `file_inventory.json`、`entrypoints.json` 等关键输入而终止。

### Rule 3：阶段前置检查

进入某一阶段前，总控至少做如下检查：

#### Phase 1

满足以下之一即可：

- 有 `{target_dir}`
- 有 `jadx_mcp = yes`，且目标样本已在 Jadx 中打开

Phase 1 必须显式区分两种分析方式：

- `analysis_mode = jadx_mcp_session`：通过 `jadx-mcp` 连接当前 Jadx 会话，对已打开样本进行分析
- `analysis_mode = local_source`：直接分析反编译源码或 APK 解包后的目录

若同时提供 `{output_dir}`，则本阶段默认要在该目录写出：

- `file_inventory.json`
- `tech_stack.json`
- `entrypoints.json`
- `env_guard_report.json`

若 `{output_dir}` 不存在，应先创建该目录，再写入结果文件。

#### Phase 2

必须有：

- `{target_dir}`
- Phase 1 产物至少存在一个：`{output_dir}/step1/entrypoints.json` 或 `{output_dir}/step1/file_inventory.json`（旧版根目录平铺产物可作兼容兜底）

且满足以下之一：

- 已连接 Burp MCP
- 已连接 Yakit MCP
- 提供 `{traffic_source}`

若缺少 Phase 1 产物，应先提示用户补做 Phase 1，而不是直接进入 Phase 2。

#### Phase 3

满足以下之一即可：

- 已连接 `ida-mcp`
- 已连接 `ghidra-mcp`
- 提供 `{native_analysis_source}`
- 提供可直接分析的 so / JNI 伪代码材料

若需要启用 SO 自动化链路，包括 `resolve_native_target.py` 收敛目标 so 以及对应引擎的 loader 自动导入（`ghidra_target_loader.py` → Ghidra，或 `ida_target_loader.py` → IDA，按 `native_mcp` 二选一，输入通用），则必须同时具备：

- 反编译代码上下文：`{target_dir}` 或 `jadx_mcp_session`
- APK 解包源码上下文：必须存在 APK 解包目录，且能访问其中的 `lib/<abi>/*.so`

缺少反编译代码上下文时，只能做单文件 native 分析，不能声称完成 Java -> JNI -> 业务字段链路分析。  
缺少 APK 解包源码目录时，可以收敛候选 so 名称，或分析用户显式提供的 `.so`，但不能声称已自动化拉取 so。显式 `.so` 只属于用户指定的 native 分析材料，不属于“自动化拉取”链路。

#### Phase 4

必须具备以下材料，否则立即终止：

1. `{output_dir}/step1/file_inventory.json` 必须存在（否则 Phase 1 未完成）
2. Phase 2 结果至少存在一个：
   - `{output_dir}/step2/protocol_map.json`
   - `{output_dir}/step2/traffic_alignment.json`
   - `{output_dir}/step2/api_endpoints.json`
3. Phase 3 或 Phase 1 native 线索至少存在一个：
   - `{output_dir}/step3/crypto_native_analysis.json`
   - `{output_dir}/step3/jni_analysis.json`
   - `{output_dir}/step1/raw_native_bridges.json`

若 `file_inventory.json` 缺失，立即终止并提示 Phase 1 未完成。  
若 Phase 2 与 Phase 3 关键输入均缺失，立即终止并提示先完成前序阶段。  
若只有部分 Phase 3 输入缺失但 Phase 2 结果齐全，可继续但须降低 native 相关结论置信度。

#### Phase 5

应已有以下之一：

- `vuln_analysis.json`
- `risk_matrix.json`
- 已确认或待验证的漏洞条目

#### Phase 6

应已有前 1-5 阶段中的至少一部分结果。

若不满足前置条件，总控必须指出缺少什么，而不是勉强推进。

### Rule 4：先给模板，再执行

当用户进入某个阶段但材料不全时，总控必须优先做以下动作：

1. 识别当前阶段编号
2. 返回该阶段的标准输入模板
3. 明确哪些字段是必填，哪些是可选
4. 等用户补齐后，再路由到对应 Agent

总控不得在材料明显缺失时直接进行“猜测式分析”。

## MCP 使用规范

### Phase 1：APK 静态侦察

- 首选：`jadx-mcp`
- 备选：无 MCP，直接使用 VS Code / 本地目录检索

本阶段支持两种分析方式：

- `jadx_mcp_session`：通过 `jadx-mcp` 直接读取 Jadx 当前已打开样本的 Manifest、类、方法、字符串、资源和调用线索
- `local_source`：直接分析已经脱壳、反编译或解包后的本地目录

本阶段默认至少应落盘以下产物：

- `file_inventory.json`
- `tech_stack.json`
- `entrypoints.json`
- `env_guard_report.json`

其中 `env_guard_report.json` 不能因为“没有发现明确检测逻辑”而省略。若当前阶段仅确认壳、风控 SDK、native 安全库等间接信号，也应明确写出：

- `confirmed`
- `suspected`
- `sdk_signal_only`
- `not_confirmed_yet`
- `not_observed`

等状态之一，供第二步抓包准备和第四步风险收口直接消费。

本阶段的核心不是“执行反编译工具”，而是分析已经得到的材料：

- Manifest
- 权限与组件
- 三方 SDK
- 环境对抗逻辑
- 签名 / 加密 / Token / JNI / WebView 入口

在已提供 `{output_dir}` 的前提下，本阶段默认应将关键结果结构化落盘，而不是只停留在对话中：

- `file_inventory.json`
- `tech_stack.json`
- `entrypoints.json`
- `env_guard_report.json`

若 `{output_dir}` 当前不存在，应先创建后再落盘。

### Phase 2：流量与代码对齐

- 首选：Burp MCP / Yakit MCP

本阶段主要借助 MCP 获取：

- 请求列表
- Header / Query / Body 字段
- 响应特征
- 登录、支付、资料、上传等分类流量

再与代码目录中的：

- BaseURL
- Retrofit / OkHttp
- request 包装器
- sign / token / data / timestamp 等构造逻辑

进行映射。

本阶段最重要的不是单纯得到接口列表，而是尽量吐出可供 Phase 4 直接消费的字段级证据，例如：

- `field_role`
- `location`
- `builder_path`
- `crypto_entry_candidate`
- `related_endpoint_group`
- `value_shape`
- `related_native_candidate`
- `replay_relevant`
- `matched_field_flows`

### Phase 3：SO 与 JNI 深度分析

- 首选：`ida-mcp` / `ghidra-mcp`
- 主路径：通过 `ida-mcp` 或 `ghidra-mcp` 直接驱动 IDA / Ghidra 分析 so 与 JNI 逻辑
- 本地脚本只负责补 JNI / bridge / loadLibrary 线索，不作为 so 逆向分析主手段

本阶段主要借助 MCP 获取：

- JNI 入口
- 交叉引用
- so 伪代码
- 关键函数上下文
- 算法与 native 防护逻辑线索

本阶段输出应尽量补齐可供 Phase 4 直接消费的还原字段，例如：

- `java_entry`
- `native_entry`
- `related_fields`
- `related_endpoints`
- `crypto_algorithm_candidate`
- `key_derivation`
- `iv_derivation`
- `salt_derivation`
- `input_order`
- `output_encoding`
- `restoration_confidence`

### Phase 4-6

- 无强制 MCP
- 主要消费前序结果
- 如需补证据，可回调前序 MCP，但不能借机改写当前 6 阶段主流程

其中：

- Phase 4 除 `vuln_analysis.json`、`risk_matrix.json` 外，若已存在 `raw_secrets.json` 或 WebView / JSBridge 相关线索，应尽量补齐 `secrets_report.json`、`jsbridge_analysis.json`
- Phase 6 应优先消费上述补齐后的结构化产物，而不是在报告阶段重新从零归并

## 脚本调用规范

默认规则如下：

- 当 Phase 1 采用 `local_source` 路径时，默认执行以下 4 个本地索引脚本：
  - `tools/scripts/endpoint_extractor.py`
  - `tools/scripts/secret_scanner.py`
  - `tools/scripts/native_bridge_indexer.py`
  - `tools/scripts/env_guard_indexer.py`
  - 上述 4 个脚本执行完毕后，自动运行 `tools/scripts/ai_summarizer.py` 读取其输出，生成 `ai_summary.json`
- 当 Phase 1 采用 `jadx_mcp_session` 路径时，不默认执行上述 4 个索引脚本和摘要生成，优先直接消费 `jadx-mcp` 上下文
- `tools/scripts/sign_rebuilder.py` 为 Phase 5 签名重算工具，读取 Phase 3/4 产物生成 sign，不属于 Phase 1 默认执行范围
- `tools/scripts/resolve_native_target.py`、`tools/scripts/ghidra_target_loader.py` 与 `tools/scripts/ida_target_loader.py` 继续按原有自动化逻辑执行，不受上述规则影响（后两者按 `native_mcp` 二选一，输入通用）

对应作用：

- `endpoint_extractor.py`：提取 URL、接口路径、BaseURL、deeplink、provider、network security config
- `secret_scanner.py`：提取密钥、Token、证书、云凭证、调试痕迹、内网线索
- `native_bridge_indexer.py`：提取 JNI、WebView、JSBridge、Native 加载与桥接线索，仅作为 Phase 3 的前置线索补充
- `env_guard_indexer.py`：提取 Root、模拟器、代理、SSL Pinning、Frida、签名校验、完整性校验线索
- `tools/frida/android_phase1_bypass.js`：提供授权环境下的 Root / 模拟器 / 代理 / SSL Pinning 基础绕过模板

## 阶段推进标准

总控在每阶段结束后，应尽量确认有标准产物可供下一阶段使用。

### Phase 1 结束后

至少应尽量具备：

- `file_inventory.json`
- `tech_stack.json`
- `entrypoints.json`
- `env_guard_report.json`

当采用 `local_source` 路径时，4 个索引脚本默认执行，还应产出：

- `raw_endpoints.json`
- `raw_secrets.json`
- `raw_native_bridges.json`
- `raw_env_guards.json`
- `ai_summary.json`

如识别到环境对抗逻辑，还应尽量补齐：

- `frida_bypass_plan.json`
- `frida/android_phase1_bypass.js`

### Phase 2 结束后

至少应尽量具备：

- `api_endpoints.json`
- `protocol_map.json`
- `traffic_alignment.json`

并优先补齐以下字段，供 Phase 4 直接消费：

- `field_role`
- `builder_path`
- `crypto_entry_candidate`
- `related_endpoint_group`
- `value_shape`
- `related_native_candidate`
- `replay_relevant`
- `matched_field_flows`

### Phase 3 结束后

至少应尽量具备：

- `crypto_native_analysis.json`
- `jni_analysis.json`

并优先补齐以下字段，供 Phase 4 直接消费：

- `java_entry`
- `native_entry`
- `related_fields`
- `related_endpoints`
- `crypto_algorithm_candidate`
- `key_derivation`
- `iv_derivation`
- `salt_derivation`
- `input_order`
- `output_encoding`
- `restoration_confidence`

### Phase 4 结束后

至少应尽量具备：

- `vuln_analysis.json`
- `risk_matrix.json`

并在 `vuln_analysis.json` 中尽量补齐：

- `crypto_findings`
- `signature_findings`
- `crypto_restoration`
- `packet_risks`
- `source_phase_2_fields`
- `source_phase_3_fields`
- `gap_filled_by_phase4`

### Phase 5 结束后

至少应尽量具备：

- `validation_cases.json`
- `test_plan.md`
- `repro_steps.md`
- `poc_scripts_index.json`

如材料充足，还应尽量具备：

- `pocs/{vuln_id}/validate_request.py`
- `pocs/{vuln_id}/runtime_observe.js`
- `pocs/{vuln_id}/README.md`

### Phase 6 结束后

至少应尽量具备：

- `security_report.md`
- `findings.json`
- 与主报告配套的附件和全量明细

## 标准对话协议

以下模板用于“用户只说开始第 N 步”时，总控应返回给用户的标准输入格式。

总控必须先返回模板，再执行分析。

路径约定：

- 面向用户提供的样本路径、输出路径可以是用户自己的真实路径
- Skill 内部引用仓库文件时，一律以 `ai-mobile-reverse-skills/` 为根目录使用相对路径
- 不应在 Skill 文档中写入任何个人机器的绝对目录

### Phase 1 输入模板

当用户说“开始第一步”时，总控应优先返回：

```text
step: 1
analysis_mode: jadx_mcp_session/local_source
target_dir: sample_target/decompiled
output_dir: analysis_runs/current_run
jadx_mcp: yes/no
```

字段说明：
- `analysis_mode`：必填，二选一
- `target_dir`：当 `analysis_mode = local_source` 时必填，指向反编译源码或 APK 解包目录
- `output_dir`：必填，结果输出目录；若不存在应先创建
- `jadx_mcp`：当 `analysis_mode = jadx_mcp_session` 时必填，应为 `yes`
- `apk_path`：可选，仅用于补充样本元信息

### Phase 2 输入模板

当用户说“开始第二步”时，总控应优先返回：

```text
step: 2
target_dir: sample_target/decompiled
output_dir: analysis_runs/current_run
mcp: burp/yakit/none
traffic_source: analysis_runs/current_run/traffic/traffic.json
```

字段说明：
- `target_dir`：必填
- `output_dir`：必填；若不存在应先创建
- `mcp`：可选，Burp MCP / Yakit MCP / none
- `traffic_source`：当未直接使用 MCP 时建议提供

### Phase 3 输入模板

当用户说“开始第三步”时，总控应优先返回：

```text
step: 3
target_dir: sample_target/decompiled
output_dir: analysis_runs/current_run
mcp: ida-mcp/ghidra-mcp/none
native_analysis_source: sample_target/native/sample.i64_or_sample.gpr_or_libsample.so
```

字段说明：
- `target_dir`：建议提供
- `output_dir`：必填；若不存在应先创建
- `mcp`：可选，`ida-mcp` / `ghidra-mcp` / none，由使用者自行选择分析后端
- `native_analysis_source`：当使用本地 so / IDA / Ghidra 材料时填写

### Phase 4 输入模板

当用户说“开始第四步”时，总控应优先返回：

```text
step: 4
output_dir: analysis_runs/current_run
allow_reanalyze_code: yes/no
```

字段说明：
- `output_dir`：必填；若不存在应先创建
- `allow_reanalyze_code`：可选，是否允许 Phase 4 回看 Java/smali/config

### Phase 5 输入模板

当用户说“开始第五步”时，总控应优先返回：

```text
step: 5
output_dir: analysis_runs/current_run
authorized_only: yes/no
```

字段说明：
- `output_dir`：必填；若不存在应先创建
- `authorized_only`：建议填写，默认为 yes
- Phase 5 默认除验证方案外，还应尽量为每个漏洞生成对应的最小 POC 脚本模板

### Phase 6 输入模板

当用户说“开始第六步”时，总控应优先返回：

```text
step: 6
output_dir: analysis_runs/current_run
target_name: 项目名称
report_type: brief/full
include_appendix: yes/no
```

字段说明：
- `output_dir`：必填；若不存在应先创建
- `target_name`：建议填写
- `report_type`：可选，简版或完整报告
- `include_appendix`：可选，是否附带附录与复现步骤

### 用法 1：从第一步开始

用户说：

`我现在连接上了 jadx-mcp，可以使用了，目标样本已经在 Jadx 里打开，输出目录是 analysis_runs/current_run，帮我进行第一步。`

总控应：

1. 识别为 Phase 1
2. 检查 `jadx_mcp = yes`，且目标样本已在 Jadx 中打开
3. 路由到 `agents/agent-01-sample-recon.md`
4. 直接按该文件执行

### 用法 2：进入第二步

用户说：

`继续第二步。`

总控应：

1. 识别为 Phase 2
2. 默认继承第一步已确认的 `{target_dir}` 与 `{output_dir}`
3. 若当前模式为 `step_by_step`，则只执行 Phase 2 并等待人工确认
4. 若当前模式为 `auto_chain + A/C`，则在满足条件后继续自动推进后续阶段
5. 路由到 `agents/agent-02-protocol-mapper.md`

### 用法 3：从第三步开始

用户说：

`我现在连接上了 ida-mcp，可以使用了，IDA 工程在 sample_target/native/sample.i64，帮我进行第三步。`

或：

`我现在连接上了 ghidra-mcp，可以使用了，Ghidra 工程在 sample_target/native/sample.gpr，帮我进行第三步。`

总控应：

1. 识别为 Phase 3
2. 检查 so / JNI / IDA 材料
3. 路由到 `agents/agent-03-crypto-native-analyzer.md`
4. 聚焦 JNI、算法、Native 防护与还原材料

### 用法 4：直接进入后续阶段

用户说：

`我已经完成前面几步，当前有 vuln_analysis.json 和 risk_matrix.json，帮我进行第五步。`

总控应：

1. 识别为 Phase 5
2. 确认已有漏洞条目
3. 路由到 `agents/agent-05-validation-designer.md`
4. 只做最小影响验证设计，不改写前序分析结论

### 用法 5：自动链 A

用户先说：

```text
run_mode: auto_chain
auto_chain_mode: A
```

然后继续提供第一步模板。总控应：

1. 先执行 Phase 1
2. 等待人工完成抓包准备、MCP 接通和其他前置动作
3. 进入 Phase 2 后自动推进到 Phase 6
4. 在自动推进过程中默认继承已确认的路径与上下文

### 用法 6：自动链 B / C

用户先说：

```text
run_mode: auto_chain
auto_chain_mode: B
```

或：

```text
run_mode: auto_chain
auto_chain_mode: C
```

然后继续提供第一步模板。总控应：

1. 对于 `B`：
   - 先执行 Phase 1-3
   - 从 Phase 4 开始自动推进到 Phase 6
2. 对于 `C`：
   - 从 Phase 1 开始按自动链持续推进到 Phase 6
3. 无论 `B` 还是 `C`，都不得跳过第一步的样本侦察与检测确认

## 与其他文档的关系

- 本文件负责总控、路由、前置检查、阶段推进标准
- `README.md` 负责面向使用者的完整使用手册
- `docs/MCP-INTEGRATION.md` 负责分阶段 MCP 接入规范
- `agents/*.md` 负责每个阶段的详细执行方法
- `tools/scripts/*.py` 负责本地索引、Native 目标收敛与 Ghidra / IDA 导入辅助；其中前 4 个索引脚本在 `local_source` 路径下默认执行
