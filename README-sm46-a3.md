# sm46-a3 — 为 RTX 4070 编译三元 Bonsai 引擎

本分支用于回答一个问题：**把 ninfer 的 sm_89 引擎按本卡（RTX 4070，cc 8.9 / SM 46）重编，
再加上 A3 调度补丁，到底能快多少？**

## 分支内容

| 路径 | 说明 |
|---|---|
| `engine-sm89/` | 自包含引擎源码树（已内置三元算子，无需再打补丁） |
| `engine-sm89/ffmpeg/{include,lib}` | FFmpeg 头文件 + MSVC 导入库，**编译必需** |
| `patches-a3/` | A3 调度的 4 个**完整替换文件**（不是 diff） |
| `.github/workflows/build-sm46.yml` | 编译流水线 |

`engine-sm89/` 是随包提供的源码快照，与上游 fork 不同源，因此**本分支不通过 `git clone` 拉源码**，
而是把整棵树提交进来（已去掉 `ffmpeg/bin`、`ffmpeg/doc`、`tools/freq_corpus`、`.codex`，
31 MB）。编译所需的 FFmpeg 头文件与导入库保留；FFmpeg 运行时 DLL 不需要进仓库。

## 两个改造项

| 项 | 是什么 | 怎么生效 |
|---|---|---|
| **E1** | `-DNINFER_SM_COUNT=46`。`device.h` 里 `kTargetSmCount` 出厂默认 128（RTX 4090），被当作 `gridDim.y` 上界 `kTargetSmCount / KVHeads` 使用。SM 数比实际多 ⇒ 启动的 block 远超卡能并发驻留的数量 ⇒ 尾效应浪费。 | **编译期**宏 |
| **A3** | 三元 GEMM 的 token 轴铺进 `blockIdx.y`。prefill 时 T 很大但网格只铺了输出行轴 ⇒ 并行度上不去。包方实测 prefill +31.1% 且逐位不变。 | **运行时** env `NINFER_TERNARY_TOKEN_GRID=1`（`0` 为对照臂） |

## 三个臂

| 臂 | 源码 | SM_COUNT | 运行时 | 用途 |
|---|---|---|---|---|
| `baseline` | 树原样 | 128（出厂默认） | — | **对照基线**，没有它就只能报新值、无法判断提升 |
| `sm46-a3` | 树 + A3 补丁 | 46 | `TOKEN_GRID=0` | 单独看 E1 的贡献 |
| `sm46-a3` | 同一个二进制 | 46 | `TOKEN_GRID=1` | E1 + A3 合计 |

A3 是运行时 env，所以 `sm46-a3` 只需**一个**二进制就能分离 E1 和 A3。

## 触发构建

Actions 页面 → `Build Ternary ninfer (sm_89)` → Run workflow。
或：

```bash
gh workflow run build-sm46.yml --repo Yinr/ninfer-4090-windows --ref sm46-a3
```

## 相对 `ternary-ci` 配方改动的地方

沿用 2026-09-25 跑通的配方（run `36092517775`，42m17s），三处改动都有原因：

1. **不再 `git clone` + `xcopy` 40 个补丁** —— 树已内置三元算子。
2. **不再预置 BtbN FFmpeg** —— 树内 `ffmpeg/{include,lib}` 已随包提供。
3. **去掉 `-DNINFER_ENABLE_AVX2` 与 `-DNINFER_CUDA_ARCH`** —— 这两个选项在当前 `CMakeLists.txt`
   里已不存在（`NINFER_TIER` 同样如此）。

## 门禁

- `CMAKE_CUDA_ARCHITECTURES` 只接受 `120a` / `120` / `89`，写错在 configure 阶段就 FATAL。
- `NINFER_SM_COUNT` 非数字直接 FATAL。
- configure 阶段 `message(STATUS ...)` 打印 `SM_COUNT`。
- 另有一步显式断言 `NINFER_SM_COUNT=<n>` 真的出现在 `build.ninja` 的编译行里
  ——旧 CI 踩过"宏没进编译行、编 40 分钟才发现白编"的坑。
- 编译用 `-- -k 0`，一轮拿全所有编译错误。
- `apps/CMakeLists.txt` 无 `if()` 守卫，三个 exe（`ninfer` / `ninfer-serve` / `ninfer-perplexity`）
  在 `NINFER_BUILD_APPS=ON`（默认）下**同批产出** —— 这是数值门的前提，不要只重链 serve。

## 工具链

`Jimver/cuda-toolkit@v0.2.36` + `cuda 13.3.1`（更早的 action 版本白名单里没有 13.3.1）
+ `ilammy/msvc-dev-cmd@v1`（x64）+ Ninja。
流水线第一步会打印实际生效的 MSVC / CMake / Ninja / nvcc 版本 —— GitHub 的
`windows-latest` 镜像会滚动更新，版本一旦与包方验证过的组合不符，日志里第一时间可见。

---

# 实测结果（run 36831099104）

