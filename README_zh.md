# ComfyUI-Krea2-PixelHead

用于 **Krea-2** 的 ComfyUI 免 VAE 像素直出节点，用来替代耗时较长（约 `1.47 秒`）且高分辨率下容易爆显存的 `242.0 MiB` `qwen_image_vae` 解码器。

[English](README.md) | [简体中文](README_zh.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Hugging Face](https://img.shields.io/badge/Hugging%20Face-pocis%2FKrea--2--PixelHead-yellow)](https://huggingface.co/pocis/Krea-2-PixelHead)
[![Comfy Registry](https://img.shields.io/badge/Comfy%20Registry-comfyui--krea2--pixelhead-blueviolet)](https://registry.comfy.org/publishers/wither033/nodes/comfyui-krea2-pixelhead)

参考 [*FastFrames* (`arXiv:2611.23226`)](https://arxiv.org/abs/2611.23226) 的做法，节点在 `SingleStreamDiT` 最后一步进入 `last.linear` 前截获目标图像的 `6144` 维隐藏特征（自动切掉前部文本与尾部参考图 token），连同 `2×2` 去噪潜块 `x0_hat_norm` 一起送入 `27.49M` 参数（BF16 下 `52.6 MiB`）的降维桥接 ConvNeXt + PixelShuffle 网络，直接输出 RGB 图像。

在 NVIDIA L4 上测试 `768×768`，中位数解码耗时从 `1468.9 ms` 降至 `59.55 ms`（约 `24.7×`），额外显存占用不到 `0.2 GB`。

![1:1 像素对比（左：官方 qwen_image_vae，右：Krea2PixelHead）](assets/hero_1to1_comparison.png)

## 主要特性

- **省掉重型 3D VAE 解码**：文生图全程无需加载 `qwen_image_vae`，避免 `1024～2048px` 出图在最后一步因为 VAE 3D 卷积爆显存。
- **`6144 → 1024` 线性降维桥**：通过无偏置线性层把单流 DiT 的 `6144` 维特征压到 `1024` 维，输出层固定以 `FP32` 计算，保留织物纹理、油画笔触和暗部过渡。
- **兼容常见 Krea-2 变体**：
  - 支持 `krea2_turbo`（8 步 `CFG = 1.0`）、`krea2_raw`（20～35 步 `CFG > 1.0`）、FP8/INT8/GGUF 量化版及社区微调模型；
  - 支持文生图（T2I）和风格参考图输入（`tap.py` 会根据序列长度动态计算 `txtlen` 与 `ref_tokens`）。

## 实测数据（`768×768` 留出艺术画作集 · NVIDIA L4）

以 `Comfy-Org/Krea-2` 官方 `qwen_image_vae.safetensors` 为基准，在 40 张留出的油画与版画样本上同种子对比（8 步 Turbo 与多步采样交替测试）：

| 解码器 | 体积 | 中位数延迟 | 相对速度 | PSNR (↑) | LPIPS-VGG (↓) |
| :--- | :---: | :---: | :---: | :---: | :---: |
| 官方 `qwen_image_vae` | `242.0 MiB` | `1468.90 ms` | `1.0×` | *(基准)* | `0.0000` |
| `Krea2PixelHead` | `52.6 MiB` (BF16) / `104.9 MiB` (FP32) | `59.55 ms` | `24.7×` | `33.33 dB` | `0.2300` |

- 古典与暗调肖像：`37.1 – 41.6 dB`
- 多人物彩色插画：`32.9 – 33.5 dB`
- 细密铜版画排线：`30.8 dB`

## 同系列仓库

| 适配模型 | ComfyUI 节点仓库 | Hugging Face 权重 | 说明 |
| :--- | :--- | :--- | :--- |
| **Krea-2** *(本仓库)* | [Wither033/ComfyUI-Krea2-PixelHead](https://github.com/Wither033/ComfyUI-Krea2-PixelHead) | [pocis/Krea-2-PixelHead](https://huggingface.co/pocis/Krea-2-PixelHead) | `52.6 MiB` BF16，`6144d` 降维桥 RGB，`59.55 ms` (`24.7×`)，均值 `33.33 dB` |
| **Qwen-Image 2.1** | [Wither033/ComfyUI-QwenImage21-PixelHead](https://github.com/Wither033/ComfyUI-QwenImage21-PixelHead) | [pocis/Qwen-Image-2.1-PixelHead](https://huggingface.co/pocis/Qwen-Image-2.1-PixelHead) | `v0.3` 小波版 (`79.3 MiB`, `37.5 ms`, `33.81 dB`, `LPIPS 0.0736`) + `v0.1` (`48.8 MiB`)，4 通道 RGBA |
| **MiniMax-H3** *(预览版)* | [Wither033/ComfyUI-MiniMaxH3-PixelHead](https://github.com/Wither033/ComfyUI-MiniMaxH3-PixelHead) | [pocis/MiniMax-H3-Video-PixelHead](https://huggingface.co/pocis/MiniMax-H3-Video-PixelHead) | 早期实验预览版（复杂纹理偏糊，暂不推荐正式使用，待后续优化） |
| **训练代码与方法整理** | [Wither033/FastFrames-PixelHead-Trainer](https://github.com/Wither033/FastFrames-PixelHead-Trainer) | [HF 合集页](https://huggingface.co/collections/pocis/fastframes-zero-vae-direct-pixelheads-comfyui-6b031df8e9212f8bae294154) | 通用 `pixelhead_trainer` 库、各底座挂载点说明与 Colab L4 训练脚本 |

## 安装与权重

### 1. 安装插件
可在 ComfyUI Manager 搜索 `ComfyUI-Krea2-PixelHead` 安装，或执行 `comfy node install comfyui-krea2-pixelhead`，也可手动克隆：

```bash
cd ComfyUI/custom_nodes
git clone https://github.com/Wither033/ComfyUI-Krea2-PixelHead.git
```

### 2. 下载权重
从 [pocis/Krea-2-PixelHead](https://huggingface.co/pocis/Krea-2-PixelHead) 下载权重，放入 `ComfyUI/models/pixel_heads/`（或 `ComfyUI/models/vae/`）：

- `krea2_pixel_head_bf16.safetensors`（`52.6 MiB`）：日常使用选这个即可（BF16 主干 + FP32 输出层）。
- `krea2_pixel_head.safetensors`（`104.9 MiB`）：FP32 完整权重。

## 节点连接方式

[`example_workflows/`](example_workflows/) 目录下附带了中英文工作流：
- 中文：[`01_Krea2_PixelHead_ZeroVAE_ZH.json`](example_workflows/01_Krea2_PixelHead_ZeroVAE_ZH.json)
- 英文：[`01_Krea2_PixelHead_ZeroVAE_EN.json`](example_workflows/01_Krea2_PixelHead_ZeroVAE_EN.json)

1. 把 `Krea-2 Pixel Head Tap` 串在进入 `KSampler` 前的 `MODEL` 连线上；
2. 用 `Krea-2 Pixel Decode (Zero VAE)` 替换 `VAEDecode`，接入 `LATENT`、`pixel_head` 和 `pixel_tap`。

## 局限说明

- **步数范围**：覆盖 `ModelSamplingFlux(shift=1.15)` 末步 $\sigma \in [0.045, 0.420]$（5～12 步 Turbo 与 18～35 步多步采样）。
- **离线 `.latent` 文件**：需要采样最后一步的 `6144` 维 DiT 实时特征，无法单独解码外部保存的 `.latent` 文件。

## 原作者署名、致谢与开源协议


- **原作者与核心管线致谢**：本项目的像素头核心结构、ComfyUI `Tap + Decode` 挂载范式及训练管线（包括无跳连 ConvNeXt + `2× PixelShuffle` `1x/2x` 解码结构、末层线性映射前 `f_u + g·(f_c - f_u)` 特征截获与潜变量块融合、基于 `VAE(x̂0)` 的跨 Base/Turbo 单步 `ReplayBuffer` 蒸馏、RGBA 四通道读出，以及 `v0.3` 的 2D Haar 小波子带预测与锁定低频 `LL` 的高频对抗训练思路）衍生并改编自原作者 **[Seunghyun Ji (`sorryhyun`)](https://huggingface.co/sorryhyun)**（[GitHub `@sorryhyun`](https://github.com/sorryhyun)）在 **[`sorryhyun/anima_lora`](https://github.com/sorryhyun/anima_lora)**、**[`sorryhyun/anima_pixel_decoder`](https://huggingface.co/sorryhyun/anima-pixel-decoder)** 与 **[`sorryhyun/ComfyUI-Anima-PixelDecoder`](https://github.com/sorryhyun/ComfyUI-Anima-PixelDecoder)** 中的开创性开源工作（MIT License，`Copyright (c) 2026 Seunghyun Ji`）。特此向原作者 `sorryhyun` 致谢！
- **基础论文参考**：直接像素读出（Direct Pixel Readout）理论框架参考 [*FastFrames: Video Generation at Interactive Latency* (`arXiv:2611.23226`)](https://arxiv.org/abs/2611.23226)（Gene Chou et al., Apple & Cornell, 2026）。
- **开源协议**：[MIT License](LICENSE)（完整保留原作者 `Copyright (c) 2026 Seunghyun Ji` 版权声明；配合各扩散底座模型使用时请同时遵守对应底模的许可证条款）。
