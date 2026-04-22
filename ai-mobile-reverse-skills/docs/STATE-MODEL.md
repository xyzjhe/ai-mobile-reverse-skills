# State Model

本文档定义 `analysis_state.json` 的最小状态模型，用于支撑 `ai-mobile-reverse-skills` 的逐阶段步进模式与自动链模式。

## 文件位置

状态文件默认位于：

- `{output_dir}/analysis_state.json`

其中 `{output_dir}` 采用统一输出根目录约定，例如：

- `analysis_runs/current_run`

## 文件职责

`analysis_state.json` 不保存具体漏洞结论或协议分析结果，而负责记录当前任务的流程状态，包括：

- 当前目标与路径上下文
- 当前运行模式
- 当前执行阶段
- 各阶段状态
- 人工准备状态
- Native 运行时配置
- 各阶段产物位置

## 首次使用前的前置配置

若流程需要自动收敛 so 并通过 `ghidra_target_loader.py` 导入 Ghidra，则建议在任务开始前至少确认：

- `ghidra_root`

其中，`ghidra_root` 属于不能稳定自动猜测的本机路径，建议用户首次使用前手工确认一次。例如：

- `sample_tools/Ghidra/ghidra_x.y.z_PUBLIC`

后续流程默认继承，不应在每次进入第三阶段时重复要求填写。

同时，SO 自动化链路还需要同时具备：

- 反编译代码上下文：用于定位 Java 调用、JNI 入口和业务字段关系
- APK 解包源码上下文：必须存在 APK 解包目录，且能访问其中的 `lib/<abi>/*.so`

若缺少 APK 解包源码目录，`selected_native_target.json` 只能作为候选判断依据，不能进入“自动化拉取 so”链路。用户显式提供的 `.so` 可以作为 native 分析材料使用，但不属于自动化拉取。

其余 Native 运行时参数，例如：

- `ghidra_project_dir`
- `ghidra_project_name`
- `so_search_roots`
- `preferred_abis`

默认应由系统结合 `{output_dir}`、样本目录结构、目标 so 分布和默认 ABI 策略自动推导。

## 最小 JSON 模板

```json
{
  "target_name": "demo_app",
  "run_mode": "auto_chain",
  "auto_chain_mode": "B",
  "analysis_mode": "local_source",
  "target_dir": "sample_target/decompiled",
  "output_dir": "analysis_runs/current_run",
  "current_phase": "phase_1",
  "overall_status": "running",
  "manual_ready": {
    "traffic_ready": false,
    "mcp_jadx_ready": false,
    "mcp_burp_ready": false,
    "mcp_native_ready": false
  },
  "native_runtime": {
    "native_mcp": "ghidra-mcp",
    "native_analysis_source": "auto",
    "ghidra_root": "sample_tools/Ghidra/ghidra_x.y.z_PUBLIC",
    "ghidra_project_dir": "",
    "ghidra_project_name": "",
    "so_search_roots": [],
    "preferred_abis": []
  },
  "phases": {
    "phase_1": {
      "status": "pending",
      "step_dir": "analysis_runs/current_run/step1",
      "required_outputs": [
        "file_inventory.json",
        "tech_stack.json",
        "entrypoints.json",
        "env_guard_report.json"
      ],
      "actual_outputs": [],
      "notes": ""
    },
    "phase_2": {
      "status": "pending",
      "step_dir": "analysis_runs/current_run/step2",
      "required_outputs": [
        "api_endpoints.json",
        "protocol_map.json",
        "traffic_alignment.json"
      ],
      "actual_outputs": [],
      "notes": ""
    },
    "phase_3": {
      "status": "pending",
      "step_dir": "analysis_runs/current_run/step3",
      "required_outputs": [
        "crypto_native_analysis.json",
        "jni_analysis.json"
      ],
      "actual_outputs": [],
      "notes": ""
    },
    "phase_4": {
      "status": "pending",
      "step_dir": "analysis_runs/current_run/step4",
      "required_outputs": [
        "vuln_analysis.json",
        "risk_matrix.json"
      ],
      "actual_outputs": [],
      "notes": ""
    },
    "phase_5": {
      "status": "pending",
      "step_dir": "analysis_runs/current_run/step5",
      "required_outputs": [
        "validation_cases.json",
        "test_plan.md",
        "repro_steps.md"
      ],
      "actual_outputs": [],
      "notes": ""
    },
    "phase_6": {
      "status": "pending",
      "step_dir": "analysis_runs/current_run/step6",
      "required_outputs": [
        "security_report.md",
        "findings.json"
      ],
      "actual_outputs": [],
      "notes": ""
    }
  }
}
```