硬件：RTX 4070 12 GB（12282 MiB，cc 8.9，驱动 616.92，WDDM），Windows 11。
模型：`bonsai2_27b_ternary_v2.ninfer`（与出厂部署同一个文件）。
Runner 实际工具链：**MSVC 14.51.36231（VS 18 Enterprise）** + Windows SDK 10.0.26100.0 +
CUDA 13.3.1 + Ninja —— 与包方验证版本一致，漂移风险解除。
两个臂均 Build ✅，`ninfer-serve.exe` SHA256 确实不同：
baseline `6120C3C4…EA73`，sm46-a3 `1FC33C42…1CFD`。

## 1. 数值门：三个配置逐位相同

语料 `ppl-corpus-80k.txt`（35,090 计分 token，91 个窗口），`--kv-dtype fp8`，
`--context 768 --stride 384`（原因见 §4）：

| 配置 | total_nll | perplexity |
|---|---|---|
| baseline (SM=128) | 116951.29597758339 | 28.019348717916902 |
| sm46-a3 TG=0 | 116951.29597758339 | 28.019348717916902 |
| sm46-a3 TG=1 | 116951.29597758339 | 28.019348717916902 |

**双精度 ULP 差 = 0；91 个窗口逐个比对，0 个不同。**
E1 与 A3 都严格逐位不变，包方"A3 逐位不变"的说法得到独立验证。

## 2. 速度 A/B：唯一真实收益来自 A3，且只在 prefill

四臂，每题 3 次取中位数，**开 `--no-prefix-reuse`**（见 §4），ctx 32768 / fp8 / MTP K=3。

| 臂 | math | count | prose | tech | **prefill** |
|---|---|---|---|---|---|
| factory（出厂二进制，对照） | 47.4 | 118.4 | 57.6 | 71.1 | 1445 |
| baseline（SM=128） | 46.9 (−1.1%) | 118.2 (−0.2%) | 57.2 (−0.7%) | 70.5 (−0.8%) | 1443 |
| sm46-a3 **TG=0**（SM=46） | 47.0 (−0.8%) | 117.3 (−0.9%) | 56.8 (−1.4%) | 70.0 (−1.5%) | **1432** |
| sm46-a3 **TG=1** | 46.9 (−1.1%) | 117.5 (−0.8%) | 57.1 (−0.9%) | 70.4 (−1.0%) | **1580 (+9.3%)** |

同臂内 3 次重复差异 < 0.3%，数据稳定。

**结论：**

- **A3 有效：prefill 1432 → 1580 = +10.3%**（TG=1 相对 TG=0）。
  注意包方报的 +31.1% 是相对它自己的基线口径，本机只有 ~10%。
- **E1（SM=46）在本卡上没有收益**：decode 与 prefill 都在 ±1.5% 以内，
  TG=0 臂相对 baseline 甚至略低。`gridDim.y = SM/4` 只是上界，
  decode（batch 1）的 GEMM M 维很小，本来就铺不满，砍上界帮不上忙。
- **decode 全线比出厂慢约 1%**：四个新臂一致、可复现，应归因于工具链差异
  （MSVC 14.51 / CUDA 13.3 vs 包方所用），而非补丁。
- **贪心输出逐字比对：四臂四题全部一致** ✅

### 2.1 复现性（第二轮，每题 5 次重复，空载系统，PowerShell 7.6.6）

| 臂 | prefill 第一轮 | prefill 第二轮 | A3 增益 |
|---|---|---|---|
| factory | 1445 | 1451 | — |
| baseline | 1443 | 1440 | — |
| TG=0 | 1432 | 1432 | — |
| TG=1 | 1580 | 1576 | +10.3% / **+10.06%** |

两轮差 < 0.5%，逐字比对仍然四臂全同。

### 2.2 prefill 长度扫描：A3 的收益来自哪里

一次回答两个问题：A3 是不是只在长上下文才有用？E1 到底有没有用？
三个臂，每档 3 次取中位数，`--no-prefix-reuse`，ctx 32768 / fp8。

| 目标 tok | 实测 tok | baseline SM=128 | sm46 TG=0 | sm46 TG=1 | **A3 vs TG=0** |
|---|---|---|---|---|---|
| 256 | ~233 | 1209 | 1189 | 1261 | **+6.1%** |
| 512 | ~453 | 1406 | 1388 | 1511 | **+8.9%** |
| 1024 | ~857 | 1434 | 1419 | 1563 | **+10.1%** |
| 2048 | ~1630 | 1489 | 1472 | 1626 | **+10.5%** |
| 4096 | ~3212 | 1488 | 1474 | 1628 | **+10.4%** |
| 8192 | ~6402 | 1481 | 1472 | 1629 | **+10.7%** |
| 16384 | ~12759 | 1440 | 1425 | 1570 | **+10.2%** |
| 30000 | ~23300 | 1358 | 1352 | 1477 | **+9.2%** |

