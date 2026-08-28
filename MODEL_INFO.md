# Cellpose 模型基本信息说明

## 1. 项目定位

这是 Cellpose / Cellpose-SAM 这套细胞分割模型仓库，当前代码已经是 Cellpose 4.x / Cellpose-SAM 版本。代码中默认模型是 `cpsam_v2`，并且支持额外的 DINO 版本模型：

- `cpsam_v2`: 默认主力模型，基于 SAM-ViT-L 骨干，修正了低对比区域的假阳性问题
- `cpdino`: 基于 DINOv3 ViT-L 骨干
- `cpdino-vitb`: 基于 DINOv3 ViT-B 骨干，较小模型
- `cpsam`: 原始 CellposeSAM 版本

仓库入口文件是：

- `cellpose/models.py`
- `cellpose/vit.py`
- `cellpose/cli.py`
- `cellpose/__main__.py`

其中，`CellposeModel` 会在初始化时根据 `pretrained_model` 加载权重，并根据权重中是否包含 `encoder.cls_token` 判断是 DINO 还是 SAM 骨干。

---

## 2. 模型架构与骨干网络

### 2.1 默认模型

在 `cellpose/models.py` 中，默认初始化是：

```python
CellposeModel(gpu=False, pretrained_model="cpsam_v2")
```

并且代码中定义了：

```python
MODEL_NAMES = ["cpsam_v2", "cpdino", "cpdino-vitb", "cpsam"]
```

这说明当前仓库的“内置模型”就是这四种模型。

### 2.2 骨干类型

在 `cellpose/vit.py` 中可以看到：

- `CPSAM`：使用 `sam_model_registry["vit_l"](None).image_encoder`
- `CPDINO`：使用 `dinov3_vitl16` 或 `dinov3_vitb16`

也就是说：

- `CPSAM` 实际上是基于 SAM 的 ViT-L 视觉编码器
- `CPDINO` 基于 DINOv3 的 ViT-L / ViT-B 编码器

这是一种“Transformer-based segmentation backbone + Cellpose decoding head”的结构，不是传统 U-Net 版本那种老式 Cellpose。

### 2.3 参数量说明

从代码逻辑看，模型不再使用传统小型 U-Net，而是使用大规模视觉 Transformer 骨干：

- `SAM ViT-L` 通常是约 300M 级参数
- `DINOv3 ViT-L` 也属于 300M 量级
- `DINOv3 ViT-B` 大约 80M~90M 量级

因此，这个仓库里常见模型的参数规模可以概括为：

| 模型 | 骨干 | 估算参数规模 | 说明 |
|---|---:|---:|---|
| `cpsam_v2` | SAM ViT-L | 约 300M | 默认模型，较大 |
| `cpsam` | SAM ViT-L | 约 300M | 早期 SAM 版本 |
| `cpdino` | DINOv3 ViT-L | 约 300M | 更大/更通用 |
| `cpdino-vitb` | DINOv3 ViT-B | 约 80M~90M | 更轻量 |

> 说明：仓库本身没有直接暴露“精确参数总数”常量，真实参数数要以加载的权重和 backbone 版本为准。代码里实际是使用大 Transformer backbone，所以参数量水平是几百 M 级而不是传统小型 CNN 的几 M。

---

## 3. “倍率”与尺寸归一化

Cellpose 模型并不是简单地做“固定分辨率处理”，而是做“对象尺寸归一化”。关键点在于：

- 模型训练时的 ROI 平均直径被设为 30 像素（代码里 `diam_mean = 30.`）
- 文档中明确写了：训练数据中对象直径范围大约是 7.5 ~ 120 像素，均值约 30
- 如果图像中的细胞更大/更小，可以提供 `diameter` 参数，模型会按比例缩放图像

例如：

- 若 `diameter=60`，相当于把目标缩放到 30 像素的 2 倍尺寸对应位置，通常是下采样
- 若对象更大，则通常要增加 `niter`，让动态过程（dynamics）继续迭代得更稳定

因此，这里的“倍率”更准确地说是“尺寸归一化尺度/缩放系数”，不是传统模型的放大倍数。

---

## 4. 运行方式

### 4.1 直接启动 GUI

```bash
python -m cellpose
```

这会打开 Cellpose GUI。首次运行时会自动下载模型权重到 `~/.cellpose/models/`。

### 4.2 批量处理整个目录

```bash
python -m cellpose --dir /path/to/images --pretrained_model cpsam_v2 --save_png
```

### 4.3 单张图推理

