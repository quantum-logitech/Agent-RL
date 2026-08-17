# Skill: ascend-cannsim-sim

## 描述
在**无 Ascend 硬件**环境下，用 CANN 自带的 **cannsim**（AscendOps simulation environment，即用户常说的 "msprof op simulator"）对 tilelang 编译的 Ascend kernel 做仿真执行，并生成算子**流水图**（Chrome Tracing JSON）的完整协议。该能力是 tilelang 仓库专用工具链的用法沉淀。

## 触发条件
**仅当同时满足以下条件时才调用本 Skill：**
- 用户请求涉及 **tilelang 仓库**内 Ascend kernel 的**仿真运行**（无真实硬件）
- 用户要求以下任一内容：
  - 无卡环境下跑通算子的仿真（"op simulator"、"仿真"、"流水图"、"pipeline"）
  - 生成算子执行流水图（Chrome Tracing / trace JSON）
  - cannsim / camodel / 仿真性能分析

**非仿真类任务不匹配**（例如只看 codegen 源码/HTML pass trace 走 `ascend-codegen-trace`；通用编程任务不涉及本 Skill）。

## 行为规范
1. **环境准备**：
   - `source <cann>/set_env.sh`（设 `ASCEND_TOOLKIT_HOME` 等）；`export CANNSIM_NO_DELAY=1` 加速
   - `export TORCH_DEVICE_BACKEND_AUTOLOAD=0` 规避 torch_npu 自动加载报错
   - tilelang 走 dev-root build：`export LD_LIBRARY_PATH=<repo>/build/lib:$LD_LIBRARY_PATH`
   - 若 `import tilelang` 报 `libcuda.so.1` 缺失：`ln -sf /usr/local/cuda/lib64/stubs/libcuda.so build/lib/libcuda.so.1`
   - 若报 `MaterializeScheduleUnits` 等 FFI 符号缺失：`cmake --build build -j$(nproc)`（库比源码旧）
2. **获取 Ascend C 源码**：用 `tilelang.engine.lower(func, target="ascend", enable_device_compile=False)`（不要用 `tilelang.compile`，它走 `--npu-arch=` 不接受 soc 名），`.kernel_source` 即源码
3. **编译 aibin（手动 bisheng）**：
   - 先确认 camodel 报告的 soc：跑一次 `cannsim record` 后查 `cannsim.log` 中 `halGetSocVersion`（如 `Ascend950PR_9589`）
   - `bisheng -O2 -fPIC -std=c++20 --npu-soc=<该soc名> -I<repo>/src --cce-aicore-only tl_kernel.asc -o tl_kernel.rel.o`（**soc 名只能走 `--npu-soc=`**）
   - `ld.lld -m aicorelinux -Ttext 0 --no-mmap-output-file tl_kernel.rel.o -o kernel.aibin`
4. **纯 ACL host driver（ctypes 调 libascendcl.so，绕开 torch_npu）**：
   - torch_npu soc 白名单硬编码（2.7.1 只到 Ascend910），950 系必失败 → 不用 torch_npu
   - 调用序列：`aclInit → aclrtSetDevice(0) → aclrtCreateStream → aclrtMalloc(各 buffer) → aclrtBinaryLoadFromData → aclrtBinaryGetFunction("main_kernel") → aclrtLaunchKernelWithHostArgs → aclrtSynchronize* → aclFinalize`
   - **参数顺序**：以 codegen 输出签名为准（如 `main_kernel(RSTD, W, X, Y, eps)`），不是 Python 定义顺序
   - **必须设置 dyn_ubuf_size**（`kAclLaunchKernelAttrDynUbufSize=2`，值由 .asc 最大 `buf_dyn_shmem` 偏移推算并 512 对齐），否则 kernel 挂起
   - **跳过 H2D memcpy**：camodel DAW 不映射 host 堆地址，memcpy 触发 `TDaw::getTgtEntry assertion`；仿真只看流水线，无需真实数据
5. **cannsim record 仿真**：`cannsim record "python -u launch.py kernel.aibin" -s Ascend950 -o <out> -g`
   - 成功标志：日志 `TASK_BEGIN ... TASK_DONE`
   - **用小 shape 先验证**（如 batch=64, d=1024，每核 1 次迭代）；大 shape 仿真极慢（~0.3 KHz）且 instr.bin 膨胀到 GB 级