- **A3 的收益是全区间稳定的**，不是长上下文专属：256 tok 就已经 +6.1%，
  1K–16K 稳定在 +10% 上下，23K 仍有 +9.2%。这符合它的机制 ——
  三元 GEMM 的 token 轴铺进 `blockIdx.y`，并行度提升与 T 的大小基本无关。
- **E1 在每一档都慢一点**（baseline SM=128 对比 TG=0）：
  −1.7% / −1.3% / −1.1% / −1.1% / −0.9% / −0.6% / −1.0% / −0.4%。
  幅度小但**方向完全一致，8 档里 8 档都慢**，因此判定为真实的小回归而非噪声。
  合理解释：4070 只有 46 个 SM，把 `gridDim.y` 上界从 32 砍到 11，
  在这个尺寸下并没有多出可利用的并发，只是少了一批原本能被调度器吸收的尾块。
- 附带观察：prefill 在 2K–8K 达到峰值（约 1480–1490），23K 回落到 1358，
  是 ctx 32768 下 KV 与工作区压力的影响，与本分支无关。

## 3. 现在的建议：只保留 A3，放弃 E1

| | A3 | E1 |
|---|---|---|
| 收益 | prefill 全区间 +6%~+10.7% | 无，实测每档慢 0.4–1.7% |
| 风险 | 逐位不变（已验证 0 ULP / 91 窗口） | 同为常量，但收益为负 |
| 生效方式 | 运行时 env，`TG=0` 随时回退 | 编译期宏，改一次重跑 40 分钟 CI |
| 适用面 | 与显卡无关，任何卡都只加速不改结果 | 与显卡强绑定，换卡就得重编 |

用法：`set NINFER_TERNARY_TOKEN_GRID=1`；出问题删掉这个环境变量即刻回到原行为。
**不要**设 `NINFER_SM_COUNT`，保持默认 128。

A3 只加速 prefill，decode 不变 —— decode 的 GEMM M 维小，token 轴本来就没被压满。
本机实测 A3 臂的 decode 与 TG=0 完全一致。

## 4. 两个把测量搞坏的坑（都已修正，务必知道）

### 4.1 `ninfer-perplexity` 在 context ≥ 984 时必崩（上游引擎 bug，非本分支引入）

症状：任何 `--context ≥ 984` 的 perplexity 运行在第一个窗口就抛
`error: scoring ... window 0 failed: bad allocation`。与语料长度无关，
与 context 无关，**只与窗口实际 token 数有关**（≥984 挂，≤983 过）。

根因（已定位到行，详见 `build/_ninfer_ppl_badalloc_report.md`）：
`causal_score` 的 workspace 计划只按三个 flush 张量定容，
**漏算了输出头 `ops::linear` 的三元 workspace**；
同时三元算子的容量函数少算了一块 —— `ternary_dispatch()` 在
`folded_activation()` 已经申请过之后，**又申请了一次** int8 的 `codes`/`scales`。

- 抛出点：`src/core/arena.cu:462`，`end > cap_` 时 `throw std::bad_alloc()`
- 计划：`src/targets/qwen3_6/impl/runtime/layouts_impl.h:413-420`
  （`kCausalScoreTile = 1024`，`layouts.h:23`）
- 双算：`src/ops/linear/ternary/ternary_dispatch.cpp:98-99`
- 旋转块：`src/ops/linear/ternary/ternary_rotation.cpp:45-58,74-76`

字节数校验：T=1024 需要 524,300,800 B，计划只有 508,567,552 B，**差 15,733,248 B**。
两个模型给出不同预测阈值（T=994 vs T=984），实测边界正是 **T=983 过 / T=984 挂** ⇒ 双算被证实。

**没有开关可绕**（`ninfer-perplexity` 不暴露 `prefill_chunk`/`kv_capacity`，
且 `Engine` 在 scoring 路径会强制覆盖）。当前唯一可用的规避就是 `--context ≤ 983`。

这也是为什么上面的数值门用 `--context 768`：留足余量，且 768 在任何卡上都不会触发。

### 4.2 测速脚本自己踩的两个坑（已修正）

- **`timings.prompt_n` 不是 prompt 总长**，它是"实际预填充的 token 数"。
  前缀缓存命中后从 35 掉到 7，导致各臂不可比。应该用 `usage.prompt_tokens`。
  稳妥做法是直接加 `--no-prefix-reuse`，让每次都是冷启动。
- **PowerShell 变量大小写不敏感**：`$q` 就是 `$Q`。题组用 `$Q`、
  prefill 的 nonce 也叫 `$q`，于是第一个臂跑完后 `$Q` 被覆盖成字符串，
  后面各臂 `foreach ($k in $Q.Keys)` 一次都不执行，
  且 `$Q[$k]` 取空后只发 chat template —— 表现为莫名其妙的 `prompt_n=12` 和全线 37 t/s。

另外 `Kill-Port` 只按端口杀进程，而 `ninfer-serve` 会残留并继续占显存，
导致后续臂是在显存被抢的状态下跑的。现在每臂开跑前会杀掉全部 ninfer 进程
并等显存回落到 1500 MiB 以下才继续。
