---
doc_type: user-guide
slug: overview
component: oar-ocr
status: current
summary: OAR-OCR 项目总览——能力边界、使用条件、快速上手与文档地图
tags: [overview, getting-started, requirements]
last_reviewed: 2026-07-23
---

# OAR-OCR 总览

## 功能简介

OAR-OCR 是一套用 **Rust** 实现的 OCR 与文档理解工具包。它覆盖从「读图出字」到「版面 / 表格 / 公式 / 印章」的经典流水线，并通过 `oar-ocr-vl` 提供基于 Candle 的文档 Vision-Language（VLM）推理。

你可以把它当作：

- **库**：在应用里嵌入 `OAROCR` / `OARStructure` 或底层 predictor
- **参考实现**：`examples/` 与 `oar-ocr-vl/examples/` 中有可运行示例

当前 workspace 版本：**0.8.1**（`rust-version = 1.95`，许可 **Apache-2.0**）。

## 它能做什么

| 能力 | 说明 | 入口 |
|---|---|---|
| 端到端 OCR | 文本检测 + 识别（含 PP-OCRv4/v5/v6 等） | `OAROCRBuilder` → `OAROCR` |
| 文档结构分析 | 版面、表格、公式、印章、方向与矫正 | `OARStructureBuilder` → `OARStructure` |
| 任务级 predictor | 只跑检测 / 只跑识别等单任务 | `oar-ocr-core` predictors |
| 文档 VLM | PaddleOCR-VL、GLM-OCR、MonkeyOCR、HPD、Hunyuan、MinerU 等 | `oar-ocr-vl` |
| 执行后端 | CPU（默认）及 CUDA / TensorRT / DirectML / CoreML / WebGPU / OpenVINO | Cargo features + `OrtSessionConfig` |
| 模型获取 | 本地路径、内存字节（`include_bytes!`）、可选 ModelScope 自动下载 | 路径 / bytes / `auto-download` |

经典 ONNX 流水线经 **ONNX Runtime** 推理；VLM 经 **Candle** 原生推理（CPU / CUDA / Metal，见 `oar-ocr-vl`）。

## 前置条件

### 通用

| 项 | 要求 |
|---|---|
| Rust | **1.95+**（workspace `rust-version`） |
| 工具链 | 标准 `cargo` 构建；C++ 链接环境按平台就绪 |
| 模型 | 检测 / 识别等 ONNX 文件与字典；或开启 `auto-download` 用注册名拉取 |
| 运行时 | 默认 feature 会在构建期下载兼容的 ONNX Runtime（`download-binaries`） |

### 平台注意

| 平台 | 条件 |
|---|---|
| **Windows** | 链接预构建 ORT 需要 **MSVC v143**（Visual Studio 2022 或对应 Build Tools +「使用 C++ 的桌面开发」）。VS 2019 常导致 `LNK2001: __std_find_trivial_*`，见 [FAQ](FAQ.md) |
| **CUDA GPU** | 启用 feature `cuda`，并配置兼容驱动 / CUDA / cuDNN；再在代码里选择 `OrtExecutionProvider::CUDA`（仅开 feature 不会自动走 GPU） |
| **macOS Apple Silicon（VLM）** | `oar-ocr-vl` 使用 `--features metal`；吞吐相关建议见 [environment-variables](environment-variables.md)（如 `OAR_VL_DTYPE=f16`） |
| **离线 / 企业环境** | `--no-default-features` 并自行提供 ONNX Runtime（`ORT_LIB_LOCATION` / `ORT_LIB_PATH`）；模型改用本地路径或内嵌字节 |

### 模型与缓存

