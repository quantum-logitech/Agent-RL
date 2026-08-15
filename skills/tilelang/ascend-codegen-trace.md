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

**非 tilelang 任务不涉及本 Skill**（例如通用 git、文档、其他语言/框架的代码任务，不匹配）。本 Skill 是 tilelang 仓库专用能力，触发条件限定为"用户分析 tilelang 相关代码"。

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

### Codegen 与 Trace 工具用法
- 无 Ascend 硬件下获取 codegen：`lower(func, target="ascend")` 默认 `enable_device_compile=False`，只跑 `device_codegen_without_compile`，`.kernel_source` 即 Ascend C 源码；`TORCH_DEVICE_BACKEND_AUTOLOAD=0` 规避 torch_npu 报错
- lower_trace 工具用法：`enable(mode="html", trace_dir=..., codegen_output=...)` + `lower()` 组合生成 69-pass HTML 报告；`_without_compile` codegen FFI 不在包装列表，kernel_source 需显式落盘

### 无卡仿真执行（cannsim）
- [2026-08-15 12:47] 无硬件下跑通算子仿真流水图链路：`lower(target="ascend", enable_device_compile=False)` 拿源码 → 手动 `bisheng --npu-soc=<camodel 报告 soc> --cce-aicore-only` + `ld.lld -m aicorelinux -Ttext 0` 链接 aibin → 纯 ACL ctypes driver → `cannsim record` + `cannsim report`（来源: 本会话 maint/rmsnorm_sim/ 产出 trace_core0.json）
- [2026-08-15 12:47] bisheng 的 soc 名必须走 `--npu-soc=Ascend950PR_9589`（与 camodel driver stub `halGetSocVersion` 报告一致），`--npu-arch=` 不接受 soc 名（来源: `--npu-arch=Ascend950PR_9589` 报 Unsupported，`--npu-soc=` 成功）
- [2026-08-15 12:47] 纯 ACL launch 必须按 codegen 签名顺序打包参数（如 `main_kernel(RSTD, W, X, Y, eps)` 而非 Python 定义顺序），并设置 `kAclLaunchKernelAttrDynUbufSize`（kernel 用 `buf_dyn_shmem` 时否则挂起）；跳过 H2D memcpy 避免 camodel TDaw 崩溃（来源: 参数顺序/DynUbuf/H2D 三处踩坑）
- [2026-08-15 12:47] `cannsim report` 若报 `Failed determine BAR pipeline`，是 trace_tools 不识别 `SIMT_BAR`：在 `Ascend910_9599_ESL.py` 规则表补 `SIMT_BAR → FLOWCTRL`（来源: rmsnorm SimtVF 屏障指令触发 RuntimeError，补丁后 JSON 生成成功）
