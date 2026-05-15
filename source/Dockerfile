
# Defines build arguments for the versions of PyTorch, CUDA, and cuDNN to use
ARG PYTORCH_VERSION=2.12.0
ARG CUDA_VERSION=13.0
ARG CUDNN_VERSION=9

# This image is based on the latest official PyTorch image, because it already contains CUDA, CuDNN, and PyTorch
FROM pytorch/pytorch:${PYTORCH_VERSION}-cuda${CUDA_VERSION}-cudnn${CUDNN_VERSION}-runtime

# The PyTorch CUDA images are now built on Ubuntu 24.04's system Python (no more conda),
# which is PEP 668 "externally-managed" — a bare `pip install` is refused. PyTorch itself
# is preinstalled into that system Python, so we deliberately install ComfyUI's deps into
# the same environment (a venv would hide the CUDA-matched torch build). This makes every
# `pip install` below succeed without per-command flags.
ENV PIP_BREAK_SYSTEM_PACKAGES=1

# Installs Git, because ComfyUI and the ComfyUI Manager are installed by cloning their respective Git repositories
RUN apt-get update --assume-yes && \
    apt-get install --assume-yes \
        git \
        sudo \
        curl \
        ca-certificates \
        libgl1 \
        libglib2.0-0 && \
    rm -rf /var/cache/apt/archives /var/lib/apt/lists/*

# ── Custom ComfyUI frontend ───────────────────────────────────────────────────
# Build cora-clanker/ComfyUI_frontend into a comfyui-frontend-package wheel.
# Installed after ComfyUI requirements so it overrides the pinned upstream one.
ARG COMFYUI_FRONTEND_REF=main
ARG NODE_MAJOR=24
RUN curl -fsSL https://deb.nodesource.com/setup_${NODE_MAJOR}.x | bash - && \
    apt-get install -y --no-install-recommends nodejs && \
    rm -rf /var/lib/apt/lists/* && \
    npm install -g pnpm@10 && \
    pip install build

RUN git clone https://github.com/cora-clanker/ComfyUI_frontend /opt/ComfyUI_frontend && \
    cd /opt/ComfyUI_frontend && \
    git checkout ${COMFYUI_FRONTEND_REF} && \
    pnpm install --frozen-lockfile && \
    NODE_OPTIONS='--max-old-space-size=8192' pnpm build && \
    mkdir -p comfyui_frontend_package/comfyui_frontend_package/static && \
    cp -r dist/* comfyui_frontend_package/comfyui_frontend_package/static/ && \
    cd comfyui_frontend_package && \
    COMFYUI_FRONTEND_VERSION="$(node -p "require('/opt/ComfyUI_frontend/package.json').version")" \
        python -m build --wheel

# ── ComfyUI ───────────────────────────────────────────────────────────────────
# Pin to a specific ref. Pass --build-arg COMFYUI_REF=<tag-or-sha> to override.
ARG COMFYUI_REF=v0.15.0
RUN git clone https://github.com/comfyanonymous/ComfyUI /opt/comfyui && \
    cd /opt/comfyui && \
    git checkout ${COMFYUI_REF} && \
    pip install -r requirements.txt && \
    pip install --no-deps --force-reinstall /opt/ComfyUI_frontend/comfyui_frontend_package/dist/*.whl

# ── ComfyUI-Manager ───────────────────────────────────────────────────────────
# Kept at a top-level path (NOT under custom_nodes) on purpose: compose mounts a
# host volume over /opt/comfyui/custom_nodes, which would shadow a Manager baked
# in there. The entrypoint symlinks it into custom_nodes at runtime instead.
# The directory basename must stay exactly 'comfyui-manager' (lowercase) per
# project docs; otherwise Manager cannot identify itself for self-updates.
RUN git clone --depth 1 https://github.com/Comfy-Org/ComfyUI-Manager.git /opt/comfyui-manager && \
    pip install -r /opt/comfyui-manager/requirements.txt

# Sets the working directory to the ComfyUI directory
WORKDIR /opt/comfyui

# Exposes the default port of ComfyUI (this is not actually exposing the port to the host machine, but it is good practice to include it as metadata,
# so that the user knows which port to publish)
EXPOSE 8188

# Adds the startup script to the container; the startup script will create all necessary directories in the models and custom nodes volumes that were
# mounted to the container and symlink the ComfyUI Manager to the correct directory; it will also create a user with the same UID and GID as the user
# that started the container, so that the files created by the container are owned by the user that started the container and not the root user
ADD entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]
