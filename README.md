# AI Mobile Reverse Skills

 面向移动安全分析场景的 6 阶段总控 Skill。用于统一调度 APK 静态侦察、流量与代码对齐、SO/JNI 深度分析、加密与漏洞综合分析、验证设计与报告交付流程。支持 JADX MCP、Burp/Yakit MCP、IDA/Ghidra MCP。

## 适用场景

- Android APK 静态逆向与安全画像
- 反编译代码、抓包结果、接口字段之间的联动分析
- JNI / SO / native 加密、签名、风控逻辑定位
- 弱加密、认证授权、组件安全、JSBridge、敏感信息等风险收口
- 授权测试环境下的最小验证方案与 POC 模板设计
- 移动端渗透测试报告和结构化 Findings 交付

## 目录结构

```text
.
├── README.md
├── LICENSE
└── ai-mobile-reverse-skills/
    ├── SKILL.md                         # Skill 总控入口与 6 阶段调度规则
    ├── USER-README.md                   # 用户快速说明
    ├── README.md                        # Skill 内部完整说明
    ├── agents/                          # 6 个阶段 Agent 执行规则
    ├── docs/                            # MCP 接入与状态模型
    ├── templates/                       # 报告、复现步骤、状态文件模板
    └── tools/
        ├── scripts/                     # 本地索引与 Native 自动化辅助脚本
        ├── frida/                       # 授权环境下的运行时观察/绕过模板
        └── poc_templates/               # 最小验证 POC 模板
```

## 如何使用这个仓库

这个仓库的核心入口是：

- `ai-mobile-reverse-skills/SKILL.md`

作为 Skill 包使用时，将 `ai-mobile-reverse-skills/` 放到支持 `SKILL.md` 的 Codex / AI Skill 搜索目录中，或在当前 workspace 中让 Codex 直接读取该目录下的 `SKILL.md`。仓库内部的阶段文档、脚本和模板都以 `ai-mobile-reverse-skills/` 为相对根目录引用。

如果只是想理解流程，先读 `ai-mobile-reverse-skills/USER-README.md`；如果要执行完整阶段规则，以 `ai-mobile-reverse-skills/SKILL.md` 和 `agents/` 下 6 个 Agent 文档为准。

## 6 阶段流程

| 阶段 | Agent | 目标 | 主要输出 |
|---|---|---|---|
| Phase 1 | SampleRecon | APK 静态侦察、技术栈识别、环境检测、敏感入口与 SO 线索初筛 | `file_inventory.json`、`tech_stack.json`、`entrypoints.json`、`env_guard_report.json` |
| Phase 2 | ProtocolMapper | 将抓包请求、接口字段、签名参数和代码实现对齐 | `api_endpoints.json`、`protocol_map.json`、`traffic_alignment.json` |
| Phase 3 | CryptoNativeAnalyzer | 围绕 Phase 2 线索分析 JNI / SO / native 加密和签名逻辑 | `crypto_native_analysis.json`、`jni_analysis.json` |
| Phase 4 | CryptoVulnAnalyzer | 综合前序证据，收口弱加密与高风险漏洞 | `vuln_analysis.json`、`risk_matrix.json`、`secrets_report.json`、`jsbridge_analysis.json` |
| Phase 5 | ValidationDesigner | 在授权环境下设计最小验证方案和 POC 模板 | `validation_cases.json`、`test_plan.md`、`repro_steps.md` |
| Phase 6 | Reporter | 汇总 Phase 1-5，生成交付报告和 Findings | `security_report.md`、`findings.json` |

所有模式都从 Phase 1 开始。自动链也不会跳过第一阶段。

## 运行模式

### 逐阶段步进

适合每一步都要人工复核、随时调整分析重点的场景。

```text
run_mode: step_by_step
```

特点：

- 每个阶段结束后默认挂起
- 人工确认当前阶段结果后再进入下一阶段
- 适合样本复杂、抓包前置条件不稳定、需要逐步判断的项目

### 自动链

适合前置材料比较完整，希望系统尽量连续推进到报告的场景。

```text
run_mode: auto_chain
auto_chain_mode: A/B/C
```

| 模式 | 自动化范围 | 适合情况 |
|---|---|---|
| A | Phase 1 人工确认，Phase 2-6 自动推进 | 第一阶段后需要人工完成代理、抓包、MCP 连接等准备 |
| B | Phase 1-3 人工确认，Phase 4-6 自动推进 | 前面人工深挖，后面漏洞收口、验证和报告自动化 |
| C | Phase 1-6 尽量自动推进 | 启动前已经准备好反编译目录、抓包、MCP 和 native 分析材料 |

自动链遇到关键条件缺失时会在最早阻塞阶段暂停，例如缺抓包结果、缺 `ghidra_root`、缺前序阶段产物等。

## 快速开始

推荐按“两段式”启动：先选模式，再进入第一阶段。

### 1. 选择模式

人工逐步执行：

```text
run_mode: step_by_step
```

或选择自动链：

```text
run_mode: auto_chain
auto_chain_mode: B
```

### 2. 提供第一阶段输入

分析本地反编译目录：

```text
step: 1
analysis_mode: local_source
target_dir: sample_target/decompiled
output_dir: analysis_runs/current_run
jadx_mcp: no
```

使用当前 Jadx MCP 会话：

```text
step: 1
analysis_mode: jadx_mcp_session
output_dir: analysis_runs/current_run
jadx_mcp: yes
```

字段说明：

