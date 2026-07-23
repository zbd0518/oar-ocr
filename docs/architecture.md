---
doc_type: dev-guide
slug: architecture
component: oar-ocr
status: current
summary: OAR-OCR workspace 架构——crate 边界、分层、流水线数据流与扩展点
tags: [architecture, workspace, design]
last_reviewed: 2026-07-23
---

# OAR-OCR 架构指南

## 概述

本文面向**集成方与贡献者**，说明 OAR-OCR workspace 的 crate 边界、经典 OCR 三层架构、高层流水线数据流，以及可扩展点。实现细节以源码为准；对外任务型用法见 [overview.md](overview.md) 与 [usage.md](usage.md)。

## 前置依赖（开发视角）

| 项 | 说明 |
|---|---|
| Workspace | 成员：`oar-ocr`（根包）、`oar-ocr-core`、`oar-ocr-derive`、`oar-ocr-vl` |
| Rust | edition `2024`，`rust-version` **1.95** |
| 经典推理 | ONNX Runtime，经 `ort`；feature 从根包转发到 core |
| VLM 推理 | Candle（`oar-ocr-vl`），与 ORT 路径分离 |
| 宏 | `oar-ocr-derive`：`ConfigValidator`、`TaskPredictorBuilder` 等内部细节，不建议直接依赖 |

## 快速上手（从哪一层切入）

| 你的目标 | 依赖 | 入口 |
|---|---|---|
| 端到端 OCR / 结构分析 | `oar-ocr` | `prelude`：`OAROCRBuilder`、`OARStructureBuilder` |
| 单任务 predictor、自定义编排 | `oar-ocr-core`（或经 `oar-ocr` 再导出模块） | `predictors::*`、`models`、`domain` |
| 文档 VLM / DocParser | `oar-ocr-vl` + 通常 `oar-ocr-core` | 各模型类型、`DocParser` |
| 过程宏 | 不要直接用 | 经 `oar-ocr` / `oar-ocr-core` re-export |

根包 `oar-ocr` 的公开模块大部分是 **re-export `oar-ocr-core`**；**高层流水线**留在 `oar_ocr::oarocr`。

## 核心概念

### Workspace 与 crate 角色

```text
┌─────────────────────────────────────────────────────────────┐
│  oar-ocr (facade + pipelines)                               │
│  · oarocr::{OAROCR, OARStructure, builders, results}        │
│  · re-exports: core, domain, models, predictors, …          │
│  · prelude 面向最常见任务                                    │
└───────────────────────────┬─────────────────────────────────┘
                            │ depends on
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────────┐
│ oar-ocr-core  │   │ oar-ocr-derive│   │ oar-ocr-vl        │
│ 引擎与任务    │   │ 内部 proc-macro│   │ Candle VLM +     │
│ ORT 推理      │   │               │   │ DocParser 等     │
└───────────────┘   └───────────────┘   └─────────┬─────────┘
                                                  │ uses
                                                  ▼
                                            oar-ocr-core
                                            (layout ONNX 等)
```

| Crate | 职责 | 不负责 |
|---|---|---|
| **oar-ocr** | 产品级流水线（OCR / Structure）、统一 prelude、再导出 | 不实现底层 ORT session 细节 |
| **oar-ocr-core** | 错误类型、domain、models、adapters、tasks、predictors、processors、ORT 推理与可选 download | 不提供 VLM / Candle |
| **oar-ocr-derive** | 构建器与配置校验宏 | 不作为应用直接依赖 |
| **oar-ocr-vl** | 文档 VLM、DocParser、设备与性能相关路径 | 不替代经典 det+rec 流水线 |

版本在 workspace 内对齐（当前 **0.8.1**）。根包 features（`simd`、`cuda`、`auto-download` 等）**同名转发**到 `oar-ocr-core`。

### Core 三层架构（经典 ONNX 路径）

`oar-ocr-core` 文档与目录约定的三层：

1. **Models** — ONNX session 的薄封装，处理原始 tensor 输入输出（如 DB 检测、CRNN 识别、PicoDet / RT-DETR / PP-DocLayout、SLANet、UVDoc 等）。
2. **Adapters** — 把原始输出接到领域类型，承载 pre/post 处理（`domain/adapters/*`）。
3. **Tasks / Predictors** — 语义契约（「文本检测」「版面检测」…）与一致的 builder + `predict` API（`domain/tasks/*`、`predictors/*`）。

横切模块：

- **core**：错误（`OCRError` / `OcrResult`）、batch、`OrtSessionConfig`、推理 session、可选 `download` 注册表
- **domain**：`TextRegion`、`StructureResult`、方向与预测类型等
- **processors**：几何与图像变换（bbox、crop、aspect-ratio bucketing 等）
- **utils**：`load_image` / `load_images` 等

### 两条产品流水线（oar-ocr）

位于 `src/oarocr/`：

| 流水线 | Builder | 典型阶段 |
|---|---|---|
| **OCR** | `OAROCRBuilder` → `OAROCR` | 可选文档方向 / 矫正 / 行方向 → 文本检测 → 裁剪与批处理 → 文本识别 → `OAROCRResult` / `TextRegion` |
| **Structure** | `OARStructureBuilder` → `OARStructure` | 版面检测 → 表格分类 / 单元格 / 结构 → 可选公式 → 可选内嵌 OCR → 结构化结果 / Markdown |

辅助模块包括 preprocess、edge processors、table analyzer、region stitching 等。应用侧优先 `use oar_ocr::prelude::*`。

