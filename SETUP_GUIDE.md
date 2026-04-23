# EchoMimicV2 Linux 服务器部署指南

> 从零配置到运行 Demo，按顺序逐条执行即可。

---

## 0. 目录约定

| 用途 | 路径 | 说明 |
|------|------|------|
| 工作目录 | `~/workspace/Zhengwei/echomimic_v2` | 代码、配置、脚本 |
| 大型文件 | `/cache/Zhengwei/echomimic_v2` | 预训练权重、FFmpeg 等大文件 |

> 工作目录空间有限，预训练权重等大文件存放在 `/cache` 下，通过软链接引用。

```bash
# 创建目录结构
mkdir -p ~/workspace/Zhengwei
mkdir -p /cache/Zhengwei/echomimic_v2
```

---

## 1. 系统要求

| 项目 | 要求 |
|------|------|
| OS | CentOS 7.2+ / Ubuntu 22.04+ |
| Python | 3.10（推荐），3.8 / 3.11 也可 |
| CUDA | >= 11.7（推荐 12.4） |
| GPU | A100(80G) / RTX4090(24G) / V100(16G)，最低 12GB 显存 |
| 磁盘 | 预留 30GB+ 用于模型权重（存放在 /cache） |

---

## 2. 下载代码

```bash
cd ~/workspace/Zhengwei
git clone https://github.com/antgroup/echomimic_v2
cd echomimic_v2
```

---

## 3. 创建 Conda 环境

```bash
conda create -n echomimic python=3.10 -y
conda activate echomimic
```

---

## 4. 安装 PyTorch 及相关库

```bash
pip install pip -U
```

```bash
pip install torch==2.5.1 torchvision==0.20.1 torchaudio==2.5.1 xformers==0.0.28.post3 --index-url https://download.pytorch.org/whl/cu124
```

```bash
pip install torchao --index-url https://download.pytorch.org/whl/nightly/cu124
```

> **注意**: 如果你的 CUDA 版本不是 12.4，将 `cu124` 替换为对应版本，如 `cu118`、`cu121` 等。

---

## 5. 安装项目依赖

```bash
pip install -r requirements.txt
```

```bash
pip install --no-deps facenet_pytorch==2.6.0
```

> **为什么 `--no-deps`？** facenet_pytorch 的依赖与本项目冲突，只装它本身即可。

---

## 6. 安装 FFmpeg

### 方式一：下载静态编译版（推荐）

```bash
cd /cache/Zhengwei/echomimic_v2
wget https://www.johnvansickle.com/ffmpeg/old-releases/ffmpeg-4.4-amd64-static.tar.xz
tar -xvf ffmpeg-4.4-amd64-static.tar.xz
```

设置环境变量（永久生效）：

```bash
echo "export FFMPEG_PATH=/cache/Zhengwei/echomimic_v2/ffmpeg-4.4-amd64-static" >> ~/.bashrc
source ~/.bashrc
```

### 方式二：系统包管理器安装

```bash
sudo apt update && sudo apt install -y ffmpeg
```

> 使用方式二时，FFMPEG_PATH 仍需设置，指向 ffmpeg 可执行文件所在目录。

---

## 7. 下载预训练权重

### 7.1 安装 Git LFS

```bash
git lfs install
```

> 如果未安装 git-lfs，先执行：`sudo apt install git-lfs` 或 `conda install -c conda-forge git-lfs`

### 7.2 下载主模型权重

权重存放到 `/cache` 下，再在工作目录中创建软链接：

```bash
cd /cache/Zhengwei/echomimic_v2
git clone https://huggingface.co/BadToBest/EchoMimicV2 pretrained_weights
```

```bash
cd ~/workspace/Zhengwei/echomimic_v2
ln -s /cache/Zhengwei/echomimic_v2/pretrained_weights ./pretrained_weights
```

> 这步下载较大（约 15GB+），请确保网络通畅。如 HuggingFace 不可用，可使用 ModelScope：
> ```bash
> cd /cache/Zhengwei/echomimic_v2
> git clone https://modelscope.cn/models/BadToBest/EchoMimicV2 pretrained_weights
> ```

### 7.3 下载 SD-VAE-FT-MSE（如未包含在主权重中）

```bash
cd /cache/Zhengwei/echomimic_v2/pretrained_weights
git clone https://huggingface.co/stabilityai/sd-vae-ft-mse
```

### 7.4 下载 SD-Image-Variations（如未包含在主权重中）

