# ovs-cni container image (Alpine build)

This directory builds the `ovs-cni-plugin` image on **Alpine** instead of the
upstream `cmd/Dockerfile` which uses CentOS Stream 9 + ubi-minimal. Published
to `ghcr.io/fivetime/ovs-cni-plugin`.

| Tag | Meaning |
| --- | --- |
| `latest` | Newest commit on `main` |
| `<git-sha>` | Specific commit |
| `v<x.y.z>` | Upstream release tag synced via `sync-upstream` |

Images are multi-arch: `linux/amd64`, `linux/arm64`, `linux/ppc64le`,
`linux/s390x` (built on a single x86 self-hosted runner via QEMU; native
amd64 is direct).

## Why this Dockerfile exists

- Upstream `cmd/Dockerfile` pulls `quay.io/centos/centos:stream9` and runs
  `dnf install -y wget`, which hits `mirror.stream.centos.org`. From a
  self-hosted runner this is unreliable and frequently fails on metadata
  download. The Alpine path uses `golang:1.25-alpine3.23` (no extra package
  installs in the builder) and an `alpine:3.23` runtime.
- The four binaries (`ovs`, `marker`, `ovs-mirror-producer`,
  `ovs-mirror-consumer`) and `/.version` end up in the **same paths** as the
  upstream image, so any DaemonSet manifest written against the upstream image
  works against this one without changes.
- Runtime image stays small: `ca-certificates` only. BusyBox's `find -mmin`
  satisfies the existing liveness probe, no `findutils` package needed.

## Building locally

The build context must be the repo root (the Dockerfile does `COPY . .`),
and `.version` must already exist:

```bash
# from repo root
./hack/get_version.sh > .version
docker build -f docker/Dockerfile -t ovs-cni-plugin:dev .
```

For multi-arch (matches what CI does):

```bash
./hack/get_version.sh > .version
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/ppc64le,linux/s390x \
  -f docker/Dockerfile \
  -t ghcr.io/fivetime/ovs-cni-plugin:dev \
  --push .
```

CI workflows (`image-push-main.yaml` / `image-push-release.yaml`) already
handle the `.version` step and per-arch matrix.

## Deploying

The CNI binaries are installed onto the host by the DaemonSet's install
container — they are not long-running processes inside this image. See
`example.yml` for the minimal DaemonSet snippet, or pin the image in the
upstream `manifests/ovs-cni.yml.in` (replace the upstream registry path with
`ghcr.io/fivetime/ovs-cni-plugin:latest`).
