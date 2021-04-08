---
layout: post
title:  "Writing Custom CUDA Kernels for PyTorch"
date:   2021-03-21
categories: jekyll update
---

This is a short post about how to invoke a custom C++ CUDA Kernel from Python/Torch.

A CUDA kernel is a C++ function that runs on an NVidia GPU. 
GPU architecture differs from CPU architecture in that you have vastly more (hardware) threads to play with 
(and hence opportunities for far greater parallelism) 

Consider the task of adding two very large vectors, in CPU land you might do this in a for loop:

```python
import random

# first initialize two large arrays of floats with random values
a = []
b = []
for i in range(0, 10000):
	a[i] = random.uniform(10.5, 75.5)
	b[i] = random.uniform(10.5, 75.5)

# now add them up:
c = []
for i in range(0, 10000):
	c[i] = a[i]+b[i]

```

The runtime efficiency for this addition is O(N) .. works great for small N ...
But for VERY large arrays, you would naturally start to think about ways to optimize it.
Using NumPy might be a first good option as that is a highly optimized library for doing this sort of thing. Another 
option might be to utilize threads on your CPU ... let's say you have a machine with 4 cores, so you have the possibility
of at most, 4 threads running in parallel right there... so why not partition the input arrays into four partitions and process
those in parallel on separate threads? That would surely give you a boost...
Taking that idea further, wouldn't it be great if the CPU had 10000 threads .. so that you could do each float addition
in parallel on it's own thread! Well typically CPU's don't have that many threads available .. and beyond the number of cores
in your machine, your thread scheduler would start to kick in and manage your threads, thus reducing the amount of true
parallelism achieved.

But GPUs *do* have that many threads! So what we can do is take advantage of them by running a kernel function that does 
the addition on the GPU:

Here is the vector addition function written as a C++ template function in CUDA: 
```c++
template <typename T>
__global__ void sum(T *a, T *b, T *c, int N) {
  int i = blockIdx.x * blockDim.x + threadIdx.x;
  if (i <= N) {
    c[i] = a[i] + b[i];
  }
}

```
This function (aka kernel) will run on the GPU device, where you have concepts such as grids, which contain blocks, which 
in turn contain threads, lots of threads .. and instead of using a single thread in a for loop to sum the vectors, here 
we assume we have enough threads executing this kernel to cover every single element in the input vectors - so then 
we just have to figure out which index a particular thread is operating on:
```c++
int i = blockIdx.x * blockDim.x + threadIdx.x;
```

To invoke this kernel function from the host (CPU) we use some special <<< >>> syntax:
```c++
#include <cuda_runtime.h>
#include <stdexcept>
#include <algorithm>

constexpr int CUDA_NUM_THREADS = 128;

constexpr int MAXIMUM_NUM_BLOCKS = 4096;

inline int GET_BLOCKS(const int N) {
  return std::max(std::min((N + CUDA_NUM_THREADS - 1) / CUDA_NUM_THREADS,
           MAXIMUM_NUM_BLOCKS), 1);
}

template <typename T>
void AddGPUKernel(T *in_a, T *in_b, T *out_c, int N, cudaStream_t stream) {
  
  sum<T><<<GET_BLOCKS(N), CUDA_NUM_THREADS, 0, stream>>>(in_a, in_b, out_c, N);

  cudaError_t err = cudaGetLastError();
  if (cudaSuccess != err)
    throw std::runtime_error("CUDA kernel failed : " + std::to_string(err));
}
```

So here you can see that when we invoke the kernel, we specify the number of blocks/threads to use. Obviously you should
check your GPU device properties to see how many threads you can have in a block and so on...

Anyway, so now that we have converted our vector adder into a C++ CUDA kernel, how can we invoke that from a Python/Torch 
program? The answer is to use the [PyBind](https://github.com/pybind/pybind11) library to wrap the kernel in some boilerplate:

```c++
#include <ATen/cuda/CUDAContext.h>
#include <torch/extension.h>
#include <pybind11/pybind11.h>
#include <cuda_runtime.h>

namespace py = pybind11;

// declare templates for front (cpp) and back (cuda) sides of function:
template <typename T>
void AddGPUKernel(T *in_a, T *in_b, T *out_c, int N, cudaStream_t stream);

template <typename T>
void AddGPU(at::Tensor in_a, at::Tensor in_b, at::Tensor out_c) {
  
  int N = in_a.numel();
  
  if (N != in_b.numel())
    throw std::invalid_argument("Size mismatch A.numel(): " + std::to_string(in_a.numel())
          + ", B.numel(): " + std::to_string(in_b.numel()));

  out_c.resize_({N});

  // call the kernel function...
  AddGPUKernel<T>(in_a.data_ptr<T>(), 
                  in_b.data_ptr<T>(),
                  out_c.data_ptr<T>(), 
                  N, 
                  at::cuda::getCurrentCUDAStream());
}

// instantiate the CPP template for T=float:
template void AddGPU<float>(at::Tensor in_a, at::Tensor in_b, at::Tensor out_c);

// declare the extension module with the AddGPU function:
PYBIND11_MODULE(TORCH_EXTENSION_NAME, m){
  m.doc() = "pybind11 example plugin";
  m.def("AddGPU", &AddGPU<float>);
}
```
Also note the use of PyTorch's [ATen](https://pytorch.org/cppdocs/) library here to represent the input/output vectors as tensor<T>'s
for simplicity .. as this library encapsulates the tedious host<-->device memory allocation/copying that needs to happen..

Finally you need to compile the CUDA code and bind it to an actual Python module:
```python
import os
from setuptools import setup
from torch.utils.cpp_extension import CUDAExtension, BuildExtension

os.system('make -j%d' % os.cpu_count())

setup(
    name='CustomPyTorchCUDAKernel',
    version='1.0.0',
    install_requires=['torch'],
    packages=['CustomPyTorchCUDAKernel'],
    package_dir={'CustomPyTorchCUDAKernel': './'},
    ext_modules=[
        CUDAExtension(
            name='CustomPyTorchCUDAKernelBackend',
            include_dirs=['./'],
            sources=[
                'pybind/bind.cpp',
            ],
            libraries=['make_pytorch'],
            library_dirs=['objs'],
        )
    ],
    cmdclass={'build_ext': BuildExtension},
    author='Aaron Cody',
    author_email='aaron@aaroncody.com',
    description='Custom PyTorch CUDA Extension',
    keywords='PyTorch CUDA Extension',
    url='https://github.com/miramar-labs/cuda-torch-custom-kernel',
    zip_safe=False,
)
```

With all these pieces in place, you can now call the GPU-optimized vector addition directly from Python:

```python
import torch
import CustomPyTorchCUDAKernel as CustCUDA

if torch.cuda.is_available():
    a = torch.cuda.FloatTensor(4)
    b = torch.cuda.FloatTensor(4)
    a.normal_()
    b.normal_()
    c = CustCUDA.add_gpu(a, b)
    print(a, b, c)
```

That's it!

Code repo [here](https://github.com/miramar-labs/cuda-torch-custom-kernel)