6. **cannsim report 生成流水图**：`cannsim report -e <archive> -o <archive>/report`
   - 成功标志：`Json Saved at: .../report/trace_core0.json`（Chrome Tracing 格式）
   - report 三级 fallback：cannprof → BiProfRunner → trace_tools；cannprof 缺失/BiProfRunner import 损坏时自动降级，trace_tools 纯 Python 也能出 JSON
   - 若报 `Failed determine BAR pipeline`：trace_tools 不识别 `SIMT_BAR`（SimtVF 屏障指令）→ 在 `<cann>/python/site-packages/cannsim/trace_tools/model2trace/Ascend910_9599_ESL.py` 规则表补 `(re.compile(r"SIMT_BAR"), pattern_none, lambda ...: DavinciV220PipelineType.PID_PIPELINE_FLOWCTRL)`
7. **验证**：确认 `trace_core0.json` 为合法 Chrome Tracing JSON，含 `X` 事件（指令执行区间）覆盖多条 pipeline（SCALAR/RVECSU/RVECEX/RVECST/FLOWCTRL 等）
8. **保留算子源码（每个仿真必做）**：在仿真 archive 下建 `kernel_src/` 目录，随产物一并保留：
   - `.asc`（AscendC 源码，`lower(target="ascend", enable_device_compile=False)` 的 `.kernel_source`）
   - `.aibin`（编译产物）
   - `compile_*.py`（生成 .asc/.aibin 的脚本）
   - `launch_*.py`（ACL driver）
   - **Python 级 tilelang 算子源码**（如 `flash_attention/core.py` 的 `flash_attention_fwd`，按实际来源复制）
   - 附 `README.md` 记录算子签名、shape、dyn_ubuf、soc 及实际 launch 配置（有 grid 的算子按需记录 num_blocks）
   - 原因：同名 `.aibin`/`.asc` 会被后续 `compile_*.py` 重跑覆盖，不保留源码则无法追溯已仿真算子的原始定义（来源: 2026-08-17 会话，sim_mha1 的 aibin 被 16:44 重编译覆盖，只能靠 kernel_src/ 追溯）
9. **record 退出码非 0 不代表内核失败**：`cannsim record` 可能报 `Simulation FAILED` / `USER_APP failed (exit=-11)`（用户程序退出阶段段错误），但只要 `cannsim.log` 含 `TASK_BEGIN ... TASK_DONE` 且 `MHA_LAUNCH_OK`，产物完整；直接对 archive 跑 `cannsim report` 仍可生成流水图（来源: 2026-08-17 sim_mha1_rerun 实测，record 判 FAILED 但 report 产出 175MB trace_core0.json / 628K 事件）
## 已沉淀的经验
- [2026-08-15 12:47] 完整链路：lower(不编译) → bisheng --npu-soc + ld.lld → 纯 ACL driver → cannsim record → cannsim report 产出流水图 JSON（来源: 本会话 maint/rmsnorm_sim/ 全链路跑通，trace_core0.json 128KB/412 事件）
- [2026-08-15 12:47] 大 shape 仿真陷阱：batch=4096/d=4096 时 instr.bin 增长到 3.6GB 且 10 分钟跑不完；先用 batch=64/d=1024 验证链路（来源: 本会话实测）
- [2026-08-15 12:47] 纯 ACL driver 三处必踩坑：codegen 参数重排、dyn_ubuf_size 未设置导致挂起、H2D memcpy 触发 TDaw 崩溃（来源: 本会话逐一修复）
- [2026-08-15 12:47] trace_tools SIMT_BAR 补丁是 Ascend950 系 SimtVF kernel 出流水图的必要修复（来源: 补丁后 report 从 RuntimeError 变为 Json Saved）
- [2026-08-17 15:21] **每个仿真必须保留算子源码**：archive 下 `kernel_src/` 保存 `.asc` + `.aibin` + compile/launch 脚本 + **Python 级 tilelang 算子源码**（core.py 等）+ README；同名文件会被后续 compile 重跑覆盖，不保留无法追溯（来源: 用户明确建议，sim_mha1_rerun 已落实）
- [2026-08-17 15:21] record 判 FAILED（USER_APP exit=-11）但 TASK_DONE 出现时，产物可用，直接跑 report 仍出流水图（来源: sim_mha1_rerun 实测，trace_core0.json 175MB/628K 事件/13 pipeline）
