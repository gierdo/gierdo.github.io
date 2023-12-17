---
title: Local AI assistant in vim - Update with AMD iGPU and ROCm
categories:
  - programming
  - linux
tags:
  - vim
  - openai
  - ai
  - containers
  - llama
  - ROCm
  - AMD
---

# Update: Local AI assistant in vim with codellama - now with iGPU accelerated inference

You might have read my initial post about [using a local codellama for a vim
coding assistant]({% link _posts/2023-11-24-llama-vim.md %}). The initial
version used pure CPU, OpenBLAS optimized inference. Which is not ideal, to say
the least.


> The proposed solution only uses CPU vectorization and should (TM) be portable.
> I don't have a machine with a GPU capable of doing shiny Cuda/hipBLAS/ROCm
> acceleration. This means that the performance is OK(ish) for one single
> parallel user. Which is exactly what the setup was intended for from the start.
> Be aware though that the running model needs 9 GiB of memory while idling and
> will make use of pretty much all of your CPU while working. That's also why I
> set it up to only start when it's really used.
>
> ...
>
> If your setup has a ROCm supported AMD graphics card or a CUDA supported Nvidia
> card, you will probably want to spend some time in getting `llama.cpp` to make
> use of it for a significant performance increase, at the price of removed
> portability and added maintenance.

Long story short, I dug a little deeper and tried to make GPU accelerated
inference work on my t14s with its integrated AMD graphics card, an
_AMD Ryzen 7 PRO 7840U w/ Radeon 780M Graphics_.

The issue is that iGPUs are not _officially_ supported by _ROCm_, which is
AMD's software platform for GPU computing.

