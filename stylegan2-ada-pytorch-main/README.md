## StyleGAN2-ADA &mdash; PyTorch implementation to generate synthetic kidney Glomeruli images


** Based on Training Generative Adversarial Networks with Limited Data**<br>
Tero Karras, Miika Aittala, Janne Hellsten, Samuli Laine, Jaakko Lehtinen, Timo Aila<br>
https://arxiv.org/abs/2006.06676<br>

## Release notes

This repository is a faithful reimplementation of [StyleGAN2-ADA](https://github.com/NVlabs/stylegan2-ada/) in PyTorch, focusing on correctness, performance, and compatibility.

**Correctness**
* Full support for all primary training configurations.
* Extensive verification of image quality, training curves, and quality metrics against the TensorFlow version.
* Results are expected to match in all cases, excluding the effects of pseudo-random numbers and floating-point arithmetic.

**Performance**
* Training is typically 5%&ndash;30% faster compared to the TensorFlow version on NVIDIA Tesla V100 GPUs.
* Inference is up to 35% faster in high resolutions, but it may be slightly slower in low resolutions.
* GPU memory usage is comparable to the TensorFlow version.
* Faster startup time when training new networks (<50s), and also when using pre-trained networks (<4s).
* New command line options for tweaking the training performance.

**Compatibility**
* Compatible with old network pickles created using the TensorFlow version.
* New ZIP/PNG based dataset format for maximal interoperability with existing 3rd party tools.
* TFRecords datasets are no longer supported &mdash; they need to be converted to the new format.
* New JSON-based format for logs, metrics, and training curves.
* Training curves are also exported in the old TFEvents format if TensorBoard is installed.
* Command line syntax is mostly unchanged, with a few exceptions (e.g., `dataset_tool.py`).
* Comparison methods are not supported (`--cmethod`, `--dcap`, `--cfg=cifarbaseline`, `--aug=adarv`)
* **Truncation is now disabled by default.**

## Data repository

| Path | Description
| :--- | :----------
| [stylegan2-ada-pytorch](https://drive.google.com/drive/folders/1nB8nRY251_gTd4wDlb1o-m6iPXWH_OHt?usp=sharing) | DataSet for Training 
| &ensp;&ensp;&boxur;&nbsp; [pretrained](https://drive.google.com/drive/folders/1nB8nRY251_gTd4wDlb1o-m6iPXWH_OHt?usp=sharing) | Pre-trained models

## Requirements

* Linux and Windows are supported, but we recommend Linux for performance and compatibility reasons.
* 1&ndash;8 high-end NVIDIA GPUs with at least 12 GB of memory. We have done all testing and development using NVIDIA DGX-1 with 8 Tesla V100 GPUs.
* 64-bit Python 3.7 and PyTorch 1.7.1. See [https://pytorch.org/](https://pytorch.org/) for PyTorch install instructions.
* CUDA toolkit 11.0 or later.  Use at least version 11.1 if running on RTX 3090.  (Why is a separate CUDA toolkit installation required?  See comments in [#2](https://github.com/NVlabs/stylegan2-ada-pytorch/issues/2#issuecomment-779457121).)
* Python libraries: `pip install click requests tqdm pyspng ninja imageio-ffmpeg==0.4.3`.  We use the Anaconda3 2020.11 distribution which installs most of these by default.
* Docker users: use the [provided Dockerfile](./Dockerfile) to build an image with the required library dependencies.

The code relies heavily on custom PyTorch extensions that are compiled on the fly using NVCC. On Windows, the compilation requires Microsoft Visual Studio. We recommend installing [Visual Studio Community Edition](https://visualstudio.microsoft.com/vs/) and adding it into `PATH` using `"C:\Program Files (x86)\Microsoft Visual Studio\<VERSION>\Community\VC\Auxiliary\Build\vcvars64.bat"`.

## Getting started

Pre-trained networks are stored as `*.pkl` files that can be referenced using local filenames or URLs:
```
python generate.py --outdir=out --trunc=1 --seeds=85,265,297,849 \
    --network=(URL or file-path of pre-trained downloaded from link above)
```
Outputs from the above commands are placed under `out/*.png`, controlled by `--outdir`. 



## Using networks from Python

You can use pre-trained networks in your own Python code as follows:

```.python
with open('ffhq.pkl', 'rb') as f:
    G = pickle.load(f)['G_ema'].cuda()  # torch.nn.Module
z = torch.randn([1, G.z_dim]).cuda()    # latent codes
c = None                                # class labels (not used in this example)
img = G(z, c)                           # NCHW, float32, dynamic range [-1, +1]
```


## Preparing datasets

Datasets are stored as uncompressed ZIP archives containing uncompressed PNG files and a metadata file `dataset.json` for labels.

Custom datasets can be created from a folder containing images; see [`python dataset_tool.py --help`](./docs/dataset-tool-help.txt) for more information. Alternatively, the folder can also be used directly as a dataset, without running it through `dataset_tool.py` first, but doing so may lead to suboptimal performance.


## Training new networks

In its most basic form, training new networks boils down to:

```.bash
python train.py --outdir=~/training-runs --data=~/mydataset.zip --gpus=1 --dry-run
python train.py --outdir=~/training-runs --data=~/mydataset.zip --gpus=1
```

The first command is optional; it validates the arguments, prints out the training configuration, and exits. The second command kicks off the actual training.

In this example, the results are saved to a newly created directory `~/training-runs/<ID>-mydataset-auto1`, controlled by `--outdir`. The training exports network pickles (`network-snapshot-<INT>.pkl`) and example images (`fakes<INT>.png`) at regular intervals (controlled by `--snap`). For each pickle, it also evaluates FID (controlled by `--metrics`) and logs the resulting scores in `metric-fid50k_full.jsonl` (as well as TFEvents if TensorBoard is installed).

The name of the output directory reflects the training configuration. For example, `00000-mydataset-auto1` indicates that the *base configuration* was `auto1`, meaning that the hyperparameters were selected automatically for training on one GPU. The base configuration is controlled by `--cfg`:


Please refer to [`python train.py --help`](./docs/train-help.txt) for the full list.

## Expected training time

The total training time depends heavily on resolution, number of GPUs, dataset, desired quality, and hyperparameters. The following table lists expected wallclock times to reach different points in the training, measured in thousands of real images shown to the discriminator ("kimg"):

| Resolution | GPUs | 1000 kimg | 25000 kimg | sec/kimg          | GPU mem | CPU mem
| :--------: | :--: | :-------: | :--------: | :---------------: | :-----: | :-----:
| 128x128    | 1    | 4h 05m    | 4d 06h     | 12.8&ndash;13.7   | 7.2 GB  | 3.9 GB
| 128x128    | 2    | 2h 06m    | 2d 04h     | 6.5&ndash;6.8     | 7.4 GB  | 7.9 GB
| 128x128    | 4    | 1h 20m    | 1d 09h     | 4.1&ndash;4.6     | 4.2 GB  | 16.3 GB
| 128x128    | 8    | 1h 13m    | 1d 06h     | 3.9&ndash;4.9     | 2.6 GB  | 31.9 GB
| 256x256    | 1    | 6h 36m    | 6d 21h     | 21.6&ndash;24.2   | 5.0 GB  | 4.5 GB
| 256x256    | 2    | 3h 27m    | 3d 14h     | 11.2&ndash;11.8   | 5.2 GB  | 9.0 GB
| 256x256    | 4    | 1h 45m    | 1d 20h     | 5.6&ndash;5.9     | 5.2 GB  | 17.8 GB
| 256x256    | 8    | 1h 24m    | 1d 11h     | 4.4&ndash;5.5     | 3.2 GB  | 34.7 GB
| 512x512    | 1    | 21h 03m   | 21d 22h    | 72.5&ndash;74.9   | 7.6 GB  | 5.0 GB
| 512x512    | 2    | 10h 59m   | 11d 10h    | 37.7&ndash;40.0   | 7.8 GB  | 9.8 GB
| 512x512    | 4    | 5h 29m    | 5d 17h     | 18.7&ndash;19.1   | 7.9 GB  | 17.7 GB
| 512x512    | 8    | 2h 48m    | 2d 22h     | 9.5&ndash;9.7     | 7.8 GB  | 38.2 GB
| 1024x1024  | 1    | 1d 20h    | 46d 03h    | 154.3&ndash;161.6 | 8.1 GB  | 5.3 GB
| 1024x1024  | 2    | 23h 09m   | 24d 02h    | 80.6&ndash;86.2   | 8.6 GB  | 11.9 GB
| 1024x1024  | 4    | 11h 36m   | 12d 02h    | 40.1&ndash;40.8   | 8.4 GB  | 21.9 GB
| 1024x1024  | 8    | 5h 54m    | 6d 03h     | 20.2&ndash;20.6   | 8.3 GB  | 44.7 GB

The above measurements were done using NVIDIA Tesla V100 GPUs with default settings (`--cfg=auto --aug=ada --metrics=fid50k_full`). "sec/kimg" shows the expected range of variation in raw training performance, as reported in `log.txt`. "GPU mem" and "CPU mem" show the highest observed memory consumption, excluding the peak at the beginning caused by `torch.backends.cudnn.benchmark`.


```



References:
1. [GANs Trained by a Two Time-Scale Update Rule Converge to a Local Nash Equilibrium](https://arxiv.org/abs/1706.08500), Heusel et al. 2017
2. [Demystifying MMD GANs](https://arxiv.org/abs/1801.01401), Bi&nacute;kowski et al. 2018
3. [Improved Precision and Recall Metric for Assessing Generative Models](https://arxiv.org/abs/1904.06991), Kynk&auml;&auml;nniemi et al. 2019
4. [Improved Techniques for Training GANs](https://arxiv.org/abs/1606.03498), Salimans et al. 2016
5. [A Style-Based Generator Architecture for Generative Adversarial Networks](https://arxiv.org/abs/1812.04948), Karras et al. 2018

## License

Copyright &copy; 2021, NVIDIA Corporation. All rights reserved.

This work is made available under the [Nvidia Source Code License](https://nvlabs.github.io/stylegan2-ada-pytorch/license.html).

## Citation

```
@inproceedings{Karras2020ada,
  title     = {Training Generative Adversarial Networks with Limited Data},
  author    = {Tero Karras and Miika Aittala and Janne Hellsten and Samuli Laine and Jaakko Lehtinen and Timo Aila},
  booktitle = {Proc. NeurIPS},
  year      = {2020}
}
```

## Development

This is a research reference implementation and is treated as a one-time code drop. As such, we do not accept outside code contributions in the form of pull requests.
