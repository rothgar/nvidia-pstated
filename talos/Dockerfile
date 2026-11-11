# Talos Linux system extension for nvidia-pstated.
#
# Build (from the repository root):
#   docker buildx build --platform linux/amd64,linux/arm64 \
#     --build-arg VERSION=v1.0.9 \
#     -f talos/Dockerfile -t ghcr.io/rothgar/nvidia-pstated:v1.0.9 --push .

FROM ubuntu:24.04 AS build

ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl gcc libc6-dev libnvidia-ml-dev \
 && rm -rf /var/lib/apt/lists/*

# Same NVAPI SDK release and checksum pinned in CMakeLists.txt
ARG NVAPI_URL=https://download.nvidia.com/XFree86/nvapi-open-source-sdk/R555-OpenSource.tar
ARG NVAPI_SHA256=71339c274a6a633f19b6bd358c7f3045063c6bc106b7dc488aaa7360a6d2b9d7
RUN curl -fsSL -o /tmp/nvapi.tar "${NVAPI_URL}" \
 && echo "${NVAPI_SHA256}  /tmp/nvapi.tar" | sha256sum -c - \
 && mkdir -p /opt/nvapi \
 && tar -xf /tmp/nvapi.tar -C /opt/nvapi \
 && ln -s "$(dirname "$(find /opt/nvapi -name nvapi.h -print -quit)")" /opt/nvapi-include

WORKDIR /src
COPY src/ src/

# libnvidia-ml.so.1 is linked dynamically, libnvidia-api.so.1 is dlopen()ed.
# Both are provided on the Talos host by the nvidia-container-toolkit extension
# in /usr/local/lib, which is in the default search path of the Talos glibc extension.
RUN gcc -O2 -Wall -o /nvidia-pstated \
      -isystem /opt/nvapi-include \
      src/main.c src/nvapi.c src/utils.c \
      -lnvidia-ml -ldl \
 && strip /nvidia-pstated

ARG VERSION=dev
COPY talos/manifest.yaml /manifest.yaml
RUN sed -i "s/@VERSION@/${VERSION#v}/" /manifest.yaml

FROM scratch
COPY --from=build /manifest.yaml /manifest.yaml
COPY talos/nvidia-pstated.yaml /rootfs/usr/local/etc/containers/nvidia-pstated.yaml
COPY --from=build /nvidia-pstated /rootfs/usr/local/lib/containers/nvidia-pstated/usr/bin/nvidia-pstated
COPY LICENSE /rootfs/usr/local/lib/containers/nvidia-pstated/usr/share/licenses/nvidia-pstated/LICENSE