Long story short, it can work with a bit of bending and glue, which
[others](https://www.reddit.com/r/LocalLLaMA/comments/170tghx/guide_installing_rocmhip_for_llamacpp_on_linux/)
and [probably many
more](https://medium.com/@rafaelmanzanom/ditching-cuda-for-amd-rocm-for-more-accessible-llm-inference-ryzen-apus-edition-92c3649f8f7d)
also have figured out. Information is scattered around, but most of the time
the described setups are pretty unrepeatable.

I tried to make a kind of compromise between the specific hardware dependent
setup and a generic solution, using a fully containerized setup with hardware
specific overrides.

## The Problem

LLM inference, which is the process of a language model generating a response
to a prompt based on the input it has been given, is a task that favours
parallel execution on GPUs over the execution on general purpose CPUs. Which is
one of the reasons why all chip manufacturers, led by Nvidia, are in a hard
buzzword bingo tournament over AI accelerator chips.

What's pretty challenging there is that you can't just compile a program and
throw it on any GPU you want. The programs have to be compiled rather
specifically for individual GPU architectures.

For GPU accelerated graphics, there are standardized abstraction layers, which
are manufacturer independent and make programs somewhat portable. E.g.
[OpenGL](https://en.wikipedia.org/wiki/OpenGL),
[Vulkan](https://en.wikipedia.org/wiki/Vulkan) or (in the windows world)
[Direct3D](https://en.wikipedia.org/wiki/Direct3D). While there are abstraction
layers for GPU accelerated computing that make this task easier than doing
everything GPU specific, there is no global standard (yet). Nvidia established
their [CUDA](https://en.wikipedia.org/wiki/CUDA) platform with great success,
reaching a state of an almost-monopoly, surpassing AMD and Intel with their
offerings by a [large
margin](https://www.visualcapitalist.com/nvidia-vs-amd-vs-intel-comparing-ai-chip-sales/)
of (for Q2 2023) $10.3B in sales for Nvidia vs $1.3B (AMD) and $4.0B (Intel).

This is probably also influenced by the fact that it is relatively easy to play
around with- and develop software for CUDA. Pretty much every NVIDIA graphics
card, including laptop graphics or ordinary gaming hardware, is supported by
CUDA.

Which is _absolutely not_ the case for AMD's ROCm, where only specific hardware
is officially supported, and more recent integrated graphics adapters, while
pretty performant and good value for graphics stuff, are completely
unsupported. Which means that it is hard and annoying for developers. And, as
Steve Balmer once said, it's all about
[developers](https://www.youtube.com/watch?v=Vhh_GeBPOhs).

So, here I am, trying to run a codellama locally, on my laptop, with an AMD CPU
and GPU, trying to benefit from nice GPU magic.

## The solution

Here is what I hoped for when I tried to make it work, specifically in a
container environment:

- AMD ROCm works with the open source `amdgpu` driver, no need to fuddle around
  on the kernel layer and keep kernel stuff, host stuff and container content
  in sync, loosing the nice decoupling we get with containers
- The officially non-existent support for my GPU is more of a "We didn't really
  test it, a few obscure things might not work, but technically, it works" kind
  of situation. This was kind of realistic. AMD doesn't reinvent all their GPUs
  from scratch, they don't speak fundamentally different languages.

Both of these hopes were fulfilled, so I will draft the solution here:

### Container image with ROCm SDK, llama.cpp with ROCm support

ROCm 6.0 is available for ubuntu jammy, so I based my image on that. I followed
the [ROCm
documentation](https://rocm.docs.amd.com/projects/install-on-linux/en/latest/how-to/native-install/ubuntu.html)
and the [llama-cpp-python
documentation](https://github.com/abetlen/llama-cpp-python/pkgs/container/llama-cpp-python#hipblas)
to install all dependencies to build and run llama.cpp with ROCm support, and
not too much else.

The `OVERRIDE_GFX_VERSION` attribute is _hardware specific_ and is needed for
my graphics card, a `Radeon 780M`, which announces itself as having
`GFX_VERSION 11.0.3`, which does not have matching library support. Luckily,
overriding the value to the closest available value worked.

```bash
$ ls /opt/rocm/lib/rocblas/library/
.
.
.
TensileLibrary_lazy_gfx1030.dat
TensileLibrary_lazy_gfx1100.dat
TensileLibrary_lazy_gfx1101.dat
TensileLibrary_lazy_gfx1102.dat
TensileLibrary_lazy_gfx900.dat
TensileLibrary_lazy_gfx906.dat
TensileLibrary_lazy_gfx908.dat
TensileLibrary_lazy_gfx90a.dat
TensileLibrary_lazy_gfx940.dat
TensileLibrary_lazy_gfx941.dat
TensileLibrary_lazy_gfx942.dat
```

Here's the image definition:

```Dockerfile
FROM ubuntu:jammy

RUN apt-get update && apt-get install -y --no-install-recommends \
  ninja-build \
  libopenblas-dev \
  python3-pip \
  wget \
  gpg \
  git \
  cmake \
  pkg-config \
  build-essential

RUN mkdir --parents --mode=0755 /etc/apt/keyrings \
  && wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
  gpg --dearmor | tee /etc/apt/keyrings/rocm.gpg > /dev/null \
  && echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/6.0 jammy main" > /etc/apt/sources.list.d/rocm.list \
  && echo 'Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600' > /etc/apt/preferences.d/rocm-pin-600 \
  && apt-get update \
  && apt-get install -y --no-install-recommends rocm-hip-sdk

ARG VERSION=0.2.20

ARG OVERRIDE_GFX_VERSION=11.0.2
ENV HSA_OVERRIDE_GFX_VERSION=${OVERRIDE_GFX_VERSION}

RUN  CC=hipcc CXX=hipcc CMAKE_ARGS="-DLLAMA_HIPBLAS=on" pip install llama-cpp-python[server]==${VERSION}

ENV HOST=0.0.0.0
ENV PORT=8000
EXPOSE 8000

CMD ["bash", "-c", "python3 -m llama_cpp.server --n_gpu_layers 10 --host ${HOST} --port ${PORT}"]
```

This image is built and pushed to
`ghcr.io/gierdo/dotfiles/llama-cpp-python-server-rocm:0.2.20-11.0.2`, the first
version attribute for the version of python-llama-cpp, the second for the
GFX_VERSION.

### Container system integration

The running container requires special access to the graphics card, so we have
to make sure that

- The device is mounted in the container
- Access permissions allow the device to be used in the container

```bash
$ podman run --annotation run.oci.keep_original_groups=1 -v ${HOME}/.local/lib/llama/models:/models --device /dev/kfd --device /dev/dri --rm -it ghcr.io/gierdo/dotfiles/llama-cpp-python-server-rocm:0.2.20-11.0.2
```

Using the systemd podman integration, this leads to the following systemd user
unit:

```text
[Unit]
Description=Local llama.cpp api server

[Container]
Image=ghcr.io/gierdo/dotfiles/llama-cpp-python-server-rocm:0.2.20-11.0.2
AddDevice=/dev/dri
AddDevice=/dev/kfd
Annotation="run.oci.keep_original_groups=1"
SecurityLabelDisable=true
Volume=${HOME}/.local/lib/llama/models:/models
PublishPort=9741:8000
Environment=MODEL=/models/codellama-13b-instruct.Q4_K_M.gguf
Environment=HSA_OVERRIDE_GFX_VERSION=11.0.2
Exec= sh -c "python3 -m llama_cpp.server --n_gpu_layers 10 --host 0.0.0.0 --port 8000"
```

The value of _10_ layers to be handed over to the GPU was chosen iteratively
for my specific system. My current configuration has 3994M of _VRAM_, handing
over 15 layers leads to 95% of VRAM being consumed by it at idle, so it runs
out of memory during inference.

### Initial results

Besides the different container, the rest of my setup is basically as described
in my [previous article]({% link _posts/2023-11-24-llama-vim.md %}).

While running, we see that the graphics pipe is at 100% and VRAM is
pretty much full.

The performance of the model is very usable, but not fundamentally different
from the pure CPU model on smaller prompts. This appears to be different for
larger context. I'll benchmark it against a pure CPU model in another article,
but I don't expect overwhelming results. I expect the impact to be a bit larger
with more VRAM and more handed-over layers.

```text
                Graphics pipe 100.00%
──────────────────────────────────────
                 Event Engine   0.00%

  Vertex Grouper + Tesselator   0.00%

            Texture Addresser   0.00%
                Texture Cache   0.00%

                Shader Export   0.00%
  Sequencer Instruction Cache   0.00%
          Shader Interpolator   0.00%
       Shader Memory Exchange   0.00%

               Scan Converter   0.00%
           Primitive Assembly   0.00%

                  Depth Block   0.00%
                  Color Block   0.00%
               Clip Rectangle 100.00%

           3439M / 3994M VRAM  86.12%
            293M / 13866M GTT   2.11%
   0.80G / 0.80G Memory Clock 100.00%
   2.70G / 2.70G Shader Clock 100.00%
```

And it works!

I marked the table above to put it into context and asked codellama about it.


```text
Do you understand this table?

--------

  Yes, I understand the table. It appears to be a breakdown of the performance
  metrics for a computer graphics rendering system, showing the percentage of
  time spent in each pipeline stage and the percentage of VRAM and GTT
  utilization. The table also includes information about memory clock and
  shader clock speeds.

--------
```

## tl,dr:

- llm stuff works well on GPUs
- Computing on GPUs is weird, manufacturer and hardware specific
- AMD graphics on recent laptops are not officially supported by AMD for
  non-graphics stuff
- AMD laptop graphics can work for llm stuff, even if not officially supported
- The performance impact is not overwhelming, but noticeable
