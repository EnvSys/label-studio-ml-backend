# syntax=docker/dockerfile:1

ARG MAINTAINER="Environment Systems Ltd. <developers@envsys.co.uk>"
ARG PYTHON_UNBUFFERED=1
ARG PYTHON_DONT_WRITE_BYTECODE=1
ARG DEBIAN_FRONTEND=noninteractive
ARG UV_INDEX_STRATEGY=unsafe-best-match
ARG UV_LINK_MODE=copy
ARG VENV_PATH=/venv
ARG TEST_ENV=false


FROM ubuntu:noble AS builder

ARG MAINTAINER
LABEL maintainer="${MAINTAINER}"

ARG PYTHON_UNBUFFERED
ARG PYTHON_DONT_WRITE_BYTECODE
ARG DEBIAN_FRONTEND
ARG UV_INDEX_STRATEGY
ARG UV_LINK_MODE
ARG VENV_PATH

ENV PYTHONUNBUFFERED=${PYTHON_UNBUFFERED}
ENV PYTHONDONTWRITEBYTECODE=${PYTHON_DONT_WRITE_BYTECODE}
ENV DEBIAN_FRONTEND=${DEBIAN_FRONTEND}
ENV UV_INDEX_STRATEGY=${UV_INDEX_STRATEGY}
ENV UV_LINK_MODE=${UV_LINK_MODE}
ENV VIRTUAL_ENV=${VENV_PATH}
ENV PATH="${VENV_PATH}/bin:$PATH"

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get update -y \
    && apt-get install --no-install-recommends -y \
        git \
        wget \
        curl \
        python3 \
        python3-pip \
        python3-dev \
        libffi-dev \
        gcc

RUN --mount=type=cache,target=/root/.cache/uv \
    python3 -m pip install --break-system-packages uv \
    && uv venv ${VENV_PATH}

# Force CPU-only torch before ultralytics can pull in CUDA variants
RUN --mount=type=cache,target=/root/.cache/uv \
    uv pip install \
        torch==2.6.0+cpu \
        torchvision==0.21.0+cpu \
        --index-url https://download.pytorch.org/whl/cpu

COPY requirements-base.txt requirements.txt requirements-test.txt ./

RUN --mount=type=cache,target=/root/.cache/uv \
    uv pip install -r requirements-base.txt \
    && uv pip install -r requirements.txt

ARG TEST_ENV
RUN --mount=type=cache,target=/root/.cache/uv \
    if [ "$TEST_ENV" = "true" ]; then \
        uv pip install -r requirements-test.txt; \
    fi

WORKDIR /app
COPY . /app/


FROM ubuntu:noble AS production

ARG MAINTAINER
LABEL maintainer="${MAINTAINER}"

ARG PYTHON_UNBUFFERED
ARG PYTHON_DONT_WRITE_BYTECODE
ARG DEBIAN_FRONTEND
ARG VENV_PATH

ENV PYTHONUNBUFFERED=${PYTHON_UNBUFFERED}
ENV PYTHONDONTWRITEBYTECODE=${PYTHON_DONT_WRITE_BYTECODE}
ENV DEBIAN_FRONTEND=${DEBIAN_FRONTEND}
ENV VIRTUAL_ENV=${VENV_PATH}
ENV PATH="${VENV_PATH}/bin:$PATH"
ENV PIP_CACHE_DIR=/.cache
ENV PORT=9090
ENV WORKERS=2
ENV THREADS=4
ENV PYTHONPATH=/app

# Runtime deps only - no compiler toolchain, no *-dev packages
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt/lists,sharing=locked \
    apt-get update -y \
    && apt-get install --no-install-recommends -y \
        ffmpeg \
        libsm6 \
        libxext6 \
        python3 \
        ca-certificates \
    && rm -rf /usr/share/doc/* \
    && rm -rf /usr/share/man/* \
    && rm -rf /usr/share/lintian/*

COPY --from=builder ${VENV_PATH} ${VENV_PATH}
COPY --from=builder /app /app

WORKDIR /app

EXPOSE 9090

CMD ["/app/start.sh"]
