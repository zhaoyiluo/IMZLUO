---
title: "DeepSpeed - Make distributed training easy, efficient, and effective"
subtitle: "A deep learning optimization library for extreme-scale model training"
date: 2021-07-07T00:42:55+08:00
lastmod: 2021-07-07T00:42:55+08:00
draft: false
authors: ["zhaoyiluo"]
description: "Introduction to DeepSpeed innovated by Microsoft."

tags: []
categories: ["Machine Learning"]
series: []

featuredImage: ""
featuredImagePreview: ""
---

## Overview

DeepSpeed is a deep learning optimization library announced by the DeepSpeed team at Microsoft, that makes distributed training easy, efficient, and effective. It is a further step of [ZeRO-Infinity](https://www.microsoft.com/en-us/research/blog/zero-infinity-and-deepspeed-unlocking-unprecedented-model-scale-for-deep-learning-training/) in training models with tens of trillions of parameters. In addition to the optimizations for scale ZeRO-Infinity provides, DeepSpeed also improves the speed, cost, and usability. This article will begin with the technologies used in the training step followed by the inference step to provide readers a better feeling though the latter part is the highlight point.

## Training

### Challenges

The main challenge with training for the large-scale models is to reduce training time without adding additional hardware, thus we need to figure out how to reduce latency and cost.

### Solutions

The DeepSpeed team introduces two compressed-training strategies to support fast and low-cost training while simultaneously delivering high accuracy. Besides, they also provide a new profiling tool to identify training performance bottlenecks.

#### Compressed training

The training exploits coarse-grained sparsity in Transformer layers via [Progressive Layer Dropping (PLD)](https://www.microsoft.com/en-us/research/publication/accelerating-training-of-transformer-based-language-models-with-progressive-layer-dropping/) to obtain reduced training cost. It's an algorithm that dynamically switches off Transformer layers during each iteration based on a progressive schedule that accounts for model sensitivity along both the temporal and depth dimensions.

![diagram](https://www.microsoft.com/en-us/research/uploads/prod/2021/05/DeepSpeed5_fig8_final-1024x392.jpg)

By adopting this strategy, DeepSpeed can reduce training time per sample by an average of 24 percent. On the other hand, when combined with the Pre-LN Transformer architecture, it facilitates training with more aggressive learning rates, achieving the same pretraining validity with 53 percent fewer training samples. In total, it yields a 2.8x faster convergence speed without hurting accuracy.

![chart, histogram](https://www.microsoft.com/en-us/research/uploads/prod/2021/05/DeepSpeed5_fig9_final.jpg)

#### 1-bit LAMB

Communication is a major bottleneck for training large models with hundreds or even thousands of GPUs. When it comes to commodity systems with limited-bandwidth TCP interconnect networks, this problem becomes more serious. Basically, there are two directions to address this issue,  communication volume reducing and number/frequency of communications decreasing. However the existing compression strategy 1-bit Adam cannot be directly applied to LAMB due to its unique adaptive layer-wise learning rates while adopting only one of the directions is not sufficient.

Thus, 1-bit LAMB, a novel communication-efficient algorithm is proposed by the team, that supports compressed communication in the context of large-batch optimization with adaptive layer-wise learning rates. The team also introduces new system implementations for compressed communication (for both 1-bit LAMB and 1-bit Adam) using the NVIDIA Collective Communications Library (NCCL) backend of PyTorch Distributed.

For BERT-Large pretraining tasks with batch sizes from 8K to 64K, the evaluations demonstrate that 1-bit LAMB with NCCL backend achieves 4.6x communication volume reduction, up to 2.8x training throughput speedup while retaining the same downstream task accuracy as LAMB under the same number of training samples.

![img](https://www.microsoft.com/en-us/research/uploads/prod/2021/05/DeepSpeed5_fig10_final-1024x604.jpg)

#### DeepSpeed Profiler performance tool

[DeepSpeed Flops Profiler](https://www.deepspeed.ai/tutorials/flops-profiler/) helps users easily measure both the model training/inference speed (latency, throughput) and efficiency (floating-point operations per second, also called FLOPS) of a large-scale deep learning model and its submodules, exposing inefficiencies in existing implementations.

- Identifying performance gaps

The first step is to identify whether there exist any performance inefficiencies that require further optimizations. Suppose we are using an NVIDIA GPU, we can get to read the theoretical peak performance from the website. But is the training or inference workload efficiently using those cores?  DeepSpeed Flops Profiler automatically calculates the number of parameters and tracks the execution time of each submodule and provides aggregated TFLOPS/s of a model, which in turn we can identify if a performance gap exists in the current model execution and whether further optimizations are needed.

- Identifying performance bottlenecks

The second step is to identify whether there exist any performance bottlenecks that require further optimizations. DeepSpeed Flops Profiler tracks detailed submodule runtime information and presents that fine-grained information, such as the fine-grained FLOPS/sec information for individual tensor operators or sublayers. Programmers can then make use of this information and decide whether multiple operators should be fused together to reduce the overhead from kernel invocation or a specific operator should be optimized through custom optimization.

## Inference

### Challenges 

Two of the main challenges with inference for large-scale models include latency and cost. This is because models are extremely computationally expensive and often too slow to respond in many practical scenarios. Besides, models with tens or hundreds of billions of parameters, trained with aggregated memory from multiple GPUs, simply become too large to fit on a single GPU’s device memory for inference. Though DeepSpeed supports training advanced large-scale models, there are three major limitations in existing inference solutions.

- Lack of support for multi-GPU inference to fit large models and meet latency requirements.
- Limited GPU kernel performance when running inference with small batch sizes.
- Difficulties in exploiting quantization, which includes both quantizing the model to reduce the model size and latency as well as supporting the high-performance inference of quantized models without specialized hardware.

### Solutions

The DeepSpeed team rolls out three main technologies for optimizing inference cost and latency, which show 1.9–4.4x latency speedups and 3.4–6.2x throughput gain and cost reduction when compared with existing work.

#### Inference-adapted parallelism

Multi-GPU parallelism is the first step to split the inference workload across multiple GPUs and it can reduce inference latency to meet the stringent latency requirements of production workloads. It introduces cross-GPU communication and thus reduces per-GPU computation granularity. To reduce parallelism overhead, users are responsible for tunning the parallelism degree and identify the optimal value for a given model architecture and hardware platform. 

#### Inference-optimized CUDA kernels

Inference with small batch sizes brings two major challenges. On the one hand, kernel invocation time and main memory latency become major bottlenecks due to limited work in each kernel. On the other hand, the default GeMM (General Matrix Multiplication) library is not well-tuned for extremely small batch sizes. Thus, two innovative optimizations are proposed to achieve significant latency reduction and throughput improvement.

- Deep fusion

Unlike other kernel-fusion techniques, DeepSpeed Inference can fuse multiple operators into a single kernel. That's to say, it can fuse element-wise operations, matrix multiplications, transpositions, and reductions all into a single kernel.

- Inference-customized GeMM

Small batch sizes result in skinny GeMM operations where the activations are a skinny matrix and the performance of GeMM is dominated by the time it takes to read the parameters from the main memory rather than the computation time itself. DeepSpeed Inference kernels are fine-tuned to maximize the memory bandwidth utilization for loading the parameters and can achieve up to 20 percent better performance than NVIDIA cuBLAS for inference workloads with batch sizes 1–10.

To incorporate the aforementioned optimizations, two sets of Transformer kernels in DeepSpeed Inference are provided.

- Generic Transformer replaces individual PyTorch operators within Transformer such as LayerNorm, Softmax, and bias-add with highly optimized DeepSpeed versions created using deep fusion.

- Specialized Transformer takes deep fusion one step further by creating fused schedules that not only fuse micro-operators within a PyTorch macro-operator (such as Softmax), but also fuse multiple macro-operators (such as Softmax and LayerNorm together with transpose op, and even GeMM).

![diagram](https://www.microsoft.com/en-us/research/uploads/prod/2021/05/Fig1_DeepSpeed5_Blog.jpg)

- Effective quantize-aware training

[DeepSpeed Quantization Toolkit](https://aka.ms/DS-QuantizationToolkit) are provided to further reduce the inference cost for large-scale models, which includes two parts. It allows users to easily quantize models that can efficiently execute with low-precision, such as 8-bit integer (INT8) instead of 32-bit floating-point (FP32), leading to both memory savings and latency reduction without hurting accuracy.

- Mixture of Quantization (MoQ)

Due to the influences brought by the small batch sizes, it is therefore sufficient to quantize just the parameters, while the activation can be computed and stored in FP16. MoQ uses the existing FP16 mixed-precision training pipeline in DeepSpeed by simply converting the FP32 parameter value to lower precision (INT4, INT8, etc.) and then storing them as FP16 parameters (FP16 datatype but with values mapping to lower precision) during the weight update.

With unquantized activations, flexible quantization schedule, and adaptive targets using second-order information, MoQ is much more robust in terms of accuracy when compared to conventional quantization approaches for the same compression ratio.

- High-performance INT8 inference kernels

These kernels are the extensions of Generic and Specialized Transformer kernels discussed aforementioned. They offer the same set of optimizations as the FP16 versions, but instead of loading FP16 parameters from main memory, they load INT8 parameters and convert them to FP16 on-the-fly before jumping into the inference computation, which reduces the data movement from main memory by half and results in 2x improvement in performance.

## Usage

An easy-to-use API is provided for users to use DeepSpeed. More information refer to [PythonRepo](https://pythonrepo.com/repo/microsoft-DeepSpeed-python-machine-learning).

![img](https://www.microsoft.com/en-us/research/uploads/prod/2021/05/DeepSpeed_fig2_5blog-1024x543.jpg)

```python
# DeepSpeed MoQ
import deepspeed
if args.MoQ:
    model, ... = deepspeed.initialize(
                args=args,
                model=model,
                config_params=MoQ_config,
                ...)
    # Run training with MoQ
    
# Initialize the model with DeepSpeed-Inference 
# using inference-kernels and configuring the parallelism setting 
import deepspeed.module_inject as module_inject
    
injection_policy={original_layer_implementation:
                    module_inject.replace_policy....}
model = deepspeed.init_inference(model, 
                                 mp_size=parallel_degree,
                                 mpu=mpu,
                                 checkpoint=[checkpoint_list],
                                 dtype=args.dtype,
                                 injection_policy=injection_policy,
                                )
```

## Result

The DeepSpeed team has conducted the following tests from different angles to verify its performance in latency reduction, cost reduction, throughput boosting, etc.

First, the team tested DeepSpeed on open-source models with publicly available checkpoints, like BERT, GPT-2, and GPT-Neo. The picture below presents the execution time of DeepSpeed Inference on a single NVIDIA V100 Tensor Core GPU with generic and specialized Transformer kernels respectively. The results show that the generic kernels provide 1.6–3x speedups to these models over the PyTorch baseline and the library can further reduce latency through the specialized kernels, achieving 1.9–4.4x speedups.

![img](https://www.microsoft.com/en-us/research/uploads/prod/2021/05/DeepSpeed5_fig3_blog_final.jpg)

Second, the team tested DeepSpeed through automated tensor-slicing model parallelism across multiple GPUs. The picture below presents the execution time of GPT-Neo (2.7B) for baseline and DeepSpeed Inference (DS-Inference) on one GPU and two GPUs with two-way model parallelism. On one side, DeepSpeed Inference speeds up the performance by 1.6x and 1.9x on a single GPU by employing the generic and specialized Transformer kernels, respectively. On the other side, it can further decrease the latency by using automated tensor slicing to partition the model across two GPUs. Altogether, the library achieves 2.3x speedup by combining the impact of the customized-inference kernels with the model-parallel inference execution.

![chart, bar chart](https://www.microsoft.com/en-us/research/uploads/prod/2021/05/DeepSpeed5_fig4_blog_final-1024x588.jpg)

Third, the team tested DeepSpeed by varying the number of Transformer layers and hidden sizes to match the existing network architecture: GPT-2 (1.5B), Turing-NLG (17B), and GPT-3 (175B). The picture below presents the inference throughput per GPU for the three models. It improves per-GPU throughput by 2–3.7x when using the same precision of FP16 as the baseline. It can get an improved throughput after enabling quantization.

![chart, bar chart](https://www.microsoft.com/en-us/research/uploads/prod/2021/05/DeepSpeed5_fig5_final-1024x541.jpg)

These three tests show the outstanding performance DeepSpeed brings. The team has also conducted other tests to further explore how excellent DeepSpeed is. More information refer to [the original article](https://www.microsoft.com/en-us/research/blog/deepspeed-accelerating-large-scale-model-inference-and-training-via-system-optimizations-and-compression/).

## References

1. [ZeRO-Infinity and DeepSpeed: Unlocking unprecedented model scale for deep learning training](https://www.microsoft.com/en-us/research/blog/zero-infinity-and-deepspeed-unlocking-unprecedented-model-scale-for-deep-learning-training/)
2. [Accelerating Training of Transformer-Based Language Models with Progressive Layer Dropping](https://www.microsoft.com/en-us/research/publication/accelerating-training-of-transformer-based-language-models-with-progressive-layer-dropping/)
3. [DeepSpeed Flops Profiler](https://www.deepspeed.ai/tutorials/flops-profiler/)
4. [Mixture-of-Quantization: A novel quantization approach for reducing model size with minimal accuracy impact](https://www.deepspeed.ai/news/2020/05/27/MoQ.html)
5. [Microsoft DeepSpeed Python Machine Learning](https://pythonrepo.com/repo/microsoft-DeepSpeed-python-machine-learning)
6. [DeepSpeed: Accelerating large-scale model inference and training via system optimizations and compression](https://www.microsoft.com/en-us/research/blog/deepspeed-accelerating-large-scale-model-inference-and-training-via-system-optimizations-and-compression/)