## 阶段状态字段

推荐统一使用以下状态值：

- `pending`
- `running`
- `waiting_review`
- `completed`
- `blocked`
- `failed`

### 含义

- `pending`：阶段尚未开始
- `running`：阶段正在执行
- `waiting_review`：阶段已执行完，等待人工确认
- `completed`：阶段已完成，可供下游消费
- `blocked`：缺少材料、条件不满足或人工要求暂停
- `failed`：执行报错或结果不可用

## 模式行为

### `step_by_step`

- 每个阶段执行结束后默认写为 `waiting_review`
- 人工确认后再写为 `completed`
- 总控仅在人工确认后进入下一阶段

### `auto_chain`

- 阶段完成并满足切换条件后，当前阶段写为 `completed`
- 总控自动更新 `current_phase` 并进入下一阶段
- 若切换条件不满足，则写为 `blocked`

## Native 运行时配置字段

### `native_runtime.native_mcp`

- `ghidra-mcp`
- `ida-mcp`
- `none`

### `native_runtime.native_analysis_source`

- `auto`
- `selected_target_json`
- 显式 so / IDA / Ghidra 工程路径，仅作为用户指定的 native 分析材料，不代表自动化拉取 so

### `native_runtime.ghidra_root`

Ghidra 安装根目录。  
这是自动导入 so 到 Ghidra 时必须提前确认的本机路径，默认不应由系统凭空猜测。

### `native_runtime.ghidra_project_dir`

Ghidra 项目目录。若未单独指定，应默认放在：

- `analysis_runs/current_run/ghidra_projects`

### `native_runtime.ghidra_project_name`

Ghidra 项目名。若未单独指定，应默认按当前任务名或输出目录名自动生成。

### `native_runtime.so_search_roots`

用于自动查找 so 的根目录列表。若未单独指定，应优先根据样本解包目录自动推导，通常指向解包目录下的 `lib/`。

### `native_runtime.preferred_abis`

用于在同名 so 存在多份时确定优先顺序。若未单独指定，默认可按以下顺序处理：

- `arm64-v8a`
- `armeabi-v7a`

## 自动链切换条件

### 链路 A

目标：

- Phase 1 人工确认
- Phase 2-6 自动推进

最小条件：

- `phase_1.status = completed`
- `manual_ready.traffic_ready = true`
- `manual_ready.mcp_burp_ready = true` 或 `{traffic_source}` 已提供
- `manual_ready.mcp_native_ready = true` 或 `{native_analysis_source}` 已提供

### 链路 B

目标：

- Phase 1-3 人工确认
- Phase 4-6 自动推进

最小条件：

- `phase_1.status = completed`
- `phase_2.status = completed`
- `phase_3.status = completed`

### 链路 C

目标：

- 从 Phase 1 开始持续自动推进到 Phase 6

最小条件：

- `analysis_mode = local_source` 或 `manual_ready.mcp_jadx_ready = true`
- `manual_ready.traffic_ready = true`
- `manual_ready.mcp_burp_ready = true` 或 `{traffic_source}` 已提供
- `manual_ready.mcp_native_ready = true` 或 `{native_analysis_source}` 已提供
- `phase_1.status = completed`

## 更新职责

职责划分如下：

- `agents/*.md`：负责各阶段分析执行
- `analysis_state.json`：负责记录流程状态
- `SKILL.md` 总控：负责创建与更新状态文件

正常使用时，测试人员不需要手工维护本文件。它应由流程在阶段开始、结束、挂起、阻塞或失败时自动更新。
