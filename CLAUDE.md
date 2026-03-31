# CLAUDE.md - BannerGen

## Project Overview

BannerGen is a Salesforce Research library for multi-modality ad banner generation. It generates banners from background images and foreground text specifications using three parallel deep learning methods: **LayoutDETR**, **InstructPix2Pix**, and **RetrieveAdapter**.

- **License**: Apache 2.0
- **Language**: Python 3.8.10
- **Framework**: PyTorch (1.12.1 / 2.1.0), CUDA 11.3
- **Environment**: Ubuntu 20.04, NVIDIA GPU with >18GB VRAM (tested on A100)

## Repository Structure

```
BannerGen/
├── banner_gen.py              # Main CLI entry point
├── environment.yaml           # Conda environment spec
├── setup.sh                   # Browser & dependency initialization
├── Dockerfile                 # NVIDIA PyTorch container setup
├── LayoutDETR/                # Method 1: Layout Detection Transformer
│   ├── gen_single_sample_API.py   # Generation API
│   ├── configs/               # BannerConfig, RendererConfig, model configs
│   ├── training/              # Training pipeline (networks, losses, loops)
│   ├── dnnlib/                # NVIDIA DNN utilities
│   ├── detr_util/             # DETR utilities
│   └── torch_utils/           # PyTorch utilities
├── InstructPix2Pix/           # Method 2: Instruction-based image editing
│   ├── gen_single_sample_API.py   # Generation API
│   ├── stable_diffusion/      # Stable Diffusion integration
│   └── configs/               # Generation and banner configs
├── RetrieveAdapter/           # Method 3: Template retrieval + adaptation
│   ├── gen_single_sample_API.py   # Generation API
│   ├── SmartCropping/         # Saliency-aware cropping (U2Net, MTCNN)
│   ├── templates/             # HTML/CSS templates and fonts
│   └── torch_utils/           # PyTorch utilities
├── utils/
│   ├── util.py                # Shared utilities (seeds, bbox conversion, I/O)
│   └── quantize.py            # Quantization utilities
├── test/data/                 # Test examples (images + banner_content.json)
│   ├── example1/              # LayoutDETR test case
│   ├── example2/              # RetrieveAdapter test case
│   └── example3/              # Additional test case
├── executables/               # Chrome/chromedriver binaries
└── fig/                       # Documentation figures
```

## Build & Run Commands

### Environment Setup

```bash
conda env create -f environment.yaml
conda activate bannergen
chmod +x setup.sh && ./setup.sh   # Installs Chrome, chromedriver, fixes OpenCV
```

### Running Banner Generation

```bash
# Main entry point with three model choices
python banner_gen.py --model_name=LayoutDETR --model_path=./weights/
python banner_gen.py --model_name=InstructPix2Pix --model_path=./weights/
python banner_gen.py --model_name=RetrieveAdapter --model_path=./weights/

# Custom inputs
python banner_gen.py --model_name=LayoutDETR --model_path=./weights/ \
  --image_path=test/data/example1/burning.jpg \
  --header_text='Custom Header' \
  --body_text='Body Text' \
  --button_text='CLICK HERE' \
  --num_result=6 \
  --output_path=./result/
```

### Key CLI Arguments

| Argument | Default | Description |
|---|---|---|
| `--model_name` | `LayoutDETR` | One of: LayoutDETR, InstructPix2Pix, RetrieveAdapter |
| `--model_path` | (required) | Path to model weights directory |
| `--image_path` | `test/data/example1/burning.jpg` | Background image |
| `--banner_content_path` | `test/data/example1/banner_content.json` | Text/style specification |
| `--num_result` | `3` | Number of banners to generate |
| `--output_path` | `result/` | Output directory |
| `--post_process` | `"{'jitter': True, 'alignment': True}"` | Post-processing options |

### Docker

```bash
# Base image: nvcr.io/nvidia/pytorch:21.08-py3
docker build -t bannergen .
```

## Testing

There is no automated test suite (no pytest/unittest). Testing is manual via `banner_gen.py` with the provided test data in `test/data/`. Each example directory contains a background image and a `banner_content.json` file.

## Code Conventions

### File Structure Pattern

1. Copyright/license header (Apache 2.0, Salesforce)
2. Module docstring
3. Imports (stdlib, third-party, local)
4. Module-level constants
5. Helper functions/classes
6. Main functions

### Copyright Header

```python
"""
Copyright (c) 2023 Salesforce, Inc.
All rights reserved.
SPDX-License-Identifier: Apache License 2.0
"""
```

### Naming

- **Functions/variables**: `snake_case` (e.g., `load_model`, `generate_banners`)
- **Classes**: `PascalCase` (e.g., `BannerConfig`, `CFGDenoiser`)
- **Constants**: `UPPER_SNAKE_CASE` (e.g., `BROWSER_CONFIG`, `HTML_TEMP`, `LABEL_LIST`)

### Architecture Pattern

Each generation method follows the same interface pattern:
- `load_model(model_path)` - Load model weights
- `generate_banners(model, image_path, elements, ...)` - Generate output banners
- Configuration via `configs/banner_config.py` containing `BannerConfig` and `RendererConfig` classes

The main `banner_gen.py` dispatches to the appropriate method's `gen_single_sample_API` module.

### Model Weights Mapping

```python
BANNER_GEN_MODEL_MAPPER = {
    'LayoutDETR': {'layout': 'ads_multi.pkl'},
    'InstructPix2Pix': {'layout': 'instructpix2pix.ckpt'},
    'RetrieveAdapter': {'superes': 'rdn-liif.pth', 'saliency': 'u2net.pth'}
}
```

## Key Dependencies

- **Deep Learning**: PyTorch, torchvision, transformers (4.20.1), diffusers (0.22.2)
- **Vision**: OpenCV, Pillow, scikit-image, timm (0.5.4), kornia (0.6)
- **Detection**: facenet_pytorch (MTCNN), PaddleOCR
- **Browser Rendering**: Selenium + Headless Chrome (HTML-to-PNG pipeline)
- **Config**: omegaconf (2.3.0), PyYAML
- **Monitoring**: wandb, tensorboardX

## Data Format

Banner content is specified via JSON:

```json
{
  "task": "banner",
  "contentStyle": {
    "elements": [
      {
        "type": "header",
        "text": "Header Text",
        "style": { "fontFamily": "Arial", "color": "#FFFFFF", "fontFormat": "bold" }
      },
      {
        "type": "body",
        "text": "Body text content.",
        "style": { "fontFamily": "Arial", "color": "#FFFFFF" }
      },
      {
        "type": "button",
        "text": "CLICK HERE",
        "buttonParams": { "backgroundColor": "#000000", "radius": 8 },
        "style": { "fontFamily": "Bayon-Regular", "color": "#FFFFFF" }
      }
    ]
  }
}
```

Supported element types: `header`, `pre-header`, `post-header`, `body`, `disclaimer`, `button`, `callout`, `logo`.

## Contribution Guidelines

- Salesforce CLA required for contributions
- PR-based workflow against `main` branch
- Atomic commits with descriptive messages referencing issue numbers
- See CONTRIBUTING.md for full details
