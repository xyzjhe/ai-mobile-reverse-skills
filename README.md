# AI Mobile Reverse Skills

 面向移动安全分析场景的 6 阶段总控 Skill。用于统一调度 APK 静态侦察、流量与代码对齐、SO/JNI 深度分析、加密与漏洞综合分析、验证设计与报告交付流程。支持 JADX MCP、Burp/Yakit MCP、IDA/Ghidra MCP。

## 一、适用场景

- Android APK 静态逆向与安全画像
- 反编译代码、抓包结果、接口字段之间的联动分析
- JNI / SO / native 加密、签名、风控逻辑定位
- 弱加密、认证授权、组件安全、JSBridge、敏感信息等风险收口
- 授权测试环境下的最小验证方案与 POC 模板设计
- 移动端渗透测试报告和结构化 Findings 交付

## 二、架构设计
主要由以下几个核心模块构成：
- 根总控 SKILL.md：作为整个流程的调度中心，负责意图拦截、标准输入模板返回、任务路由分发以及阶段执行规则的约束 。
- 6 个阶段 Agent：针对移动安全分析全生命周期定制的规则集，涵盖了从第一阶段的 APK 静态侦察到第六阶段的安全报告汇总 。
- MCP 分阶段接入规范：明确了 Jadx-MCP（静态分析）、Burp/Yakit-MCP（分析）以及 Ghidra/IDA-MCP（Native 深挖）在不同阶段的接入与调用标准 。
- 本地索引脚本 (Python 探针)：负责高覆盖率盲扫的工具集，包括接口提取 (endpoint_extractor.py)、硬编码扫描 (secret_scanner.py)、JNI 桥接索引以及目标 SO 自动收敛与加载工具 。
- 统一的结构化输出设计：通过规范化的 JSON 和 Markdown 产物，确保前序阶段的分析线索能够被后续 Agent 自动继承与深度联动 。
```text
ai-mobile-reverse-skills/
├── SKILL.md                                  # 总控入口：阶段路由、输入模板、执行规则
├── README.md                                 # 使用手册：流程说明、完整示例、交互方式
├── agents/                                   # 六阶段 Agent 规则集
│   ├── agent-01-sample-recon.md              # 第一阶段：APK 静态侦察
│   ├── agent-02-protocol-mapper.md           # 第二阶段：流量与代码对齐
│   ├── agent-03-crypto-native-analyzer.md    # 第三阶段：SO / JNI 深度分析
│   ├── agent-04-crypto-vuln-analyzer.md      # 第四阶段：弱加密与高风险漏洞筛查
│   ├── agent-05-validation-designer.md       # 第五阶段：最小验证 POC 设计
│   └── agent-06-reporter.md                  # 第六阶段：安全报告汇总
├── docs/                                     # 阶段接入与补充文档
│   └── MCP-INTEGRATION.md                    # MCP 分阶段接入规范
├── templates/                                # 报告与复现模板
│   ├── mobile-reverse-report-template.md     # 移动安全报告模板
│   └── repro-steps-template.md               # 复现步骤模板
├── tools/                                    # 配套工具与模板资源
│   ├── frida/                                # Frida 相关模板
│   │   ├── README.md                         # Frida 模板说明
│   │   └── android_phase1_bypass.js          # Phase 1 运行时准备 / 观察模板
│   ├── poc_templates/                        # POC / 验证模板
│   │   ├── README.md                         # POC 模板说明
│   │   ├── CASE_README.md.tmpl               # 单漏洞验证说明模板
│   │   ├── frida_runtime_observe.js.tmpl     # Frida 运行时观察模板
│   │   └── python_http_validation.py.tmpl    # HTTP 验证脚本模板
│   └── scripts/                              # 本地索引脚本
│       ├── README.md                         # 脚本说明与 sample schema
│       ├── endpoint_extractor.py             # 接口 / URL / 字段线索提取
│       ├── env_guard_indexer.py              # Root / 代理 / Frida / SSL Pinning 线索提取
│       ├── native_bridge_indexer.py          # JNI / JSBridge / native crypto 线索提取
│       ├── secret_scanner.py                 # 硬编码密钥 / Token / 证书 / 云凭证扫描
│       ├── resolve_native_target.py          # 自动收敛第三阶段优先分析的 SO 目标
│       └── ghidra_target_loader.py           # 自动导入目标 SO 到 Ghidra 项目
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7c754161e2d2472c88d6cf4a08d196d6.png#pic_center)


## 三、如何使用这个仓库

这个仓库的核心入口是：

- `ai-mobile-reverse-skills/SKILL.md`

作为 Skill 包使用时，将 `ai-mobile-reverse-skills/` 放到支持 `SKILL.md` 的 Codex / AI Skill 搜索目录中，或在当前 workspace 中让 Codex 直接读取该目录下的 `SKILL.md`。仓库内部的阶段文档、脚本和模板都以 `ai-mobile-reverse-skills/` 为相对根目录引用。

如果只是想理解流程，先读 `ai-mobile-reverse-skills/USER-README.md`；如果要执行完整阶段规则，以 `ai-mobile-reverse-skills/SKILL.md` 和 `agents/` 下 6 个 Agent 文档为准。

## 四、阶段流程说明

| 阶段 | Agent | 目标 | 主要输出 |
|---|---|---|---|
| Phase 1 | SampleRecon | APK 静态侦察、技术栈识别、环境检测、敏感入口与 SO 线索初筛 | `file_inventory.json`、`tech_stack.json`、`entrypoints.json`、`env_guard_report.json` |
| Phase 2 | ProtocolMapper | 将抓包请求、接口字段、签名参数和代码实现对齐 | `api_endpoints.json`、`protocol_map.json`、`traffic_alignment.json` |
| Phase 3 | CryptoNativeAnalyzer | 围绕 Phase 2 线索分析 JNI / SO / native 加密和签名逻辑 | `crypto_native_analysis.json`、`jni_analysis.json` |
| Phase 4 | CryptoVulnAnalyzer | 综合前序证据，收口弱加密与高风险漏洞 | `vuln_analysis.json`、`risk_matrix.json`、`secrets_report.json`、`jsbridge_analysis.json` |
| Phase 5 | ValidationDesigner | 在授权环境下设计最小验证方案和 POC 模板 | `validation_cases.json`、`test_plan.md`、`repro_steps.md` |
| Phase 6 | Reporter | 汇总 Phase 1-5，生成交付报告和 Findings | `security_report.md`、`findings.json` |

所有模式都从 Phase 1 开始。自动链也不会跳过第一阶段。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/625a635a779845ea8461726b75abae9e.png)

## 五、MCP 接入说明

MCP 是工具上下文入口，不替代阶段判断，本skills使用以下mcp。

| MCP | 主要用途 | 典型阶段 |
|---|---|---|
| `jadx-mcp` | 读取 Jadx 当前样本的类、方法、资源、字符串和调用线索 | Phase 1、Phase 4 |
| Burp MCP / Yakit MCP | 读取抓包请求、Header、Body、响应摘要和接口场景 | Phase 2、Phase 5 |
| `ida-mcp` / `ghidra-mcp` | 分析 SO、JNI、伪代码、交叉引用和 native 加密逻辑 | Phase 3 |

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2d222bef41924711b782fbac97bd80cf.png)

完整规范见：

- `ai-mobile-reverse-skills/docs/MCP-INTEGRATION.md`
## 六、运行模式

### 5.1、逐阶段步进

适合每一步都要人工复核、随时调整分析重点的场景。

```text
run_mode: step_by_step
```

特点：

- 每个阶段结束后默认挂起
- 人工确认当前阶段结果后再进入下一阶段
- 适合样本复杂、抓包前置条件不稳定、需要逐步判断的项目

### 5.2、自动链

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

## 六、快速开始

推荐按“两段式”启动：先选模式，再进入第一阶段。

### 6.1、选择模式

人工逐步执行：

```text
run_mode: step_by_step
```

或选择自动链：

```text
run_mode: auto_chain
auto_chain_mode: B
```

### 6.2、提供第一阶段输入

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
 
## 七、更新说明

### 2026/4/23
初版发布
### 2026/5/20
- 新增 `ai_summarizer.py`：4 个索引脚本执行后自动生成压缩摘要，减少 AI token 消耗
- 新增 `sign_rebuilder.py`：支持 17 种算法和 pipeline 链式组合，Phase 5 直接生成 sign 复现请求
- `ghidra_target_loader.py` 支持 macOS 和 Windows，用户需提前填写 `ghidra_root`


