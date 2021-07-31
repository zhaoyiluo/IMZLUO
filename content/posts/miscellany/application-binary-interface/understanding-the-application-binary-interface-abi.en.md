---
title: "Understanding the Application Binary Interface (ABI)"
subtitle: "What is the ABI info when you are using CMake to build your project?"
date: 2021-07-30T21:18:09+08:00
lastmod: 2021-07-30T21:18:09+08:00
draft: false
authors: ["zhaoyiluo"]
description: ""

tags: []
categories: ["Miscellany"]
series: []

featuredImage: ""
featuredImagePreview: ""
---

## Foreword

Recently, I solved a problem that the user had difficulties in building the [Forward Project](https://github.com/Tencent/Forward) and encountered the problem regarding the undefined references. The detailed discussion was conducted in Simplified Chinese, however, I also recommend you to take a look at this [issue](https://github.com/Tencent/Forward/issues/25) about how I solved it if you are comfortable reading something in Chinese. By the way, don't let that issue bother you and this article is mainly about Application Binary Interface (ABI).

So let's get started! Software engineers choose different build tools based on their preferences to take control and build their projects. Some may choose `CMake`, some may choose `Bazel`, and some may choose other build tools. It doesn't matter which build tool you are using, but it does matter if you could understand all the information prompted during the build process if you would like to be an excellent engineer.

Before writing this article, every time when I want to build a project, what I hope to see is that after I run the command `cmake .. [any other options]`, the console needs to show me something like `Build files have been written to: [some place]`. Nothing else. Below shows an example of how I build the Forward Project. 

```bash
zhaoyiluo@pc:~/Forward/build$ cmake ..  -DTensorRT_ROOT=/home/zhaoyiluo/libs/TensorRT-7.2.1.6 -DENABLE_PROFILING=ON -DENABLE_DYNAMIC_BATCH=ON -DENABLE_TENSORFLOW=ON -DENABLE_UNIT_TESTS=ON
-- The C compiler identification is GNU 9.1.0
-- The CXX compiler identification is GNU 9.1.0
-- The CUDA compiler identification is NVIDIA 11.1.105
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc - works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ - works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Check for working CUDA compiler: /usr/local/cuda/bin/nvcc
-- Check for working CUDA compiler: /usr/local/cuda/bin/nvcc - works
-- Detecting CUDA compiler ABI info
-- Detecting CUDA compiler ABI info - done
-- Detecting CUDA compile features
-- Detecting CUDA compile features - done
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Failed
-- Looking for pthread_create in pthreads
-- Looking for pthread_create in pthreads - not found
-- Looking for pthread_create in pthread
-- Looking for pthread_create in pthread - found
-- Found Threads: TRUE  
-- Found CUDA: /usr/local/cuda (found version "11.1") 
-- CUDA_NVCC_FLAGS:  -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_75,code=sm_75 -gencode arch=compute_75,code=compute_75
-- Using the single-header code from /home/zhaoyiluo/Forward/source/third_party/json/single_include/
-- Found TensorRT: /home/zhaoyiluo/libs/TensorRT-7.2.1.6/lib/libnvinfer.so;/home/zhaoyiluo/libs/TensorRT-7.2.1.6/lib/libnvinfer_plugin.so;/home/zhaoyiluo/libs/TensorRT-7.2.1.6/lib/libnvonnxparser.so;/home/zhaoyiluo/libs/TensorRT-7.2.1.6/lib/libnvparsers.so (found version "7.2.1") 
LINK_DIR = /home/zhaoyiluo/Forward/source/third_party/tensorflow/lib
-- Found PythonInterp: /home/local/anaconda3/bin/python (found version "3.7.3") 
-- Check if compiler accepts -pthread
-- Check if compiler accepts -pthread - yes
-- Configuring done
-- Generating done
-- Build files have been written to: /home/zhaoyiluo/Forward/build
```

It's easy to catch there are several lines related to `ABI info`. But I ignore its existence for several centuries. So what exactly is it?

```bash
# ...
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
# ...
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
# ...
-- Detecting CUDA compiler ABI info
-- Detecting CUDA compiler ABI info - done
# ...
```

## What is the Application Binary Interface (ABI)?

According to Wikipedia, an **application binary interface** (**ABI**) is an [interface](https://en.wikipedia.org/wiki/Interface_(computing)) between two binary program modules. Often, one of these modules is a [library](https://en.wikipedia.org/wiki/Library_(computing)) or [operating system](https://en.wikipedia.org/wiki/Operating_system) facility, and the other is a program that is being run by a user, in [computer software](https://en.wikipedia.org/wiki/Computer_software). It defines how data structures or computational routines are accessed in [machine code](https://en.wikipedia.org/wiki/Machine_code), which is a low-level, hardware-dependent format. In contrast, an [*API*](https://en.wikipedia.org/wiki/Application_programming_interface) defines this access in [source code](https://en.wikipedia.org/wiki/Source_code), which is a relatively high-level, hardware-independent, often [human-readable](https://en.wikipedia.org/wiki/Human-readable) format (as shown in Figure 1). A common aspect of an ABI is the [calling convention](https://en.wikipedia.org/wiki/Calling_convention), which determines how data is provided as input to, or read as output from, computational routines.

{{< image src="/images/abi/a-high-level-comparison-of-in-kernel-and-kernel-to-userspace-apis-and-abis.webp" caption="Figure 1: A high-level comparison of in-kernel and kernel-to-userspace APIs and ABIs." >}}

Your brain probably goes blank when you read the explanation for the first time, the same as me. This is because the problem is transparent for most engineers who only need to develop application programs or release a library in the form of source code. However, if you are the author who needs to release the library in the form of binary, you have to get familiar with it. When our program includes a library that is released in the form of binary, what we are actually using at the source-code level is the APIs provided in this library. After the program is compiled and linked, our program needs to communicate with the library, and the communication is conducted via the ABIs. In other words, the APIs define the order in which you pass arguments to a function, while the ABIs define the mechanics of how these arguments are passed (registers, stack, etc.). From this point of view, you can think of ABI as the low-level implementation of API (as shown in Figure 2). That explains why most of the time software engineers do not need to care about it.

{{< image src="/images/abi/the-linux-kernel-and-gnu-c-library-define-the-linux-api-after-compilation-the binaries-offer-an-abi-keeping-this-abi-stable-over-a-long-time-is-important-for-isvs.webp" caption="Figure 2: The Linux kernel and GNU C Library define the Linux API. After compilation, the binaries offer an ABI. Keeping this ABI stable over a long time is important for ISVs." >}}

To dive deeper, we need to know what details the ABI covers:
- a processor instruction set (with details like register file structure, stack organization, memory access types, ...)
- the sizes, layouts, and alignments of basic data types that the processor can directly access
- the calling convention, which controls how the arguments of functions are passed, and return values retrieved. For example, it controls:
  - whether all parameters are passed on the stack, or some are passed in registers;
  - which registers are used for which function parameters;
  - and whether the first function parameter passed on the stack is pushed first or last.
- how an application should make system calls to the operating system, and if the ABI specifies direct system calls rather than procedure calls to system call stubs, the system call numbers.
- and in the case of a complete operating system ABI, the binary format of object files, program libraries, and so on.

## How do the ABIs influence our programs?

Feeling much better? Yeah! But another problem comes.

When engineers are mentioning the ABIs in their daily work, what they are actually talking about? Based on my research, people are talking about the binary-compatible of the ABIs. The binary-compatible is another big topic and involves too many aspects. Here is an example for you to have a better understanding. Suppose the library which your program includes is updated at some times, but without changing the function signatures and usages, we call it source-compatible if you need to re-compile the program with the newer library in order to let the program run without the crash. On the contrary, if the program can run without crash by simply replacing the old version with the new version without re-compiling, we call the ABIs binary-compatible. To achieve and maintain the binary-compatible, it involves topics such as calling convention, name mangling, etc., which are above the discussion of this article. I would be willing to discuss these topics with you if you are also interested in them. 

## Conclusion

An oversimplified summary is that the APIs let you know "Here are all the functions you may call" while the ABIs let you know "This is how to call a function". Anyway, you won't need to explicitly provide the ABIs unless you are doing the design work in a very low-level system environment. Feel free to forget all the things I have mentioned above and just re-compile your program every time you update the libraries or anything related to your programs!

## References

1. [Wikipedia - Application binary interface](https://en.wikipedia.org/wiki/Application_binary_interface)
2. [What is an application binary interface (ABI)?](https://stackoverflow.com/questions/2171177/what-is-an-application-binary-interface-abi)