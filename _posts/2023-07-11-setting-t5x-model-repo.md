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

TBD...get yourself a folder, init a repo in it.

# Dataset preparation

Let's get a toy dataset first. If you check the T5x source code, you'll find many different datasets used, [WMT](https://www.tensorflow.org/datasets/catalog/wmt19_translate), [c4](https://www.tensorflow.org/datasets/catalog/c4), [the pile](https://pile.eleuther.ai). The tasks definition get the most of those datasets from [Tensorflow Datasets archive](https://www.tensorflow.org/datasets), which is pretty convenient when the dataset is published there and you have no need for anything else. Yet, if you want to bring some customization to an existing dataset, or acquire the dataset from another source, like huggingface datasets, you have to introduce your custom seqio dataset generator. We'll get to that here. 

I have deliberately divided the dataset preparation and usage into two distinct phases. While these steps could be merged and optimized considerably through techniques such as streaming, batching, and parallelism, my method of adopting a multi-step process ensures simplicity and clarity. This leaves room for future improvements, allowing adjustments to be made according to your specific needs.

## Step 0: getting some data locally to play with

As a first step we acquire a small dataset in an easy-to-digest form, so we pick it after with a T5x trainer. I 


## SeqIO tasks

[SeqIO](https://github.com/google/seqio) readme describes the library as "Task-based datasets, preprocessing, and evaluation for sequence models". With SeqIO you can define, where does your dataset come from (text file, Tensorflow dataset, or some generator function), and what are the processing steps before the data is fed into the model. The steps 


## Step 1: wrapping the data into seqio task