- `analysis_mode`: `local_source` 表示分析本地反编译/解包目录，`jadx_mcp_session` 表示使用已打开的 Jadx MCP 会话
- `target_dir`: 反编译后的主分析目录；使用 Jadx MCP 时可不填
- `output_dir`: 统一输出目录，后续阶段默认继承
- `jadx_mcp`: 当前是否已经接通 `jadx-mcp`

## 后续阶段输入模板

如果第一阶段已经提供过 `target_dir` 和 `output_dir`，后续阶段默认继承路径，不需要每次重复填写。需要覆盖时再显式声明。

Phase 2：

```text
step: 2
mcp: burp/yakit/none
traffic_source: analysis_runs/current_run/traffic/traffic.json
```

Phase 3：

```text
step: 3
mcp: ida-mcp/ghidra-mcp/none
native_analysis_source: auto
```

Phase 4：

```text
step: 4
allow_reanalyze_code: yes
```

Phase 5：

```text
step: 5
authorized_only: yes
```

Phase 6：

```text
step: 6
target_name: demo_app
report_type: full
include_appendix: yes
```

## 输出目录约定

第一次提供 `output_dir` 后，它会作为整条分析链的统一根目录：

```text
analysis_runs/current_run/
├── analysis_state.json
├── step1/
├── step2/
├── step3/
├── step4/
├── step5/
└── step6/
```

`analysis_state.json` 是流程状态文件，用于记录运行模式、当前阶段、阻塞原因、人工准备状态、Native 配置和各阶段产物路径。它不保存漏洞结论，漏洞和证据仍写入对应阶段目录。

完整字段定义见：

- `ai-mobile-reverse-skills/docs/STATE-MODEL.md`

## MCP 接入

MCP 是工具上下文入口，不替代阶段判断。阶段结论仍以各 Agent 文档为准。

| MCP | 主要用途 | 典型阶段 |
|---|---|---|
| `jadx-mcp` | 读取 Jadx 当前样本的类、方法、资源、字符串和调用线索 | Phase 1、Phase 4 |
| Burp MCP / Yakit MCP | 读取抓包请求、Header、Body、响应摘要和接口场景 | Phase 2、Phase 5 |
| `ida-mcp` / `ghidra-mcp` | 分析 SO、JNI、伪代码、交叉引用和 native 加密逻辑 | Phase 3 |

完整规范见：

- `ai-mobile-reverse-skills/docs/MCP-INTEGRATION.md`

## 本地脚本

脚本层用于批量扫描和生成结构化线索，不直接输出最终漏洞结论。

Phase 1 走 `local_source` 时默认使用：

```bash
python3 ai-mobile-reverse-skills/tools/scripts/endpoint_extractor.py --target-dir sample_target/decompiled --output-dir analysis_runs/current_run
python3 ai-mobile-reverse-skills/tools/scripts/secret_scanner.py --target-dir sample_target/decompiled --output-dir analysis_runs/current_run
python3 ai-mobile-reverse-skills/tools/scripts/native_bridge_indexer.py --target-dir sample_target/decompiled --output-dir analysis_runs/current_run
python3 ai-mobile-reverse-skills/tools/scripts/env_guard_indexer.py --target-dir sample_target/decompiled --output-dir analysis_runs/current_run
```

Native 自动化辅助：

```bash
python3 ai-mobile-reverse-skills/tools/scripts/resolve_native_target.py --output-dir analysis_runs/current_run --target-dir sample_target/decompiled
python3 ai-mobile-reverse-skills/tools/scripts/ghidra_target_loader.py --output-dir analysis_runs/current_run --target-dir sample_target/apk_unpacked --project-dir analysis_runs/current_run/ghidra_projects --project-name sample_project --ghidra-root sample_tools/Ghidra/ghidra_x.y.z_PUBLIC
```

更多参数和输出 schema 见：

- `ai-mobile-reverse-skills/tools/scripts/README.md`

## Ghidra 自动导入说明

如果希望 Phase 3 自动从 APK 解包目录中收敛目标 SO 并导入 Ghidra，需要提前准备：

- `ghidra_root`: 本机 Ghidra 安装根目录
- APK 解包源码目录，且包含 `lib/<abi>/*.so`
- Phase 1 / Phase 2 已生成可用于判断目标 SO 的上下文产物

只有反编译目录时，系统可以收敛候选 SO 名称，但不能自动从 `lib/<abi>/*.so` 拉取文件。用户显式提供的 `.so` 可以作为 native 分析材料使用，但不等同于“自动化拉取 SO”。

## 常见问题

### 这个项目能不能直接 `python main.py` 跑起来？

不能。当前仓库没有封装独立 CLI。它的核心是 Codex Skill 规则、阶段 Agent 文档和辅助脚本。

### Phase 1 会自动脱壳吗？

不会。Phase 1 只分析已经脱壳、反编译或解包后的材料，或通过 `jadx-mcp` 读取当前已打开样本。

### 不接 MCP 能用吗？

可以做一部分。`local_source` 模式可以使用本地目录和脚本完成静态侦察；但抓包联动、IDA/Ghidra 分析等能力会受限。

### 自动链是不是完全无人值守？

不是。自动链会尽量连续推进，但遇到缺抓包、缺 MCP、缺 Ghidra 配置、缺关键产物时会暂停并说明阻塞项。

### 生成的 POC 能直接打真实目标吗？

不应该。Phase 5 的 POC 是授权测试环境下的最小验证模板，默认使用占位目标和人工补齐参数，目标是验证存在性，而不是扩大影响。


