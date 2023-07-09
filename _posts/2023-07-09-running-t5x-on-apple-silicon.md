---
layout: post
title: Running t5x on macos
---

# Background

If you are in language model space, you cannot avoid 🤗 transformers and other libraries from that ecosystem. From my experience, those libraries are top-quality, they provide invaluable service to the community, and the whole 🤗 story is a great example of how open-source has to be. Also, if you are into reading source code and want to get better with complex python codebases, [transformers github repo](https://github.com/huggingface/transformers.git) is the place to learn best practices.

Yet, 🤗 transformers is not the only library in the space, and some others might suit better for more special cases. Prior to 🤗 transformers, you could stick to [t2t](https://github.com/tensorflow/tensor2tensor) which seems to be succeeded by [trax](https://github.com/google/trax) these days, [fairseq](https://github.com/facebookresearch/fairseq) was also used to produce many papers and models. Several of the modern high-performing LMs, such as [flan-t5-family](https://huggingface.co/docs/transformers/model_doc/flan-t5) and [ul2](https://huggingface.co/docs/transformers/model_doc/ul2) were pretrained with [t5x](https://github.com/google-research/t5x). 

<!--more-->

[T5x](https://github.com/google-research/t5x) is an all-in-one toolset to pretrain, fine-tune, evaluate and run inference for LMs of various architectures.  As it is built on top of [Jax](https://github.com/google/jax) for dealing with neural networks and tensors, GPUs and TPUs are supported, with the latter being utilized very efficiently. It also uses [seqio](https://github.com/google/seqio) for handling and processing datasets. The depth of dependency tree is pretty large though, it relies on [tensorflow](https://github.com/tensorflow/tensorflow), [gin-config](https://github.com/google/gin-config), [gcs-related libraries](https://github.com/googleapis/python-storage), [sentencepiece tokenizer](https://github.com/google/sentencepiece) and many many others. Number of dependencies is positively correlated with the chance of the library not working smoothly on some of the platforms, which happened in this case. [Tensorflow-text](https://github.com/tensorflow/text) package, which is used to wrap and process textual data as tensorflow tensors, has [known](https://github.com/tensorflow/text/issues/1077) [issues](https://developer.apple.com/forums/thread/700906) to install on macos with apple silicon. Luckily, due to contributions from the open source community, and especially [sun1638650145](https://github.com/sun1638650145) we can use tensorflow-text, and subsequently T5x with macs. The process is not as smooth as `pip install`, yet doable, and lets one prototype, debug and even train small models without having continuous access to an external machine.

In this post I provide a step-by-step guide, which is "works-on-my-machine" certified. Also, given how quickly things move with library versions and the ecosystem, the guide might need some adjustments for package versions, and maybe even dependencies in the future. The way to run T5x might come magical or too complicated, there's a good reason for it, I might cover it in some later posts.

I use conda python, but any python 3.10 installation should do (pyenv or brew, I guess). Also this guide currently confirmed to work for cpu version of Jax. I do not see any reason for it not to work for apple silicon GPU, but I have some issues on my machine, and CPU version works fine for some light debugging and prototyping anyway.
Tensorflow for macos will give us extra issues and warnings during the installation, but as we do not use tensorflow itself for training here, it is fine. If you plan to use tensorflow later, ~~I pity you~~ check [here](https://developer.apple.com/metal/tensorflow-plugin/), but I cannot say for sure if everything would work inside of this environment later.

# Points to consider

- The whole exercise takes about 9GB on disk, together with venv, repositores and example dataset.
- I tend to use `python -m pip ...` everywhere, due to pip softlink being frequently broken in the environments I work in. If you are confident, that's not the case and you're not up for typing extra, feel free to just call `pip` directly.

## Step-by-step guide

### init and activate your python environment

For the sake of exercise, I named my T5x space root directory `t5x_tftext_free` .

	- `mkdir t5x_tftext_free && cd t5x_tftext_free`
	- `python -m venv ./.venv`
	- `. ./.venv/bin/activate`
	- `python -m pip install --upgrade pip`

### Put libraries into your environment

#### Setting  up tensorflow-text

We start with the package causing issues with T5x on macos.

- get tensorflow-text wheel from [the prebuilt releases](https://github.com/sun1638650145/Libraries-and-Extensions-for-TensorFlow-for-Apple-Silicon/releases)
- Install it into your environment from the place where your package got downloaded into `python -m pip install -I ~/Downloads/tensorflow_text-2.12.1-cp310-cp310-macosx_11_0_arm64.whl`

#### Setting up seqio

- `git clone https://github.com/google/seqio.git`
- Edit `setup.py` in the cloned repo, find `tensorflow-text` inside of `install_requires` list and comment it out.
- install the library with `python -m pip install -e ./seqio` (still, assuming you are inside of the space root directory).

#### Setting up t5x

- `git clone https://github.com/google-research/t5x.git`
- Make two changes in `./t5x/setup.py`:
	- Find `seqio` dependency and comment it out
	- change tensorflow-cpu to tensorflow
- Install it with `python -m pip install -e ./t5x/`

#### Setting up t5

The repository for the original [T5 paper](https://arxiv.org/abs/1910.10683) was based on [tensorflow-mesh](https://github.com/tensorflow/mesh). Despite being deprecated, it is still partially used for T5x, as some of the text preprocessing tasks didn't change. We'll have to install it as a dependency.

- `git clone https://github.com/google-research/text-to-text-transfer-transformer.git`
- edit `text-to-text-transfer-transformer/setup.py` and comment out `seqio-nightly`
- `python -m pip install -e ./text-to-text-transfer-transformer`

#### Fixing tensorflow version

At this stage you'll get tensorflow 2.13.X installed, which is incompatible with tensorflow-text that you got from the custom built wheel in the beginning. You have to force your version back to 2.12.0 by `python -m pip install tensorflow-macos==2.12.0`.

After you do that, check the version by running
```python
import tensorflow as tf
print(tf.__version__)
```
in the python shell.

#### Extra libraries that might come handy
- you might also want to install `ipython` for checking things on the go. Feel free to go with jupyter(lab) though. I prefer `ipython` as it forces me to put anything I want to save into a script file, rather than have a huge pile of unmaintained and disconnected notebooks. So `python -m pip install ipython`
- `tqdm` comes together with other dependencies, but you might still want to ensure that you have it as things might change in the future (`python -m pip install tqdm`)
- You'll want to track the training process later, set up `tensorboard`. `python -m pip install tensorboard`.

### Get things running

#### Get a dataset for training

Get yourself a dataset for testing things out. We'll use a popular machine translation dataset to simplify things, as there's already a t5x model configuration prepared for it.
- `mkdir tfds_data`
- `export TFDS_DATA_DIR=$(pwd)/tfds_data`
	- Keep in mind, that many of the t5x and t5x-related mechanisms (jax included) have hard time dealing with relative paths, so put an absolute path here and for anything below too, and also do not use relative references in absolute paths as well (as in `/home/user/workspace/something/../something_else/` will cause problems)
- `tfds build wmt_t2t_translate --data_dir=$TFDS_DATA_DIR --max_examples_per_split 1000000` (limit here is both for speed and due to an error of too many open files if I go without it)
	- this one will take a while. You'll see some warning and errors about the lack of google cloud credentials (if you don't have those configured), that's fine.

#### Set the model folder and configuration

Prepare a folder for your model, feel free to use a different folder name, yet this model is for testing only, so there's no reason to keep it for later.
- `mkdir model_artifacts`
- `export MODEL_DIR=$(pwd)/model_artifacts`

Now to the model configuration
- edit `./t5x/t5x/examples/t5/t5_1_1/examples/base_wmt_from_scratch.gin`, find `include "t5x/examples/t5/t5_1_1/base.gin"` and use `tiny.gin` instead of `base.gin` to speed up the training run.

#### Run the training

- `export T5X_DIR=$(pwd)/t5x`
- `python ${T5X_DIR}/t5x/train.py --gin_file="t5x/examples/t5/t5_1_1/examples/base_wmt_from_scratch.gin"  --gin.MODEL_DIR=\"${MODEL_DIR}\" --tfds_data_dir=${TFDS_DATA_DIR}`
- Wait for a bit to see the training steps succeeding

You should see output similar to this.

```
I0709 15:29:52.025439 8383831552 train.py:613] BEGIN Train loop.
I0709 15:29:52.025554 12803141632 logging_writer.py:48] [0] collection=train timing/train_iter_warmup=1.90735e-06
I0709 15:29:52.025690 8383831552 train.py:618] Training for 500 steps.
I0709 15:29:52.026439 8383831552 trainer.py:509] Training: step 0
I0709 15:30:02.658104 8383831552 trainer.py:509] Training: step 1
I0709 15:30:22.038341 8383831552 trainer.py:509] Training: step 3
```

If you see it, feel free to `ctrl+c` out of it, and delete both, the model and the tfds directory contents. Those were used only to confirm that the library works.

Enjoy building your next great model!