```bash
cd /cache/Zhengwei/echomimic_v2/pretrained_weights
git clone https://huggingface.co/lambdalabs/sd-image-variations-diffusers
```

### 7.5 下载 Whisper 音频模型（如未包含在主权重中）

```bash
mkdir -p /cache/Zhengwei/echomimic_v2/pretrained_weights/audio_processor
wget -O /cache/Zhengwei/echomimic_v2/pretrained_weights/audio_processor/tiny.pt https://openaipublic.azureedge.net/main/whisper/models/65147644a518d12f04e32d6f3b26facc3f8dd46e5390956a9424a650c0ce22b9/tiny.pt
```

---

## 8. 验证权重文件

```bash
cd ~/workspace/Zhengwei/echomimic_v2
ls pretrained_weights/denoising_unet.pth \
   pretrained_weights/reference_unet.pth \
   pretrained_weights/motion_module.pth \
   pretrained_weights/pose_encoder.pth
```

> 四个文件都存在则基础权重下载成功。加速版额外需要：
> - `pretrained_weights/denoising_unet_acc.pth`
> - `pretrained_weights/motion_module_acc.pth`

完整权重目录结构应为：

```
pretrained_weights/
├── denoising_unet.pth
├── denoising_unet_acc.pth          # 加速版（可选）
├── reference_unet.pth
├── motion_module.pth
├── motion_module_acc.pth           # 加速版（可选）
├── pose_encoder.pth
├── audio_mapper-50000.pth
├── wav2vec2-base-960h/
├── AutoFlow/
├── sd-vae-ft-mse/
├── sd-image-variations-diffusers/
└── audio_processor/
    └── tiny.pt
```

---

## 9. 运行 Demo

```bash
cd ~/workspace/Zhengwei/echomimic_v2
```

### 方式一：Gradio Web 界面（标准版）

```bash
python app.py
```

浏览器访问控制台输出的地址（默认 `http://0.0.0.0:7860`）。

### 方式二：Gradio Web 界面（加速版，推荐）

```bash
python app_acc.py
```

加速版推理速度提升 9 倍（A100 上 ~50s/120帧 vs 标准版 ~7min/120帧）。

### 方式三：命令行推理（标准版）

```bash
python infer.py --config='./configs/prompts/infer.yaml'
```

指定输入参数：

```bash
python infer.py \
  --config='./configs/prompts/infer.yaml' \
  -W 768 -H 768 \
  -L 120 \
  --seed 3407 \
  --context_frames 12 \
  --context_overlap 3 \
  --cfg 2.5 \
  --steps 30 \
  --fps 24 \
  --refimg_name 'natural_bk_openhand/0035.png' \
  --audio_name 'chinese/echomimicv2_woman.wav' \
  --pose_name "01"
```

### 方式四：命令行推理（加速版，推荐）

```bash
python infer_acc.py --config='./configs/prompts/infer_acc.yaml'
```

> 加速版默认 6 steps、CFG 1.0，会自动处理配置文件中定义的所有测试用例。

---

## 10. 输出说明

| 运行方式 | 输出路径 |
|----------|----------|
| `infer.py` | `outputs/{model_flag}-seed{seed}/{ref_flag}/{pose_name}/` |
| `infer_acc.py` | `output/{date}/{time}--step_{steps}-{W}x{H}--cfg_{cfg}/` |
| `app.py` / `app_acc.py` | `outputs/{timestamp}_sig.mp4` |

---

## 11. 常见问题

### Q: 显存不够怎么办？
Gradio 界面中勾选 **int8 quantization**，可在 12GB 显存上运行（音频需 < 5 秒）。

### Q: FFMPEG_PATH 未设置会怎样？
代码启动时会打印警告，视频合成会失败。务必确认环境变量已设置：
```bash
echo $FFMPEG_PATH
```

### Q: HuggingFace 下载慢或无法访问？
使用 ModelScope 镜像替代：
```bash
cd /cache/Zhengwei/echomimic_v2
git clone https://modelscope.cn/models/BadToBest/EchoMimicV2 pretrained_weights
```

### Q: CUDA 版本与 cu124 不匹配？
修改第 4 步中的 `cu124` 为你的 CUDA 版本，如 `cu118`、`cu121`。可通过 `nvidia-smi` 查看驱动支持的 CUDA 版本。

### Q: 权重文件不完整？
主权重仓库（`BadToBest/EchoMimicV2`）可能不包含所有子模型，需按 7.3-7.5 步骤单独下载 `sd-vae-ft-mse`、`sd-image-variations-diffusers` 和 `tiny.pt`。