### VLM 与 DocParser（oar-ocr-vl）

- **布局优先**：外置 layout ONNX（如 PP-DocLayoutV3）+ VL 区域识别 → **DocParser**（适合 PaddleOCR-VL 系、GLM-OCR 等）。
- **模型原生全页**：OvisOCR2、HPD-Parsing、MonkeyOCRv2 的 Layout/EndToEnd、Hunyuan 提示驱动、MinerU 两步抽取等——**不要**强行套 DocParser。
- 设备字符串（如 `cpu` / `cuda` / `metal`）与大量 `OAR_VL_*` 环境变量控制 dtype、attention、CUDA graph 等，见 [environment-variables.md](environment-variables.md)。

## 数据流（经典 OCR）

```text
Image(s)
    │
    ▼
[可选] 文档方向分类 / 透视矫正 / 文本行方向
    │
    ▼
TextDetection predictor  ──►  检测框 + score
    │
    ▼
Crop / batch（region_batch_size 等）
    │
    ▼
TextRecognition predictor ──►  text + confidence
    │
    ▼
OAROCRResult { text_regions: [TextRegion, ...] }
```

Structure 路径以 **layout elements** 为中心，再按类型分支到 table / formula / OCR 等，最后聚合为可导出结构（含 `to_markdown()` 一类结果 API，见 structure 结果类型）。

## 接口参考（架构层）

### 根包模块地图

| 模块 | 来源 | 用途 |
|---|---|---|
| `oarocr` | 本 crate | 高层 builder 与流水线 |
| `core` / `domain` / `models` / `processors` / `predictors` | re-export core | 定制与单任务 |
| `download` | core，需 `auto-download` | 注册名解析与缓存 |
| `utils` | core + OCR 可视化等 | 读图等 |
| `prelude` | 聚合 | 日常集成 |

### Feature 与执行后端（架构约束）

- **默认**：`download-binaries` + `simd`（CPU 预/后处理加速，**不**改变 ORT EP）。
- **EP features**（`cuda`、`tensorrt`、`directml`、`coreml`、`webgpu`、`openvino`）：编译期打开对应 provider；运行时用 `OrtSessionConfig::with_execution_providers` 选择，建议显式带 `CPU` fallback。
- **`auto-download`**：builder 的模型路径可为「文件系统路径 **或** 注册文件名」；缓存根为 `$OAR_HOME`（默认 `~/.oar`）。
- 完整组合与系统要求：[features.md](features.md)。

### 配置注入点

- 流水线级：`OAROCRBuilder` / `OARStructureBuilder` 的 model 路径、batch size、任务 config（如 `TextDetectionConfig`）、`ort_session(...)`。
- 任务级：各 `*Predictor::builder()`（core）。
- 全局环境：见 [environment-variables.md](environment-variables.md)（如 `OAR_HOME`、`CUDA_LAUNCH_BLOCKING` 相关说明、VLM 调试开关）。

## 常见场景

### 1. 只集成库，不碰内部层

依赖 `oar-ocr`，用 prelude 与 [usage.md](usage.md)。无需理解 adapters。

### 2. 替换某一任务的模型实现

在 core 的 **model + adapter + task** 路径上对齐现有 predictor 契约，或通过已有 adapter 系统挂接；保持 predictor 的 `predict(Vec<Image>)` 风格与错误类型一致。

### 3. 应用内嵌模型、无外置文件

builder 接受 ONNX 字节（`include_bytes!`），与 path / auto-download 并列，见 usage 中 memory loading。

### 4. 文档理解选路径

| 需求 | 建议路径 |
|---|---|
| 稳定 det+rec、多语言字典 | `oar-ocr` 经典流水线 |
| 版面 + 表格 + 公式结构化 | `OARStructure` |
| 区域语义 / 图表 / 端到端 Markdown | `oar-ocr-vl` 按模型选 DocParser 或 native |

### 5. 离线构建

`--no-default-features`，提供系统 ORT，关闭或不用 `auto-download`，模型路径全部本地或内嵌。

## 已知限制与注意事项

- **Windows 链接**：预构建 ORT 面向 MSVC v143；旧工具链会出现 unresolved `__std_*`（FAQ）。
- **GPU 对 tiny/small 不一定更快**：测量与建议见 FAQ；评测需排除首次 warmup。
- **Feature / EP 错配**：未启用 feature 的 provider 会在配置阶段报错，而非静默回退（除非你显式配置了 fallback 链且请求的是已启用的 provider）。
- **derive crate 为内部实现细节**：公开集成面是 `oar-ocr` / `oar-ocr-core` / `oar-ocr-vl`。
- **文档语言与 CodeStable**：本仓库对外文档（`docs/`、README）以英文为主；`.codestable/` 下流程产出按 attention 使用中文。本文为中文 dev-guide，与现有英文 docs 并存——若发布策略要求统一语言，可再改一版英文 `current`。

## 相关文档

- [overview.md](overview.md) — 总览与使用条件
- [usage.md](usage.md) — API 与 builder 细节
- [features.md](features.md) — Cargo features
- [models.md](models.md) — 模型清单与 auto-download
- [environment-variables.md](environment-variables.md) — 运行时变量
- [oar-ocr-core/README.md](../oar-ocr-core/README.md) — core 三层与 predictor 示例
- [oar-ocr-vl/README.md](../oar-ocr-vl/README.md) — VLM 与 DocParser
- 根 [README.md](../README.md) — 支持的模型族一览
