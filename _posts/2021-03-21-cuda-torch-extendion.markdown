---
layout: post
title:  "Writing Custom CUDA Kernels for PyTorch"
date:   2021-03-21
categories: jekyll update
---

Earlier this year I delved into the world of 'GPU Optimization' - specifically using raw CUDA kernels to optimize 
bottlenecks in my PyTorch code. In this post I plan to cover a number of topics:

- PyTorch boilerplate for exposing CPP/CUDA functions
- CUDA Kernel functions & architecture (grids/blocks/threads/memory etc)
- Optimization techniques
- Profiling techniques

- initial code repo [here](https://github.com/miramar-labs/cuda-torch-custom-kernel)

Coming soon ... ;)