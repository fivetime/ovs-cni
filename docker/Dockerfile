# syntax=docker/dockerfile:1.7
###############################################################################
# Stage 1: builder —— 编译 4 个 ovs-cni 二进制
#
# 与上游 cmd/Dockerfile 行为等价的几点：
#   1) ARG TARGETOS/TARGETARCH/BUILDOS/BUILDARCH 由 docker buildx 在多架构
#      构建时自动注入，编出的二进制平台与 buildx 请求一致。
#   2) Dockerfile 不自己跑 hack/get_version.sh —— 与上游一致，要求 CI
#      workflow 先在 host 上执行 `./hack/get_version.sh > .version`，结果
#      文件经 build context 进入镜像后被 runtime 拷到 /.version。
#   3) GOFLAGS=-mod=vendor + -tags no_openssl + CGO_ENABLED=0 与上游一致，
#      产物是纯静态二进制，runtime 无需 glibc。
###############################################################################
FROM --platform=$BUILDPLATFORM golang:1.25-alpine3.23 AS builder

ARG VERSION="dev"

# 兼容 `make docker-build goarch=...` 的本地用法（同上游）
ARG goarch=

ARG TARGETOS
ARG TARGETARCH
ARG BUILDOS
ARG BUILDARCH

ENV GOOS=${TARGETOS:-linux} \
    GOARCH=${goarch:-${TARGETARCH:-amd64}} \
    CGO_ENABLED=0 \
    GOFLAGS=-mod=vendor

WORKDIR /workdir

COPY . .

RUN set -eux \
    && mkdir -p /workdir/bin \
    && go build -tags no_openssl -ldflags="-s -w" -o /workdir/bin/ovs                  ./cmd/plugin \
    && go build -tags no_openssl -ldflags="-s -w" -o /workdir/bin/marker               ./cmd/marker \
    && go build -tags no_openssl -ldflags="-s -w" -o /workdir/bin/ovs-mirror-producer  ./cmd/mirror-producer \
    && go build -tags no_openssl -ldflags="-s -w" -o /workdir/bin/ovs-mirror-consumer  ./cmd/mirror-consumer

###############################################################################
# Stage 2: runtime —— 仅保留 ovs-cni 运行所需
#
#   - Alpine busybox 自带 `find -mmin`，DaemonSet liveness 探针
#     (`find /tmp/healthy -mmin -2 | grep -q ...`) 无需额外安装 findutils。
#   - ca-certificates 给 marker 调 K8s apiserver 时校验 TLS 链用。
#   - 不设 ENTRYPOINT/CMD：四个二进制是被 DaemonSet 的 install/marker 容器
#     按需调用的，不是直接 `docker run` 的常驻进程（与上游布局一致）。
###############################################################################
FROM alpine:3.23

ARG VERSION="dev"

LABEL org.opencontainers.image.title="ovs-cni-plugin" \
      org.opencontainers.image.description="Open vSwitch CNI plugin + marker + mirror producer/consumer (Alpine build)" \
      org.opencontainers.image.source="https://github.com/fivetime/ovs-cni" \
      org.opencontainers.image.version="${VERSION}" \
      org.opencontainers.image.licenses="Apache-2.0"

RUN apk add --no-cache ca-certificates

COPY --from=builder /workdir/.version                 /.version
COPY --from=builder /workdir/bin/ovs                  /
COPY --from=builder /workdir/bin/marker               /
COPY --from=builder /workdir/bin/ovs-mirror-producer  /
COPY --from=builder /workdir/bin/ovs-mirror-consumer  /
