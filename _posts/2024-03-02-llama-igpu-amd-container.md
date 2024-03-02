---
title: Local AI assistant in vim - AMD iGPU and ROCm in a slightly optimized container
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

# Update: Local AI assistant in vim with codellama with iGPU accelerated inference - container image details

My previous containerization refinement of llama for the local AI assistant
stopped after fulfilling the following requirements:

- It works
- It uses ROCm for GPU accelerated inference
- It allows me to specify a ROCm version at build time
- It allows me to specify a llama-cpp-python version at build time

With these requirements, I created a single stage `Dockerfile`, resulting in a
quite large container image, and stopped right there. It worked, after all.

```bash
podman inspect -f "{{ .Size }}" ghcr.io/gierdo/dotfiles/llama-cpp-python-server-rocm:0.2.44-6.0.2-11.0.2 | numfmt --to=si
23G
```

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

ARG ROCM_VERSION=6.0.2

RUN mkdir --parents --mode=0755 /etc/apt/keyrings \
  && wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
  gpg --dearmor | tee /etc/apt/keyrings/rocm.gpg > /dev/null \
  && echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/${ROCM_VERSION} jammy main" > /etc/apt/sources.list.d/rocm.list \
  && echo 'Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600' > /etc/apt/preferences.d/rocm-pin-600 \
  && apt-get update \
  && apt-get install -y --no-install-recommends rocm-hip-sdk

ARG OVERRIDE_GFX_VERSION=11.0.2
ENV HSA_OVERRIDE_GFX_VERSION=${OVERRIDE_GFX_VERSION}

ARG LLAMA_CPP_VERSION=0.2.44
RUN CC=hipcc CXX=hipcc CMAKE_ARGS="-DLLAMA_HIPBLAS=on" pip install llama-cpp-python[server]==${LLAMA_CPP_VERSION}

ENV HOST=0.0.0.0
ENV PORT=8000
EXPOSE 8000

CMD ["bash", "-c", "python3 -m llama_cpp.server --n_gpu_layers 15 --host ${HOST} --port ${PORT}"]
```

## The problem

As you can see, I simply dumped the entire `rocm-hip-sdk` into the image and
that's that. That means, of course, that the _build time_ dependencies, so
everything needed to _compile_ software to use ROCm is part of the image, next
to everything that is required to _execute_ compiled software. The sole purpose
of the final container is to _execute_ the compiled llama-cpp-python server, so
all the _build time_ dependencies are just wasted space.

## The solution

Container builds can be split into _stages_, with only the last stage being
baked into the final image. Those _stages_ can depend upon previous stages or
share content, _copied_ from one stage into the other.

With that approach, a separate stage can be set up with all the _build time_
dependencies, used to compile the desired artifact, which is then _copied_ into
the final image which only contains the _run time_ dependencies.

As docker/podman caches and reuses identical layers, you can build your
`Dockerfile` in a way that allows the build process to reuse as much as
possible, while still keeping the final image small, in order to speed up your
local build process. You don't want to wait for apt to download _14Gb_ of
libraries just because you changed the version of a python package to be
installed, after all.

```bash
podman inspect -f "{{ .Size }}" ghcr.io/gierdo/dotfiles/llama-cpp-python-server-rocm:0.2.54-6.0.2-11.0.2 | numfmt --to=si
15G
```

```Dockerfile
FROM ubuntu:jammy AS rocm_base

RUN apt-get update && apt-get install -y --no-install-recommends \
  ca-certificates \
  wget \
  gpg \
  pkg-config \
  && rm -rf /var/lib/{apt,dpkg,cache,log}/

ARG ROCM_VERSION=6.0.2

RUN mkdir --parents --mode=0755 /etc/apt/keyrings \
  && wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | \
  gpg --dearmor | tee /etc/apt/keyrings/rocm.gpg > /dev/null \
  && echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/rocm.gpg] https://repo.radeon.com/rocm/apt/${ROCM_VERSION} jammy main" > /etc/apt/sources.list.d/rocm.list \
  && echo 'Package: *\nPin: release o=repo.radeon.com\nPin-Priority: 600' > /etc/apt/preferences.d/rocm-pin-600

# Allow caching the runtime base even if llama_cpp is rebuilt
FROM rocm_base AS runtime_base

RUN apt-get update && apt-get install -y --no-install-recommends \
  python3-pip \
  rocm-hip-libraries \
  && rm -rf /var/lib/{apt,dpkg,cache,log}/


FROM runtime_base AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
  ninja-build \
  libopenblas-dev \
  git \
  cmake \
  pkg-config \
  build-essential \
  rocm-hip-sdk \
  && rm -rf /var/lib/{apt,dpkg,cache,log}/

ARG OVERRIDE_GFX_VERSION=11.0.2
ENV HSA_OVERRIDE_GFX_VERSION=${OVERRIDE_GFX_VERSION}
ARG LLAMA_CPP_VERSION=0.2.54

RUN CC=hipcc CXX=hipcc CMAKE_ARGS="-DLLAMA_HIPBLAS=on" pip install --user llama-cpp-python[server]==${LLAMA_CPP_VERSION}

FROM runtime_base AS final
LABEL maintainer="Dominikus Gierlach <dominik.gierlach@gmail.com>"

ARG OVERRIDE_GFX_VERSION=11.0.2
ENV HSA_OVERRIDE_GFX_VERSION=${OVERRIDE_GFX_VERSION}

COPY --from=builder /root/.local /root/.local

ENV HOST=0.0.0.0
ENV PORT=8000
EXPOSE 8000

CMD ["bash", "-c", "python3 -m llama_cpp.server --n_gpu_layers 15 --host ${HOST} --port ${PORT}"]
```

The resulting image is still large and could be slimmed down. It now contains
the full collection of ROCm libraries and runtime dependencies. I could
probably walk through them and just identify the stuff that is strictly
required. But losing _8Gb_ of image size, basically for free, is a better use
of my time, I think.

## tl,dr:

In summary, utilizing a multi-stage build approach in your Dockerfile allows
you to separate your application's build dependencies from its final runtime
environment, reducing the overall size of your final container image. This
technique is also beneficial when working with large runtime environments, such
as ROCm, where runtime dependencies still significantly increase the final
image size. Organizing your container image layers ensures layer reuse and
caching during the build process, leading to further efficiency and
optimization of the build process.
