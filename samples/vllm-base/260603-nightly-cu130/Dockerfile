# syntax=docker/dockerfile:1.7
FROM debian:bookworm-slim

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8 \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    TORCH_CUDA_ARCH_LIST="12.0+PTX" \
    UV_PYTHON_INSTALL_DIR=/opt/python \
    UV_LINK_MODE=copy \
    CC=gcc \
    CXX=g++ \
    PATH="/opt/venv/bin:/root/.local/bin:${PATH}"

RUN apt-get update && apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        git \
        gcc \
        g++ \
        libc6-dev \
        libgomp1 \
        wget \
        gnupg \
    && curl -fsSL https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/3bf863cc.pub \
       | gpg --dearmor -o /usr/share/keyrings/cuda-archive-keyring.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/cuda-archive-keyring.gpg] \
       https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/ /" \
       > /etc/apt/sources.list.d/cuda.list \
    && apt-get update && apt-get install -y --no-install-recommends \
        cuda-nvcc-13-3 \
        cuda-cccl-13-3 \
        libcurand-dev-13-3 \
    && rm -rf /var/lib/apt/lists/*

ADD https://astral.sh/uv/install.sh /tmp/uv-install.sh
RUN sh /tmp/uv-install.sh && rm /tmp/uv-install.sh \
 && /root/.local/bin/uv python install 3.12 \
 && /root/.local/bin/uv venv --python 3.12 --seed /opt/venv

RUN uv pip install -U vllm --pre --extra-index-url https://wheels.vllm.ai/nightly/cu130 --extra-index-url https://download.pytorch.org/whl/cu130 --index-strategy unsafe-best-match

RUN mkdir -p /tmp/flashinfer

ADD https://github.com/flashinfer-ai/flashinfer/releases/download/v0.6.12/flashinfer_cubin-0.6.12-py3-none-any.whl /tmp/flashinfer/
ADD https://github.com/flashinfer-ai/flashinfer/releases/download/v0.6.12/flashinfer_python-0.6.12-py3-none-any.whl /tmp/flashinfer/
ADD https://github.com/flashinfer-ai/flashinfer/releases/download/v0.6.12/flashinfer_jit_cache-0.6.12+cu130-cp39-abi3-manylinux_2_28_x86_64.whl /tmp/flashinfer/

RUN uv pip install -v --python /opt/venv/bin/python --no-deps /tmp/flashinfer/*.whl

ENV CUDA_HOME=/usr/local/cuda-12.9
ENV PATH=${CUDA_HOME}/bin:${PATH}
ENV LD_LIBRARY_PATH=${CUDA_HOME}/lib64:${LD_LIBRARY_PATH}

ENV VLLM_USE_FLASHINFER_SAMPLER=1
ENV NVIDIA_TF32_OVERRIDE=1
ENV VLLM_NO_USAGE_STATS=1
ENV NCCL_P2P_DISABLE=0

EXPOSE 8000

