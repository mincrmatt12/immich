ARG DEVICE=cpu

FROM python:3.11-bookworm@sha256:f89d36dbb4728313572f88877b8be7d11fd03bea964cdf0a6b0f61edfcde3709 AS builder-cpu

FROM builder-cpu AS builder-openvino

FROM builder-cpu AS builder-cuda

FROM builder-cpu AS builder-armnn

ENV ARMNN_PATH=/opt/armnn
COPY ann /opt/ann
RUN mkdir /opt/armnn && \
    curl -SL "https://github.com/ARM-software/armnn/releases/download/v24.05/ArmNN-linux-aarch64.tar.gz" | tar -zx -C /opt/armnn && \
    cd /opt/ann && \
    sh build.sh

FROM builder-${DEVICE} AS builder

ARG DEVICE
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=true \
    VIRTUAL_ENV="/opt/venv" \
    PATH="/opt/venv/bin:${PATH}"

RUN apt-get update && apt-get install -y --no-install-recommends g++

RUN pip install --upgrade pip && pip install poetry
RUN poetry config installer.max-workers 10 && \
    poetry config virtualenvs.create false
RUN python3 -m venv /opt/venv

COPY poetry.lock pyproject.toml ./
RUN poetry install --sync --no-interaction --no-ansi --no-root --with ${DEVICE} --without dev

FROM python:3.11-slim-bookworm@sha256:7f49f147e57a65a5ca731203ed350ac5c88fa54aeb942924dd7057fe34a18e79 AS prod-cpu

FROM prod-cpu AS prod-openvino

COPY scripts/configure-apt.sh ./
RUN ./configure-apt.sh && \
    apt-get update && \
    apt-get install -t unstable --no-install-recommends -yqq intel-opencl-icd && \
    rm configure-apt.sh

FROM nvidia/cuda:12.3.2-cudnn9-runtime-ubuntu22.04@sha256:fa44193567d1908f7ca1f3abf8623ce9c63bc8cba7bcfdb32702eb04d326f7a8 AS prod-cuda

COPY --from=builder-cuda /usr/local/bin/python3 /usr/local/bin/python3
COPY --from=builder-cuda /usr/local/lib/python3.11 /usr/local/lib/python3.11
COPY --from=builder-cuda /usr/local/lib/libpython3.11.so /usr/local/lib/libpython3.11.so

FROM prod-cpu AS prod-armnn

ENV LD_LIBRARY_PATH=/opt/armnn

RUN apt-get update && apt-get install -y --no-install-recommends ocl-icd-libopencl1 mesa-opencl-icd libgomp1 && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir --parents /etc/OpenCL/vendors && \
    echo "/usr/lib/libmali.so" > /etc/OpenCL/vendors/mali.icd && \
    mkdir /opt/armnn

COPY --from=builder-armnn \
    /opt/armnn/libarmnn.so.?? \
    /opt/armnn/libarmnnOnnxParser.so.?? \
    /opt/armnn/libarmnnDeserializer.so.?? \
    /opt/armnn/libarmnnTfLiteParser.so.?? \
    /opt/armnn/libprotobuf.so.?.??.?.? \
    /opt/ann/libann.s[o] \
    /opt/ann/build.sh \
    /opt/armnn/

FROM prod-${DEVICE} AS prod
ARG DEVICE

RUN apt-get update && \
    apt-get install -y --no-install-recommends tini $(if ! [ "$DEVICE" = "openvino" ]; then echo "libmimalloc2.0"; fi) && \
    apt-get autoremove -yqq && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
ENV TRANSFORMERS_CACHE=/cache \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH" \
    PYTHONPATH=/usr/src \
    DEVICE=${DEVICE}

# prevent core dumps
RUN echo "hard core 0" >> /etc/security/limits.conf && \
    echo "fs.suid_dumpable 0" >> /etc/sysctl.conf && \
    echo 'ulimit -S -c 0 > /dev/null 2>&1' >> /etc/profile

COPY --from=builder /opt/venv /opt/venv
COPY ann/ann.py /usr/src/ann/ann.py
COPY start.sh log_conf.json ./
COPY app .
ENTRYPOINT ["tini", "--"]
CMD ["./start.sh"]

HEALTHCHECK CMD python3 healthcheck.py
