# diffusers-rs: A Diffusers API in Rust/Torch

[![Build Status](https://github.com/LaurentMazare/diffusers-rs/workflows/Continuous%20integration/badge.svg)](https://github.com/LaurentMazare/diffusers-rs/actions)
[![Latest version](https://img.shields.io/crates/v/diffusers.svg)](https://crates.io/crates/diffusers)
[![Documentation](https://docs.rs/diffusers/badge.svg)](https://docs.rs/diffusers)
![License](https://img.shields.io/crates/l/diffusers.svg)


![rusty robot holding a torch](media/robot13.jpg)

_A rusty robot holding a fire torch_, generated by stable diffusion using Rust and libtorch.

The `diffusers` crate is a Rust equivalent to Huggingface's amazing
[diffusers](https://github.com/huggingface/diffusers) Python library.
It is based on the [tch crate](https://github.com/LaurentMazare/tch-rs/).
The implementation is complete enough so as to be able to run Stable Diffusion
v1.4.

In order to run the models, one has to get the weights, see the details below and can then run
the following command. At each step, some `sd_*.png` image is generated, the last one `sd_30.png`
should be the least noisy one.

```bash
cargo run --example stable-diffusion --features clap -- --prompt "A rusty robot holding a fire torch."
```

The only supported scheduler is the Denoising Diffusion Implicit Model scheduler (DDIM). The
original paper and some code can be found in the [associated repo](https://github.com/ermongroup/ddim).

## FAQ

### Memory Issues

This requires a GPU with more than 8GB of memory, as a fallback the CPU version can be used
but is slower.

```bash
cargo run --example stable-diffusion --features clap -- --prompt "A very rusty robot holding a fire torch." --cpu-for-unet --cpu-for-vae
```

For a GPU with 8GB, one can use the [fp16 weights for the UNet](https://huggingface.co/CompVis/stable-diffusion-v1-4/tree/fp16/unet) and put only the UNet on the GPU.

```bash
PYTORCH_CUDA_ALLOC_CONF=garbage_collection_threshold:0.6,max_split_size_mb:128 RUST_BACKTRACE=1 CARGO_TARGET_DIR=target2 cargo run \
    --example stable-diffusion --features clap -- --cpu-for-vae --unet-weights data/unet-fp16.ot
```

## Examples

A bunch of rusty robots holding some torches!

![rusty robot holding a torch](media/robot3.jpg)
![rusty robot holding a torch](media/robot4.jpg)
![rusty robot holding a torch](media/robot7.jpg)
![rusty robot holding a torch](media/robot8.jpg)
![rusty robot holding a torch](media/robot11.jpg)
![rusty robot holding a torch](media/robot13.jpg)

## Getting the Weights and Vocab File

In order to run this, the weights have to be downloaded, converted to the appropriate
format and copied in the top level `data` directory. There are three set of weights to
download as well as some vocabulary file for the text model.

If there is some interest in having the final weight files available, open an issue and
we could consider packaging them.

First get the vocabulary file and uncompress it.

```bash
mkdir -p data && cd data
wget https://github.com/openai/CLIP/raw/main/clip/bpe_simple_vocab_16e6.txt.gz
gunzip bpe_simple_vocab_16e6.txt.gz
```

### Clip Encoding Weights

For the clip encoding weights, start by downloading the weight file.

```bash
wget https://huggingface.co/openai/clip-vit-large-patch14/resolve/main/pytorch_model.bin
```

Then using Python, load the weights and save them in a `.npz` file.

```python
import numpy as np
import torch
model = torch.load("./pytorch_model.bin")
np.savez("./pytorch_model.npz", **{k: v.numpy() for k, v in model.items() if "text_model" in k})
```

Finally use `tensor-tools` from the [tch repo](https://github.com/LaurentMazare/tch-rs/) to convert
this to a `.ot` file that tch can use.

```bash
cd path/to/tch-rs
cargo run --release --example tensor-tools cp ./data/pytorch_model.npz ./data/pytorch_model.ot
```

### VAE and Unet Weights

The weight files can be downloaded from huggingface's hub but it first requires you to log in (and to [accept the terms of use](https://huggingface.co/CompVis/stable-diffusion-v1-4) the first time). Then you can download the [VAE weights](https://huggingface.co/CompVis/stable-diffusion-v1-4/blob/main/vae/diffusion_pytorch_model.bin) and [Unet weights](https://huggingface.co/CompVis/stable-diffusion-v1-4/blob/main/unet/diffusion_pytorch_model.bin).

After downloading the files, use Python to convert them to `npz` files.

```python
import numpy as np
import torch
model = torch.load("./vae.bin")
np.savez("./vae.npz", **{k: v.numpy() for k, v in model.items()})
model = torch.load("./unet.bin")
np.savez("./unet.npz", **{k: v.numpy() for k, v in model.items()})
```

And again convert this to a `.ot` file via `tensor-tools`.

```bash
cargo run --release --example tensor-tools cp ./data/vae.npz ./data/vae.ot
cargo run --release --example tensor-tools cp ./data/unet.npz ./data/unet.ot
```
