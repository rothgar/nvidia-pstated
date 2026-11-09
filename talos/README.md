# nvidia-pstated Talos system extension

Runs `nvidia-pstated` as a Talos extension service (`ext-nvidia-pstated`).

## Requirements

Install alongside the official Sidero Labs extensions:

- `siderolabs/glibc`
- `siderolabs/nvidia-open-gpu-kernel-modules-*` (or `nonfree-kmod-nvidia-*`)
- `siderolabs/nvidia-container-toolkit-*` (provides `libnvidia-ml.so.1` and `libnvidia-api.so.1`)

and load the `nvidia` kernel modules as described in the
[Talos NVIDIA GPU guide](https://docs.siderolabs.com/talos/latest/configure-your-talos-cluster/hardware-and-drivers/nvidia-gpu-proprietary).

## Installation

Add the image to your Image Factory schematic / imager as a custom extension, or with `talosctl` installer images:

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/glibc
      - siderolabs/nvidia-open-gpu-kernel-modules-production
      - siderolabs/nvidia-container-toolkit-production
```

```sh
docker run --rm -t -v /dev:/dev --privileged -v $PWD/_out:/out \
  ghcr.io/siderolabs/imager:<talos-version> installer \
  --system-extension-image ghcr.io/siderolabs/glibc:<version> \
  --system-extension-image ghcr.io/siderolabs/nvidia-open-gpu-kernel-modules-production:<version> \
  --system-extension-image ghcr.io/siderolabs/nvidia-container-toolkit-production:<version> \
  --system-extension-image ghcr.io/rothgar/nvidia-pstated:<version>
```

## Verify

```sh
talosctl get extensions
talosctl services ext-nvidia-pstated
talosctl logs ext-nvidia-pstated
```

## Building

```sh
docker buildx build --platform linux/amd64,linux/arm64 \
  --build-arg VERSION=$(git describe --tags) \
  -f talos/Dockerfile -t ghcr.io/rothgar/nvidia-pstated:$(git describe --tags) --push .
```

The daemon runs with its default options. The extension service runs the
binary directly (no shell), so the fan scripts (`--enable-fan-script` /
`--disable-fan-script`) are not supported.
