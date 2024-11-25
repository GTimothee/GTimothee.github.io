---
layout: post
title: Embedding model fine-tuning
date: 2024-11-24 00:00:00
description: Embedding model fine-tuning
tags: embedding model training fine-tuning rag
categories: rag
---

> In this blog post I show you how I fine-tune an embedding model for RAG.

## Motivation
This post is an updated version of an [old post](https://www.philschmid.de/fine-tune-embedding-model-for-rag) from Philip Schmidt. Indeed, I recently used his post as a basis to train my own embedding model for a RAG system. However, I faced two major challenges:
- allowing PEFT and LoRa so that the training fits on a smaller GPU
- adding my custom tokens so that the model understand technical documentation

This blog post will follow these steps:
1. try to reproduce the initial blog post
2. add peft and LoRa
3. add caching
4. add custom tokens

At each step we will be evaluating the results and comparing them to the initial results.

## Reproducing the initial blog post

 // in progress

tf32=True,                                  # use tf32 precision
bf16=True,                                  # use bf16 precision

creates error 

ValueError: --tf32 requires Ampere or a newer GPU arch, cuda>=11 and torch>=1.7

## References
- [Philip Schmidt's post](https://www.philschmid.de/fine-tune-embedding-model-for-rag)
