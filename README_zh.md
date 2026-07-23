# OAR-OCR

[![Crates.io Version](https://img.shields.io/crates/v/oar-ocr)](https://crates.io/crates/oar-ocr)
![Crates.io Downloads (recent)](https://img.shields.io/crates/dr/oar-ocr)
[![dependency status](https://deps.rs/repo/github/GreatV/oar-ocr/status.svg)](https://deps.rs/repo/github/GreatV/oar-ocr)
![GitHub License](https://img.shields.io/github/license/GreatV/oar-ocr)

面向 OCR、文档版面分析与视觉语言（Vision-Language）文档理解的原生 Rust 工具包。

[English](README.md)

## 亮点

- 基于 PP-OCR 模型的端到端文本检测与识别，包含 PP-OCRv6。
- 文档结构分析：版面、表格、公式、印章、方向与矫正。
- 通过 `oar-ocr-vl` crate 为紧凑型文档 VLM 提供原生 Candle 推理。
- 支持 CPU / GPU 执行、模型自动下载，以及内存中加载 ONNX 模型。

## 快速开始

### 安装

```bash
cargo add oar-ocr
```

默认构建会启用 ONNX Runtime 二进制下载与 SIMD 加速。只需按应用需要追加可选能力。例如：

```bash
cargo add oar-ocr --features cuda,auto-download
```

上述命令会保留默认的 `download-binaries` 与 `simd`，使 ONNX Runtime 的 CUDA 执行提供方可供选择，并在首次使用时从 ModelScope 将缺失的已注册模型文件下载到 `$OAR_HOME`。

全部 feature 说明见 [Cargo feature 指南](docs/features.md)；模型下载与缓存行为见 [模型指南](docs/models.md#auto-download)。

Builder 也接受原始 ONNX 字节（例如 `include_bytes!`），可将模型内嵌进单一二进制。参见 [从内存加载模型](docs/usage.md#loading-models-from-memory)。

### OCR 流水线

启用 `auto-download` 时可直接传入已注册的模型名；否则请替换为本地路径。

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

### 文档结构分析

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

## 支持的模型

经典流水线通过 ONNX Runtime 运行 ONNX 模型，并支持下列模型族。具体 checkpoint、字典、下载链接与 auto-download 名称见 [预训练模型指南](docs/models.md)。

### 经典 ONNX 模型

| 任务 | 支持的模型族 |
|---|---|
| 文本检测 | PP-OCRv4、PP-OCRv5、PP-OCRv6 DB 检测器 |
| 文本识别 | PP-OCRv3、PP-OCRv4、PP-OCRv5、PP-OCRv6、SVTRv2、RepSVTR 等 CTC 识别器 |
| 文档预处理 | PP-LCNet 文档方向、PP-LCNet 文本行方向、UVDoc 矫正 |
| 版面检测 | PicoDet、RT-DETR-H、PP-DocLayout S/M/L、PP-DocLayout Plus-L、PP-DocLayoutV2/V3、PP-DocBlockLayout |
| 表格分析 | PP-LCNet 表格分类、RT-DETR-L 单元格检测，以及 SLANet、SLANet+、SLANeXt 结构识别 |
| 公式识别 | PP-FormulaNet、PP-FormulaNet Plus、UniMERNet |
| 印章文本检测 | PP-OCRv4 mobile / server 印章检测器 |

可用的文本识别 checkpoint 覆盖简体中文、繁体中文、英文、阿拉伯文、西里尔文、天城文、希腊文、东斯拉夫文、日文、格鲁吉亚文、韩文、拉丁文、泰米尔文、泰卢固文、泰文等脚本或语言。

### 视觉语言模型（Vision-Language）

[`oar-ocr-vl`](oar-ocr-vl/README.md) crate 为紧凑型文档 VLM 提供原生 [Candle](https://github.com/huggingface/candle) 推理，支持 CPU、CUDA 与 Metal。

| 模型 | 参数量 | 能力 |
|---|---:|---|
| [PaddleOCR-VL](https://huggingface.co/PaddlePaddle/PaddleOCR-VL) | 0.9B | 页面解析，文本 / 表格 / 公式 / 图表识别 |
| [PaddleOCR-VL-1.5](https://huggingface.co/PaddlePaddle/PaddleOCR-VL-1.5) | 0.9B | 在 PaddleOCR-VL 任务基础上增加文本 spotting 与印章识别 |
| [PaddleOCR-VL-1.6](https://huggingface.co/PaddlePaddle/PaddleOCR-VL-1.6) | 0.9B | 区域感知的页面解析与任务专用识别 |
| [GLM-OCR](https://huggingface.co/zai-org/GLM-OCR) | 0.9B | 页面解析，文本 / 表格 / 公式识别 |
| [OvisOCR2](https://huggingface.co/ATH-MaaS/OvisOCR2) | 0.8B | 模型原生全页文档到 Markdown 解析 |
| [MonkeyOCRv2-S-Parsing](https://huggingface.co/zenosai/MonkeyOCRv2-S-Parsing) | 0.6B | 模型原生版面、端到端解析、文本 / 公式 / OTSL 表格识别 |
| [MonkeyOCRv2-B-Parsing](https://huggingface.co/zenosai/MonkeyOCRv2-B-Parsing) | 0.7B | 更高容量的 ViT-B 变体，解析与识别任务相同 |
| [HPD-Parsing](https://huggingface.co/PaddlePaddle/HPD-Parsing) | 1B | 层级全页解析，支持持续批处理内容分支、零拷贝共享前缀 KV，以及按分支的 P-MTP |
| [HunyuanOCR 1.5 / 1.0](https://huggingface.co/tencent/HunyuanOCR) | 1B | 提示驱动的全页解析，文本 spotting、表格 / 公式 / 图表识别；1.5 可选 DFlash 解码 |
| [MinerU2.5-2509](https://huggingface.co/opendatalab/MinerU2.5-2509-1.2B) | 1.2B | 模型原生两步版面检测与内容抽取 |
| [MinerU2.5-Pro-2605](https://huggingface.co/opendatalab/MinerU2.5-Pro-2605-1.2B) | 1.2B | 更新的 MinerU2.5 checkpoint，使用相同两步流水线 |
| [MinerU-Diffusion-V1-0320](https://huggingface.co/opendatalab/MinerU-Diffusion-V1-0320-2.5B) | 2.5B | 块扩散 OCR，支持结构化两步抽取或单次文本识别 |

PaddleOCR-VL 系列与 GLM-OCR 可接入外置版面的 [`DocParser`](oar-ocr-vl/README.md#document-parsing-pipeline)。OvisOCR2、HPD-Parsing 以及 MonkeyOCRv2 S/B 解析模型则通过专用示例提供模型原生全页路径。HunyuanOCR 与 MinerU 系列同样使用各自的模型原生解析流水线。

环境配置见 [`oar-ocr-vl` 指南](oar-ocr-vl/README.md)，可运行示例见 [`oar-ocr-vl/examples`](oar-ocr-vl/examples)。

## 文档

- [使用指南](docs/usage.md) — API、builder 模式、加速器与模型加载
- [Cargo features](docs/features.md) — 默认项、执行提供方与 feature 组合
- [预训练模型](docs/models.md) — 模型文件、字典与 auto-download 行为
- [环境变量](docs/environment-variables.md) — 运行时与性能相关覆盖项
- [FAQ](docs/FAQ.md) — 常见构建与运行问题
- [项目总览](docs/overview.md) — 能力边界与使用条件（中文）
- [架构指南](docs/architecture.md) — workspace 分层与数据流（中文）

## 示例

其他流水线配置与 API 见 [使用指南](docs/usage.md)。完整经典流水线示例位于 [`examples`](examples)，VLM 示例位于 [`oar-ocr-vl/examples`](oar-ocr-vl/examples)。

## 致谢

本项目建立在以下优秀开源工作之上：

- **[ort](https://github.com/pykeio/ort)**：pykeio 提供的 ONNX Runtime Rust 绑定。该 crate 提供访问 ONNX Runtime 的 Rust 接口，支撑本 OCR 库的高效推理引擎。

- **[PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR)**：百度基于 PaddlePaddle 的出色多语言 OCR 工具包。本项目使用 PaddleOCR 的预训练模型，在多语言文本检测与识别上具备优秀的精度与性能。

- **[Candle](https://github.com/huggingface/candle)**：Hugging Face 的极简 Rust 机器学习框架。我们用 Candle 实现视觉语言模型推理。