- **手动下载**：GitHub Releases 与 [models 指南](models.md)
- **自动下载**（可选）：`cargo add oar-ocr --features auto-download`，构建器可传注册文件名；缓存目录默认 **`~/.oar`**，由 **`OAR_HOME`** 覆盖
- **内存加载**：builder 接受 ONNX 字节，便于单二进制内嵌（见 [usage](usage.md#loading-models-from-memory)）

## 如何使用（最短路径）

### 1. 安装依赖

```bash
cargo add oar-ocr
```

需要 GPU EP 与模型自动下载时：

```bash
cargo add oar-ocr --features cuda,auto-download
```

这会保留默认的 `download-binaries` 与 `simd`，并额外打开 CUDA EP 与 ModelScope 自动下载。

### 2. 最小 OCR 示例

开启 `auto-download` 时可直接用注册名；否则换成本地路径。

```rust
use oar_ocr::domain::tasks::TextDetectionConfig;
use oar_ocr::prelude::*;
use std::path::Path;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let ocr = OAROCRBuilder::new(
        "pp-ocrv6_tiny_det.onnx",
        "pp-ocrv6_tiny_rec.onnx",
        "ppocrv6_tiny_dict.txt",
    )
    .text_detection_config(TextDetectionConfig {
        score_threshold: 0.2,
        box_threshold: 0.45,
        unclip_ratio: 1.4,
        max_candidates: 3000,
        ..Default::default()
    })
    .build()?;

    let image = load_image(Path::new("document.jpg"))?;
    let results = ocr.predict(vec![image])?;

    for region in &results[0].text_regions {
        if let Some((text, confidence)) = region.text_with_confidence() {
            println!("{text} ({confidence:.2})");
        }
    }
    Ok(())
}
```

### 3. 文档结构分析（可选）

```rust
use oar_ocr::prelude::*;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let structure = OARStructureBuilder::new("pp-doclayout_plus-l.onnx")
        .with_table_classification("pp-lcnet_x1_0_table_cls.onnx")
        .with_table_structure_recognition("slanet_plus.onnx", "wireless")
        .table_structure_dict_path("table_structure_dict_ch.txt")
        .with_ocr(
            "pp-ocrv5_mobile_det.onnx",
            "pp-ocrv5_mobile_rec.onnx",
            "ppocrv5_dict.txt",
        )
        .build()?;

    let result = structure.predict("document.jpg")?;
    println!("{}", result.to_markdown());
    Ok(())
}
```

### 4. 需要 VLM 时

单独依赖 `oar-ocr-vl`（通常再加 `oar-ocr-core`），见 [oar-ocr-vl/README.md](../oar-ocr-vl/README.md)。布局优先的后端可用 **DocParser**（外置 layout ONNX + VL 区域识别）；OvisOCR2、HPD-Parsing、Hunyuan、MinerU 等走各模型原生全页路径。

## 能力边界与注意

- **Feature 不等于已选后端**：`cuda` 等只让对应 EP 可编译；必须在 `OrtSessionConfig` 中配置，并把 CPU 放在 fallback 链末尾（见 [features](features.md)）。
- **小模型 + GPU 可能更慢**：PP-OCRv6 tiny/small 单图场景下，拷贝与同步开销可能超过 GPU 收益，宜默认 CPU + `simd`，或换 medium / 批处理（见 [FAQ](FAQ.md)）。
- **请求未启用的 EP 会失败**：例如未开 `cuda` 却配置 `OrtExecutionProvider::CUDA`，`build()` 返回错误。
- **VLM 与经典流水线分 crate**：多数应用用 `oar-ocr` 即可；VLM 用 `oar-ocr-vl`，不要假设两套模型格式互通。

## 文档地图

| 文档 | 内容 |
|---|---|
| [architecture.md](architecture.md) | Workspace 分层、数据流、扩展点（开发者） |
| [usage.md](usage.md) | Builder、批处理、加速器、内存加载等 API 用法 |
| [features.md](features.md) | Cargo features 与执行提供方 |
| [models.md](models.md) | 预训练模型、字典、auto-download 名称 |
| [environment-variables.md](environment-variables.md) | `OAR_*` / `OAR_VL_*` 等运行与调试变量 |
| [FAQ.md](FAQ.md) | 构建与运行常见问题 |
| 根 [README.md](../README.md) | 对外简介与 Highlights |

## 相关功能

- 可运行示例：[`examples/`](../examples)（经典流水线）、[`oar-ocr-vl/examples/`](../oar-ocr-vl/examples)（VLM）
- 社区行为准则：[CODE_OF_CONDUCT.md](../CODE_OF_CONDUCT.md)
- 上游致谢：ONNX Runtime（via `ort`）、PaddleOCR 模型、Candle 等见根 README
