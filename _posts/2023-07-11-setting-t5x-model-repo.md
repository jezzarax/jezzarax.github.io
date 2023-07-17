---
layout: post
title: WIP - Setting a repository for t5x model
---

# Intro

My [previous post](./running-t5x-on-apple-silicon/) felt a bit unfinished without providing some idea how to set some folder structure for a new t5x model project. I'll give a rough idea here.

T5x gives you a lot of powerful tools and the source code has some insightful examples, yet here I want to achieve something simpler: 

- have a minimal set of files to prepare and consume a custom dataset
- a seqio task that would consume the dataset
- a t5x model definition to be pretrained with the dataset

# Setting the base

We take off where we finished at the end of the [last post](./running-t5x-on-apple-silicon/), having a folder with multiple repositories in it. Let's get one more, this time, it's going to be our own repository with pieces of code and configuration which get us a model.

```bash

mkdir -p t5x-playground && cd t5x-playground
git init

```

T5x is set in such a way, that having our model code being a python package is a very convenient thing to do. If you need a refresher, just take some [reading](https://packaging.python.org/en/latest/tutorials/packaging-projects/), it pays off well. Let's set a package there:

```bash
mkdir t5xpg
touch ./setup.py
touch ./t5xpg/__init__.py
```

And put a basic skeleton into the `setup.py` (adjust as you see necessary).

```python

import os
import sys
import setuptools

setuptools.setup(
    name='t5xpg',
    version="0.0.1",
    description="Basic set of t5x configs and tasks for a simple model",
    author='Alexey Kuntsevich',
    author_email='forspam@nowhere.com',
    url='http://github.com/jezzarax/t5xpg',
    license='Apache 2.0',
    packages=setuptools.find_packages(),
    include_package_data=True,
    scripts=[],
    install_requires=[],
    classifiers=[
        'Topic :: Scientific/Engineering :: Artificial Intelligence',
    ],
    keywords='sequence preprocessing nlp machinelearning',
)

```

Make the package visible within your virtualenv (you should have it present and activated since the previous post, that's important).

```bash
python -m pip install -e .
```

We have an empty package, feel free to stage, commit and push into your own repository as necessary.

# Dataset preparation

Let's get a toy dataset first. If you check the T5x source code, you'll find many different datasets used, [WMT](https://www.tensorflow.org/datasets/catalog/wmt19_translate), [c4](https://www.tensorflow.org/datasets/catalog/c4), [the pile](https://pile.eleuther.ai). The tasks definition get the most of those datasets from [Tensorflow Datasets archive](https://www.tensorflow.org/datasets), which is pretty convenient when the dataset is published there and you have no need for anything else. Yet, if you want to bring some customization to an existing dataset, or acquire the dataset from another source, like huggingface datasets, you have to introduce your custom seqio dataset generator. We'll get to that here. 

I have deliberately divided the dataset preparation and usage into two distinct phases. While these steps could be merged and optimized considerably through techniques such as streaming, batching, and parallelism, my method of adopting a multi-step process ensures simplicity and clarity. This leaves room for future improvements, allowing adjustments to be made according to your specific needs.

## Step 0: getting some data locally to play with

As a first step we acquire a small dataset in an easy-to-digest form, so we pick it after with the T5x trainer. Wikipedia is a good starting point, and there is already a [dataset](https://huggingface.co/datasets/wikitext) with a preselected set of high-quality english wikipedia articles (lucky us to have it!). Please note, that it's a very small dataset in comparison to what is generally used for the model pretraining, so we take it as a toy example. Setting a real pretraining process would require orders of magnitude more data, proper data cleaning and streaming, and a serious engineering for storing and loading of it. Yet, this dataset would fit for getting our hands dirty, or for cases when you want a small model for some specific purpose.

Get some dependencies installed, so we can download the dataset:

```bash
python -m pip install datasets tqdm
```

Here's how we can get a snapshot of it in a json-lines format. It's not a very space-efficient format, but a simple and a universal one, so you could easily prepare your own dataset later. Create the following script in the repository folder and run it after.

```python
from datasets import load_dataset
from tqdm import tqdm
import json

dataset_name = "wikitext"
dataset_subset_name = "wikitext-103-raw-v1"

ds = load_dataset(dataset_name, dataset_subset_name)
filtered_ds = ds.filter(lambda x: len(x["text"]) > 100)


for split_name in filtered_ds.keys():
    with open(f"./wikitext_{split_name}.jsonl", 'w') as ofh:
        for item in tqdm(filtered_ds[split_name], desc=f"writing {split_name} split"):
            json.dump(item, ofh)
            ofh.write("\n")
```

It'll create you three files: `wikitext_train.jsonl`, `wikitext_test.jsonl` and `wikitext_validation.jsonl`. If you feel adventurous, concatenate train and test, we'll use the other file for model validation.

## Step 1: wrapping the data into seqio task

So we have our dataset ready, let's prepare a piece of code that will feed the data from it into our model. 

Inside of the nested `t5xpg` folder (on the same level with the `__init__.py` file) create `tasks.py` and put there the following:

```python

import functools, json

import seqio
import tensorflow as tf
import t5.data
from t5.data import preprocessors
from seqio import FunctionDataSource, utils

TaskRegistry = seqio.TaskRegistry
vocabulary = seqio.ByteVocabulary()

DEFAULT_OUTPUT_FEATURES = {
    "inputs": seqio.Feature(vocabulary=vocabulary, add_eos=True, required=False),
    "targets": seqio.Feature(vocabulary=vocabulary, add_eos=True),
}

def dataset_rows_generator(split, limit_rec=None):
    recs_generated = 0
    while True and (limit_rec is None or recs_generated < limit_rec):
        with open(f"./wikitext_{split}.jsonl", "r") as dataset_fh:
            for line in dataset_fh:
                row = json.loads(line)
                recs_generated += 1
                if limit_rec is None or recs_generated < limit_rec:
                    yield row["text"]
                else:
                    break

def dataset_generator(split, shuffle_files, seed=None, dataset_params=None, limit_rec=None):
    return tf.data.Dataset.from_generator(
        functools.partial(dataset_rows_generator, split, limit_rec=limit_rec),
        output_signature=tf.TensorSpec(shape=(), dtype=tf.string, name="wikitext"),
    )

@utils.map_over_dataset
def target_to_key(x, key_map, target_key):
    """Assign the value from the dataset to target_key in key_map"""
    return {**key_map, target_key: x}

TaskRegistry.add(
    "wikitext_span_corruption",
    source=seqio.FunctionDataSource(
        dataset_fn=functools.partial(dataset_generator),
        splits=("train", "validation"),
        caching_permitted=False,
        num_input_examples=None,
    ),
    preprocessors=[
        functools.partial(
            target_to_key,
            key_map={
                "inputs": None,
                "targets": None,
            },
            target_key="targets",
        ),
        seqio.preprocessors.tokenize,
        preprocessors.span_corruption,
        seqio.preprocessors.append_eos_after_trim,
    ],
    output_features={"targets": DEFAULT_OUTPUT_FEATURES["targets"]},
    metric_fns=[],
)


```

It's not the easiest piece to consume, but I'll explain it piece by piece below. 

### Intermission 1: SeqIO tasks - WIP

[SeqIO](https://github.com/google/seqio) readme describes the library as "Task-based datasets, preprocessing, and evaluation for sequence models". With SeqIO you can define, where does your dataset come from (text file, Tensorflow dataset, or some generator function), and what are the processing steps before the data is fed into the model. The steps 

### Intermission 2: Span corruptio and T5 paper - WIP


