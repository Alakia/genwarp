<div align='center'>

![concept image](https://genwarp-nvs.github.io/img/qual_1.png)

</div>

# GenWarp: Single Image to Novel Views with Semantic-Preserving Generative Warping

[![arXiv](https://img.shields.io/badge/arXiv-red)](https://arxiv.org/abs/2405.17251) &nbsp; [![Project](https://img.shields.io/badge/Project-green)](https://genwarp-nvs.github.io/)

[Examples](#examples)
| [How to use](#how-to-use)
| [Citation](#citation)
| [Acknowledgements](#acknowledgements)

## Introduction

This repository is an official implementation for the paper "[GenWarp: Single Image to Novel Views with Semantic-Preserving Generative Warping](https://genwarp-nvs.github.io/)". Genwarp can generate novel view images from a single input conditioned on camera poses. In this repository, we offer the codes for inference of the model. For detailed information, please refer the [paper](https://arxiv.org/abs/2405.17251).

![Framework](https://genwarp-nvs.github.io/img/arch.png)

## Examples

Our model can handle images from various domains including indoor/outdoor scenes, and even illustrations with challenging camera viewpoint changes.

You can find examples on our [project page](https://genwarp-nvs.github.io/) and on our [paper](https://arxiv.org/abs/2405.17251).

![Examples](https://genwarp-nvs.github.io/img/qual_ood.png)

Generated novel views can be used for 3D reconstruction. This example we reconstructed 3D scene via [InstantSplat](https://instantsplat.github.io/). We generated the video using [this implementation](https://github.com/ONground-Korea/unofficial-Instantsplat).

<video autoplay loop src="https://github.com/user-attachments/assets/c6646fdc-4e0e-468e-b801-83fecbd2c5e8" width="852" height="480"></video>

## How to use

### Environment

You can either add packages to your python environment or use Docker to build an python environment.

#### Use Docker to build an environment

> [!NOTE]
> You may want to change username and uld variables written in DockerFile. Please check DockerFile before running the commands below.

``` shell
docker build . -t genwarp:latest
docker run --gpus=all -it -v $(pwd):/workspace/genwarp -w /workspace/genwarp genwarp
```

Inside the docker container, you can install packages as below.

#### Add to your python environment

We tested the environment with python `>=3.10` and CUDA `=11.8`.

``` shell
pip install -r requirements.txt
```

### Download pretrained models

GenWarp uses pretrained models which consist of both our finetuned models and publicly available third party ones. Download all the models to `checkpoints` directory or anywhere of your choice. You can do it manually or by the [download_models.sh](scripts/download_models.sh) script.

#### Download script

``` shell
./scripts/download_models.sh ./checkpoints
```

#### Manual download

> [!NOTE]
> Models and checkpoints provided below may be distributed under different licenses. Users are required to check licenses carefully on their behalf.

1. Our finetuned models:
    - For details about each model, check out the [model card](https://huggingface.co/Sony/genwarp).
    - [multi-dataset model 1](https://huggingface.co/Sony/genwarp)
      - download all files into `checkpoints/multi1`
    - [multi-dataset model 2](https://huggingface.co/Sony/genwarp)
      - download all files into `checkpoints/multi2`
2. Pretrained models:
    - [sd-vae-ft-mse](https://huggingface.co/stabilityai/sd-vae-ft-mse)
      - download `config.json` and `diffusion_pytorch_model.safetensors` to `checkpoints/sd-vae-ft-mse`
    - [sd-image-variations-diffusers](https://huggingface.co/lambdalabs/sd-image-variations-diffusers)
      - download `image_encoder/config.json` and `image_encoder/pytorch_model.bin` to `checkpoints/image_encoder`

The final `checkpoints` directory must look like this:

```
genwarp
└── checkpoints
    ├── image_encoder
    │   ├── config.json
    │   └── pytorch_model.bin
    ├── multi1
    │   ├── config.json
    │   ├── denoising_unet.pth
    │   ├── pose_guider.pth
    │   └── reference_unet.pth
    ├── multi2
    │   ├── config.json
    │   ├── denoising_unet.pth
    │   ├── pose_guider.pth
    │   └── reference_unet.pth
    └── sd-vae-ft-mse
        ├── config.json
        └── diffusion_pytorch_model.bin
```

### Inference

#### (Recommended) Install MDE module

The model requires depth maps to generate novel views. To this end, users can install one of Monocular Depth Estimation (MDE) models publicly available. We recommend ZoeDepth.

``` shell
git clone git@github.com:isl-org/ZoeDepth.git extern/ZoeDepth
```

#### API

**Initialisation**

Import GenWarp class and instantiate with the config. Set the path to the checkpoints directory to `pretrained_model_path` and select the model version in `checkpoint_name`. For more options, check out [GenWarp.py](genwarp/GenWarp.py)

``` python
from genwarp import GenWarp

genwarp_cfg = dict(
    pretrained_model_path='./checkpoints',
    checkpoint_name='multi1',
    half_precision_weights=True
)
genwarp_nvs = GenWarp(cfg=genwarp_full_cfg)
```

**Prepare inputs**

Load the input image and estimate the corresponding depth map. Create camera matrices for the intrinsic and extrinsic parameters. [ops.py](genwarp/ops.py) has helper functions to create matrices.

``` python
from PIL import Image
from torchvision.transforms.functional import to_tensor

src_image = to_tensor(Image.open(image_file).convert('RGB'))[None].cuda()
src_depth = depth_estimator.infer(src_image)
```

``` python
import torch
from genwarp.ops import camera_lookat, get_projection_matrix

proj_mtx = get_projection_matrix(
    fovy=fovy,
    aspect_wh=1.,
    near=near,
    far=far
)

src_view_mtx = camera_lookat(
    torch.tensor([[0., 0., 0.]]),  # From (0, 0, 0)
    torch.tensor([[-1., 0., 0.]]), # Cast rays to -x
    torch.tensor([[0., 0., 1.]])   # z-up
)

tar_view_mtx = camera_lookat(
    torch.tensor([[-0.1, 2., 1.]]), # Camera eye position
    torch.tensor([[-5., 0., 0.]]),  # Looking at.
    z_up  # z-up
)

rel_view_mtx = (
    tar_view_mtx @ torch.linalg.inv(src_view_mtx.float())
).to(src_image)
```

**Warping**

Call the main function of GenWarp. And check the result.

``` python
renders = genwarp_nvs(
    src_image=src_image,
    src_depth=src_depth,
    rel_view_mtx=rel_view_mtx,
    src_proj_mtx=src_proj_mtx,
    tar_proj_mtx=tar_proj_mtx
)

# Outputs.
renders['synthesized']     # Generated image.
renders['warped']          # Depth based warping image (for comparison).
renders['mask']            # Mask image (mask=1 where visible pixels).
renders['correspondence']  # Correspondence map.
```

#### Example notebook

We provide a complete example in [genwarp_inference.ipynb](examples/genwarp_inference.ipynb)

To access a Jupyter Notebook running in a docker container, you may need to use the host's network. For further details, please refer the manual of Docker.

``` shell
docker run --gpus=all -it --net host -v $(pwd):/workspace/genwarp -w /workspace/genwarp genwarp
```

Install additional packages to run the Jupyter Notebook.

```bash
pip install -r requirements_dev.txt
```

## Citation

``` bibtex
@misc{seo2024genwarpsingleimagenovel,
  title={GenWarp: Single Image to Novel Views with Semantic-Preserving Generative Warping}, 
  author={Junyoung Seo and Kazumi Fukuda and Takashi Shibuya and Takuya Narihira and Naoki Murata and Shoukang Hu and Chieh-Hsin Lai and Seungryong Kim and Yuki Mitsufuji},
  year={2024},
  eprint={2405.17251},
  archivePrefix={arXiv},
  primaryClass={cs.CV},
  url={https://arxiv.org/abs/2405.17251}, 
}
```

## Acknowledgements

Our codes are based on [Moore-AnimateAnyone](https://github.com/MooreThreads/Moore-AnimateAnyone) and other repositories it is based on. We thank the authors of relevant repositories and papers.
