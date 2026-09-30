---
title: CUDA Profiling
description: CUDA 性能分析工具笔记：nvtx、nsys、ncu、cuda-gdb，以及 torch profiling 计时。
pubDatetime: 2026-09-30T00:00:00Z
tags: [CUDA, profiling]
---

## nvtx

```python
import nvtx

def check_safety(x_image):
    global safety_feature_extractor, safety_checker

    with nvtx.annotate('load model', color="red"):
        if safety_feature_extractor is None:
            print('start to load safety checker models ...')
            safety_feature_extractor = AutoFeatureExtractor.from_pretrained(safety_model_id)
            safety_checker = StableDiffusionSafetyChecker.from_pretrained(safety_model_id)

# color ref: https://github.com/NVIDIA/NVTX/blob/release-v3/python/nvtx/colors.py
@nvtx.annotate('censor', color="red") # 可选color: red, green, blue, darkgreen, orange
def censor_batch(x):
    x_samples_ddim_numpy = x.cpu().permute(0, 2, 3, 1).numpy()
    x_checked_image, has_nsfw_concept = check_safety(x_samples_ddim_numpy)
    x = torch.from_numpy(x_checked_image).permute(0, 3, 1, 2)

    return x
```

## nsys

install

```bash
apt update
apt install -y --no-install-recommends gnupg
echo "deb http://developer.download.nvidia.com/devtools/repos/ubuntu$(source /etc/lsb-release; echo "$DISTRIB_RELEASE" | tr -d .)/$(dpkg --print-architecture) /" | tee /etc/apt/sources.list.d/nvidia-devtools.list
apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64/7fa2af80.pub
apt update
apt install nsight-systems-cli

# https://docs.nvidia.com/nsight-systems/InstallationGuide/index.html
```

```bash
nsys profile --delay=60 -o timeline --trace-fork-before-exec=true --cuda-graph-trace=node --trace=cuda,nvtx,cudnn,cublas  --force-overwrite true --stats=true ./main

# --trace-fork-before-exec=true: 在处理多进程时需要
#  --delay=60: 延迟60秒后开始采样
```

## ncu

```bash
ncu --set full -s 2 -f -o out --kernel-name "myKernel" python test.py

# --kernel-name 滤掉nc开头的算子
--kernel-name "regex:^[^n][^c].*"

# -s 参数：跳过程序中的前个Kernel后再进行profiling

# -f -o out: 生成文件名out.nuc, -f表示强制覆盖之前存在的同名文件

# -w true: Waits for the process to complete.
```

ncu权限修复

```python
# 报错
==ERROR== ERR_NVGPUCTRPERM - The user does not have permission to access NVIDIA GPU Performance Counters on the target device 0. For instructions on enabling permissions and to get more information see https://developer.nvidia.com/ERR_NVGPUCTRPERM

# fix
sudo sh -c 'echo "options nvidia NVreg_RestrictProfilingToAdminUsers=0" > /etc/modprobe.d/nvidia.conf'
sudo update-initramfs -u
sudo reboot
```

## cuda-gdb

```python
# setup.py
from setuptools import setup
from torch.utils.cpp_extension import CppExtension, BuildExtension

setup(
    name='flow',
    ext_modules=[
        CppExtension(
            name='flow',  # This defines TORCH_EXTENSION_NAME
            sources=['layernorm_40.cu'],
            extra_compile_args={
                **'cxx': ['-g'],       # Debug flags for host code
                'nvcc': ['-G', '-lineinfo'],  # Debug flags for device code**
            }
        ),
    ],
    cmdclass={'build_ext': BuildExtension},
)
```

run cuda-gdb

```bash
# gdb
cuda-gdb --args python xxx.py

>> break xxx.cu:10
>> run

# memcheck
compute-sanitizer --tool memcheck python xxx.py
```

常用环境变量

```bash
export CUDA_VISIBLE_DEVICES=0,2
```

利用torch profilingi

```bash
# python

start_flash = torch.cuda.Event(enable_timing=True)
end_flash = torch.cuda.Event(enable_timing=True)
start_flash.record()
for _ in range(iter):
        flash_out = flash_fusion.linear(x, layer.ln.weight, layer.ln.bias)

 end_flash.record()
 torch.cuda.synchronize()
 print("avg flash linear sync time: ", start_flash.elapsed_time(end_flash) / iter, " ms.")

# c++
#include <ATen/cuda/CUDAEvent.h>
#include <torch/csrc/cuda/Event.h>
#include <ATen/cuda/Exceptions.h>
#include <ATen/cuda/tunable/StreamTimer.h>
#include <c10/cuda/CUDAStream.h>
  cudaEvent_t start_{};
  cudaEvent_t end_{};
  cudaEventCreate(&start_);
  cudaEventCreate(&end_);
  cudaEventSynchronize(start_);
  cudaEventRecord(start_, at::cuda::getCurrentCUDAStream());

  # do infer

  AT_CUDA_CHECK(cudaEventRecord(end_, at::cuda::getCurrentCUDAStream()));
  AT_CUDA_CHECK(cudaEventSynchronize(end_));

  auto time = std::numeric_limits<float>::quiet_NaN();
  // time is in ms with a resolution of 1 us
  AT_CUDA_CHECK(cudaEventElapsedTime(&time, start_, end_));

  std::cout << "Torch CUDA time: " << time << " ms" << std::endl;

```

## cpu

https://github.com/brendangregg/FlameGraph

**flamegraph**

```shell
# 1. 安装py-spy
pip install py-spy

# 2. 直接生成火焰图（结果更精准）
py-spy record -o flamegraph.svg -- pytest test_generate.py
```

## torch profiling

![torch profiling](/images/torch_profiling.png)
