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
