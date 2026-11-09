# nvidia-pstated Talos system extension

Runs `nvidia-pstated` on [Talos Linux](https://www.talos.dev) as an extension
service (`ext-nvidia-pstated`). It automatically switches NVIDIA GPUs into a low
performance state (P8) when idle and back to high (P0) under load. This saves a
lot of idle power on cards that otherwise sit in P0 once a CUDA context exists.

Image: `ghcr.io/rothgar/nvidia-pstated` (`linux/amd64`, `linux/arm64`)

## Requirements

This extension ships only the `nvidia-pstated` binary. The runtime libraries
come from the official Sidero Labs extensions, which must be installed in the
same image:

| Extension | Provides |
|-----------|----------|
| `siderolabs/glibc` | glibc and the dynamic loader |
| `siderolabs/nvidia-open-gpu-kernel-modules-production` (or `nonfree-kmod-nvidia-production`) | NVIDIA kernel modules |
| `siderolabs/nvidia-container-toolkit-production` | `libnvidia-ml.so.1` and `libnvidia-api.so.1` |

You can use the `-lts` variants instead of `-production`. The kernel module and
container toolkit extensions must be on the same driver branch.

## Building a custom Talos image with Imager

Image Factory can only include official extensions, so use
[Imager](https://docs.siderolabs.com/talos/latest/platform-specific-installations/boot-assets)
to build boot media and installer images that include this one.

### 1. Find matching extension versions

Each Talos release pins the official extension versions it was built with.
Look them up for the Talos version you want:

```sh
TALOS_VERSION=v1.14.5

crane export ghcr.io/siderolabs/extensions:${TALOS_VERSION} - \
  | tar x -O image-digests \
  | grep -E '/glibc:|nvidia-open-gpu-kernel-modules-production|nvidia-container-toolkit-production'
```

Example output for `v1.14.5`:

```text
ghcr.io/siderolabs/glibc:2.44@sha256:eb4460f4...
ghcr.io/siderolabs/nvidia-container-toolkit-production:595.91.07-v1.19.1@sha256:04324c16...
ghcr.io/siderolabs/nvidia-open-gpu-kernel-modules-production:595.91.07-v1.14.5@sha256:09e721ce...
```

Pass the full references, including the digests, to Imager.

### 2. Build an installer image (for installs and upgrades)

```sh
TALOS_VERSION=v1.14.5
PSTATED_VERSION=latest  # or pin a tag from https://github.com/rothgar/nvidia-pstated/pkgs/container/nvidia-pstated

docker run --rm -t \
  -v "$PWD/_out:/out" -v /dev:/dev --privileged \
  ghcr.io/siderolabs/imager:${TALOS_VERSION} installer \
  --arch amd64 \
  --system-extension-image ghcr.io/siderolabs/glibc:2.44@sha256:<digest> \
  --system-extension-image ghcr.io/siderolabs/nvidia-open-gpu-kernel-modules-production:595.91.07-${TALOS_VERSION}@sha256:<digest> \
  --system-extension-image ghcr.io/siderolabs/nvidia-container-toolkit-production:595.91.07-v1.19.1@sha256:<digest> \
  --system-extension-image ghcr.io/rothgar/nvidia-pstated:${PSTATED_VERSION}
```

This writes `_out/installer-amd64.tar`. Push it to a registry your nodes can
pull from:

```sh
crane push _out/installer-amd64.tar ghcr.io/<you>/talos-installer-nvidia:${TALOS_VERSION}
```

### 3. Build boot media (optional)

To install new machines, use the same `--system-extension-image` flags and
replace `installer` with a different output profile:

- `iso`: bootable ISO (`_out/metal-amd64.iso`)
- `metal`: raw disk image (`_out/metal-amd64.raw.zst`)
- `aws`, `gcp`, `azure`, ...: cloud images

For example:

```sh
docker run --rm -t \
  -v "$PWD/_out:/out" -v /dev:/dev --privileged \
  ghcr.io/siderolabs/imager:${TALOS_VERSION} iso \
  --arch amd64 \
  --system-extension-image ... # same four images as above
```

Use `--arch arm64` for ARM machines.

### 4. Install or upgrade nodes

Point the machine config at your installer image:

```yaml
machine:
  install:
    image: ghcr.io/<you>/talos-installer-nvidia:v1.14.5
```

Or upgrade an existing node in place:

```sh
talosctl -n <node> upgrade --image ghcr.io/<you>/talos-installer-nvidia:v1.14.5
```

### 5. Load the NVIDIA kernel modules

Apply this machine config patch so the driver loads at boot:

```yaml
machine:
  kernel:
    modules:
      - name: nvidia
      - name: nvidia_uvm
      - name: nvidia_drm
      - name: nvidia_modeset
```

See the [Talos NVIDIA GPU guide](https://docs.siderolabs.com/talos/latest/configure-your-talos-cluster/hardware-and-drivers/nvidia-gpu-proprietary)
for the full setup, including the Kubernetes GPU operator.

### Using a profile instead of flags

Imager also reads a `profile.yaml` from stdin. Use this for an air-gapped
build, where you export each extension to a tarball with `crane export` and
reference it with `tarballPath` instead of `imageRef`:

```yaml
# profile.yaml
arch: amd64
platform: metal
secureboot: false
version: v1.14.5
input:
  kernel:
    path: /usr/install/amd64/vmlinuz
  initramfs:
    path: /usr/install/amd64/initramfs.xz
  baseInstaller:
    imageRef: ghcr.io/siderolabs/installer-base:v1.14.5
  systemExtensions:
    - imageRef: ghcr.io/siderolabs/glibc:2.44@sha256:<digest>
    - imageRef: ghcr.io/siderolabs/nvidia-open-gpu-kernel-modules-production:595.91.07-v1.14.5@sha256:<digest>
    - imageRef: ghcr.io/siderolabs/nvidia-container-toolkit-production:595.91.07-v1.19.1@sha256:<digest>
    - imageRef: ghcr.io/rothgar/nvidia-pstated:latest
output:
  kind: installer
  outFormat: raw
```

```sh
cat profile.yaml | docker run --rm -i \
  -v "$PWD/_out:/out" -v /dev:/dev --privileged \
  ghcr.io/siderolabs/imager:v1.14.5 -
```

## Verify

```sh
talosctl -n <node> get extensions
talosctl -n <node> services ext-nvidia-pstated
talosctl -n <node> logs ext-nvidia-pstated
```

Check the performance state from a GPU pod or with `nvidia-smi`. An idle GPU
should report `P8`.

## Limitations

- The daemon runs with its default options (see the main README).
- The service runs the binary directly with no shell, so the fan scripts
  (`--enable-fan-script` / `--disable-fan-script`) are not supported.

## Building the extension

```sh
docker buildx build --platform linux/amd64,linux/arm64 \
  --build-arg VERSION=$(git describe --tags) \
  -f talos/Dockerfile \
  -t ghcr.io/rothgar/nvidia-pstated:$(git describe --tags) \
  --push .
```

The `.github/workflows/talos-extension.yml` workflow does this automatically on
pushes to `master` and on tags.
