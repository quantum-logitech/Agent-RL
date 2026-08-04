# Skill: ascend-codegen-trace

## 描述
在**无 Ascend 硬件**环境下，为 tilelang 的 Ascend 后端测试/内核生成 HTML pass 变换可视化（TIRX 经过各层 pass 的 before/after 变换）以及最终 Ascend C codegen 源码。该能力是 tilelang 仓库专用工具链（`tilelang/tools/lower_trace`）的用法沉淀，不适用于通用编程任务。

## 触发条件
**仅当同时满足以下条件时才调用本 Skill：**
- 用户请求涉及 **tilelang 仓库**内的代码分析（Ascend 后端、`testing/ascend/` 测试、`src/ascend/` pass、`tilelang/ascend/` 模块等）
- 用户要求查看/生成以下任一内容：
  - TIRX 经过各层 pass 的 HTML 变换可视化
  - Ascend C codegen 结果
  - 无硬件环境下验证 Ascend 后端行为

**非 tilelang 任务不涉及本 Skill**（例如通用 git、文档、其他语言/框架的代码任务，不匹配）。

## 行为规范
1. **确认环境**：运行 `python -c "import tilelang; print(tilelang.__version__)"` 确认 dev-root 构建可导入；若遇 `torch_npu` 报错，加 `TORCH_DEVICE_BACKEND_AUTOLOAD=0` 环境变量
2. **选择工具**：使用 `tilelang.tools.lower_trace`（`tilelang/tools/lower_trace/`），它 monkey-patch `Pass.__call__` 和 `PassPipeline.lower`，能捕获 Ascend 新 PassPipeline 架构下每个 pass 的 TIRX before/after
3. **写 driver 脚本**：
   - `os.environ.setdefault("TORCH_DEVICE_BACKEND_AUTOLOAD", "0")`
   - 从 `tilelang.tools.lower_trace` 导入 `enable`，调用 `enable(mode="html", trace_dir=<out_dir>, codegen_output=<out_dir>/codegen.cpp)`
   - 从 `tilelang.engine.lower` 导入 `lower`，调用 `lower(func, target="ascend")`
   - **注意**：`lower()` 默认 `enable_device_compile=False`，只走 `device_codegen_without_compile`，`.kernel_source` 即最终 Ascend C 源码，无需 bisheng/Ascend 硬件
   - **必须显式保存 `kernel_source` 到文件**：codegen FFI 的 `_without_compile` 变体（`target.build.tilelang_ascend_without_compile`）不在 lower_trace 的 `_CODEGEN_FFI_NAMES` 包装列表中，lower_trace 不会自动保存该路径的源码
4. **运行**：`TORCH_DEVICE_BACKEND_AUTOLOAD=0 python <driver>.py`；退出时 lower_trace 的 atexit hook 生成 `report.html`（位于 `<trace_dir>/<script_name>/report.html`，另有 `.tir` 快照在 `.run_records/run_*/pipeline_ascend/`）
5. **验证**：
   - 确认 `report.html` 存在且包含目标 pass（如 `LegalizeSimdMerging`）
   - 用测试文件的断言核对 codegen 输出（如 `::vadds(*((&(dst[0]))), src,`、`MODE_MERGING`、无 `simd_inst::vadds(`、顺序 `vlds < ::vadds < vsts`）

## 已沉淀的经验
- [2026-08-04 19:04] 本 Skill 是 tilelang 仓库专用能力，触发条件限定为"用户分析 tilelang 相关代码"；非 tilelang 任务不匹配本 Skill（来源: 用户修改提案——"该 skill 是 tilelang 的能力，应在用户分析 tilelang 相关代码时才调用"）
- [2026-08-04 19:04] 无 Ascend 硬件下获取 codegen：`lower(func, target="ascend")` 默认 `enable_device_compile=False`，只跑 `device_codegen_without_compile`，`.kernel_source` 即 Ascend C 源码；`TORCH_DEVICE_BACKEND_AUTOLOAD=0` 规避 torch_npu 报错（来源: 本次会话成功运行）
- [2026-08-04 19:04] lower_trace 工具用法：`enable(mode="html", trace_dir=..., codegen_output=...)` + `lower()` 组合生成 69-pass HTML 报告；`_without_compile` codegen FFI 不在包装列表，kernel_source 需显式落盘（来源: 本次会话产出 `maint/lower_trace_ascend_simd_merging/`，4 项测试断言全部通过）