```bash
python -m cellpose --image_path /path/to/image.png --pretrained_model cpdino --save_tif --savedir /path/to/output
```

### 4.4 3D 图像推理

```bash
python -m cellpose --dir /path/to/3d_images --do_3D --pretrained_model cpdino --save_tif
```

### 4.5 GPU 推理

```bash
python -m cellpose --dir /path/to/images --pretrained_model cpsam_v2 --use_gpu --save_png
```

### 4.6 训练新模型

```bash
python -m cellpose --dir /path/to/train_data --train --pretrained_model cpsam_v2 --n_epochs 100 --learning_rate 1e-5
```

---

## 5. 关键参数说明

下面这些参数来自 `cellpose/cli.py`，是最常用的推理参数：

| 参数 | 含义 | 典型值 |
|---|---|---|
| `--pretrained_model` | 选择使用的内置模型或自定义路径 | `cpsam_v2`, `cpdino`, `cpdino-vitb` |
| `--use_gpu` | 是否使用 GPU | `True`/开启 |
| `--diameter` | 对象直径，用于尺寸归一化 | `30` |
| `--do_3D` | 是否按 3D 堆栈处理 | `False` |
| `--flow_threshold` | 流场误差阈值，过滤低质量细胞 | `0.4` |
| `--cellprob_threshold` | 像素候选阈值，影响保留的大/小细胞 | `0` |
| `--niter` | 动态迭代次数 | `0` / 更大值如 `2000` |
| `--min_size` | 最小掩码面积 | `15` |
| `--batch_size` | 推理 batch 大小 | `8` |
| `--save_png` | 保存 PNG 结果 | 适合查看 |
| `--save_tif` | 保存 TIFF 结果 | 适合后续处理 |
| `--save_flows` | 保存流场图 | 适合分析 |
| `--save_outlines` | 保存边界轮廓 | 适合可视化 |
| `--no_norm` | 关闭归一化 | 可选 |

### 5.1 `diameter`

这是最重要的尺寸参数之一：

- 默认是以 30 像素为参考标准
- 如果对象比默认更大，设大一点会更稳
- 如果对象比默认更小，也可以设小一点

### 5.2 `flow_threshold`

这个参数控制“流场质量”过滤，值越低会更容易保留更多候选 mask；
值越高会筛掉更不稳定的分割结果。

### 5.3 `cellprob_threshold`

控制细胞概率阈值：

- 越低，越容易找到更多/更大的候选对象
- 越高，越保守，通常较少 false positive

### 5.4 `niter`

用于动态过程的迭代次数：

- 默认值为 0，代码会根据对象尺寸自动分配
- 如果对象很长或边界复杂，可以适当增大

---

## 6. Python API 用法

除了 CLI，仓库也支持 Python 调用：

```python
from cellpose import models

model = models.CellposeModel(gpu=True, pretrained_model="cpsam_v2")
masks, flows, styles = model.eval(img, diameter=30, flow_threshold=0.4, cellprob_threshold=0.0)
```

如果输入是多通道图像，需要通过 `channel_axis` 或 `z_axis` 指定通道与深度轴；新版本里 `channels` 参数已被标记为 deprecated。

---

## 7. 这个模型的总结

从代码和 docs 来看，这个仓库的核心特征是：

1. 不是传统 U-Net，而是基于大规模 Transformer 的 Cellpose-SAM / Cellpose-DINO
2. 默认模型是 `cpsam_v2`
3. 模型尺寸以大约 300M 级参数为主，`cpdino-vitb` 是较轻的版本
4. 训练以 30 像素为平均对象直径进行归一化
5. 推理中最关键的参数是 `pretrained_model`、`diameter`、`flow_threshold`、`cellprob_threshold`
6. CLI 方案直接用 `python -m cellpose` 即可，方便批处理和大量图片推理

---

## 8. 推荐的最简使用命令

### 8.1 直接使用默认模型

```bash
python -m cellpose --dir /path/to/images --save_png --use_gpu
```

### 8.2 选择更强通用模型

```bash
python -m cellpose --dir /path/to/images --pretrained_model cpdino --save_tif --use_gpu
```

### 8.3 细胞较大时

```bash
python -m cellpose --dir /path/to/images --pretrained_model cpsam_v2 --diameter 60 --niter 2000 --save_png
```

---

## 9. 一句话概括

这个仓库的核心模型是“基于大 Transformer 的细胞分割模型”，默认使用 `cpsam_v2`，参数规模在 300M 级别，尺寸归一化以平均直径 30 像素为基准，CLI 推理非常简单：`python -m cellpose ...` 即可完成图像分割与批处理。
