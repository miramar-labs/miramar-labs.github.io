---
layout: post
title:  "Configuring VSCode for building/debugging CUDA code on Linux"
date:   2021-03-13
categories: jekyll update
---
Recently I have been migrating away from JetBrains IDEs in favor of VSCode .. because of it's excellent 
multi-language support ... (with JetBrains it seems you almost need a different IDE for every language these days).
With VSCode I can create projects with C/C++/Java/Scala/Python/Go/BASH code all in one shot - it's great!
This weekend I wanted to mess around with CUDA, so I looked into how to go about setting up that toolchain in VSCode.

Note: This post assumes a few things:
- you are running this on a machine that has an NVIDIA GPU
- the machine has the latest [NVIDIA Driver](https://www.nvidia.com/en-us/drivers/unix/) installed
- the machine has the [NVIDIA CUDA Toolkit](https://docs.nvidia.com/cuda/index.html) installed


Ok so back to VSCode - eventually I got it working by adding a few custom json settings files to my workspace:

First of all, CUDA source code files have .cu and .cuh extensions... the syntax is basically C++ but with some minor 
additions, so to get syntax highlighting working on those file types I added the following to my settings.json:

 
__CTRL+SHIFT+P: Preferences: Open Settings (JSON)__

	"files.associations": {
        "*.cu":"cpp" ,
        "*.cuh":"cpp"  
    }

Next I had to tell VSCode how to build these files - CUDA comes with it's own modified version of gcc, called nvcc, so
what you do is create a build task:

tasks.json:

```json
{
    "tasks": [
        {
            "type": "shell",
            "label": "CUDA C/C++: build active file",
            "command": "nvcc",
            "args": [
                "-g",
                "-G",
                "${file}",
                "-I${workspaceFolder}/Common",
                "-o",
                "${fileDirname}/${fileBasenameNoExtension}"
            ],
            "options": {
                "cwd": "${workspaceFolder}"
            }
        }
    ],
    "version": "2.0.0"
}
```

This allows me to quickly build the active .cu file. Note the addition of the -G flag, which generates the right symbols 
for stepping into the actual CUDA kernel code in the debugger.

For debugging, CUDA has it's own modified version of gdb (cuda-gdb), so now we need a launcher:

launch.json:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "CUDA C/C++: build and debug active file",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}/${fileBasenameNoExtension}",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "setupCommands": [
              {
                "description": "Enable pretty-printing for gdb",
                "text": "-enable-pretty-printing",
                "ignoreFailures": true
              }
            ],
            "preLaunchTask": "CUDA C/C++: build active file",
            "miDebuggerPath": "/usr/local/cuda/bin/cuda-gdb"
          }
    ]
}
```

what this does is first build the code by calling the build task in `preLaunchTask` and then runs that in `cuda-gdb` ...
The result is the ability to set breakpoints, not only in the CPU code but also in the Kernel code :

![CUDA Debugging](/assets/images/cuda-dbg.png)

__Another WIN for VSCode !__  

##### TROUBLESHOOTING

On my Ubuntu 20.04 box, I had libncurses6 packages installed by default. I also use CUDA 11.0 (as that works well with Tensorflow). 
However, initially I wasn't able to get `cuda-gdb` to run because of it's dependencies:
	
	aaron@blade:~$ ldd `which cuda-gdb`
	
	linux-vdso.so.1 (0x00007ffc16f4e000)
	libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007f36fe8fe000)
	libncursesw.so.5 => /lib/x86_64-linux-gnu/libncursesw.so.5 (0x00007f36fe8cc000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f36fe8c6000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f36fe777000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f36fe754000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f36fe562000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f36fe947000)

as you can see it depends on libncurses5, so you may need to install that too, for example:

	sudo apt-get --reinstall install libncurses5 libncursesw5