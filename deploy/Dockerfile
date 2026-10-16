# Copyright 2018 Google, Inc. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# =============================================================================
# SELF-HOSTING NOTE — read before rebuilding this image
# =============================================================================
# This fork is built *by kaniko itself* (the platform has no BuildKit builder;
# every image build goes through Argo Events sensor "sensor-for-image-build" ->
# trigger "image-build-workflow-trigger" -> WorkflowTemplate "image-template").
# Building kaniko with kaniko forced three divergences from upstream, all of
# them load-bearing:
#
#   1. kaniko ignores "RUN --mount" (pkg/commands/run.go:runCommandInExec never
#      consults Mounts), so the kaniko-base-slim stage cannot use
#      "RUN --mount=from=busybox,dst=/usr/ ...". It copies the busybox *binary*
#      instead and drives its applets by absolute path (/busybox mkdir ...),
#      removing it in the same RUN so it does not persist.
#
#   2. kaniko's RootDir is hardcoded to "/" (pkg/constants/constants.go; there
#      is no --root flag), so a COPY to /kaniko/.docker hits the pod's
#      read-only "kaniko-secret" mount. Upstream's "COPY --from=builder
#      /kaniko/.docker /kaniko/.docker" was therefore dropped; the directory is
#      supplied at runtime by the secret mount.
#
#   3. The executor binary cannot live at /kaniko/executor: with RootDir="/",
#      that path is the running builder process itself, so writing it fails with
#      "text file busy" (ETXTBSY). It is installed as /kaniko/kaniko-executor
#      and ENTRYPOINT points there.
#
# REBUILD PROCEDURE (step 1 is the one that is easy to get wrong — skipping it
# fails the build with ETXTBSY; skipping step 3 silently ships the old executor):
#
#   Step 1. Bump the destination path. The collision in (3) is unavoidable with
#           a fixed path: the image's ENTRYPOINT *is* the build's output, so
#           builder path == output path forever. Change the name in BOTH stages
#           that install the executor, and keep them in sync:
#             - kaniko-executor stage: COPY ... /kaniko/kaniko-executor + ENTRYPOINT
#             - kaniko-slim stage:     COPY ... /kaniko/kaniko-executor + ENTRYPOINT
#           (kaniko-debug inherits FROM kaniko-executor, so it follows for free.)
#           e.g. kaniko-executor -> kaniko-executor-v2 -> kaniko-executor-v3 ...
#           The running builder for this build is the *previous* custom image, so
#           the new name must differ from whatever that image's ENTRYPOINT is.
#
#   Step 2. Rebuild through the platform (same aoneci/pipeline path as any other
#           image), note the pushed digest.
#
#   Step 3. Pin the new digest in the shared template, otherwise app builds keep
#           using the old executor:
#             gitlab-app-manifests/foundation/workflows/base/sub-template/image-template.yaml
#             -> templates[0].container.image
#
# Rebuilds are rare: only needed when the executor source changes (e.g. the
# cross-stage COPY --from cache-key fix in pkg/executor/build.go). When that
# happens, expect to redo steps 1-3.
# =============================================================================

FROM golang:1.24 AS builder
WORKDIR /src

# This arg is passed by docker buildx & contains the target CPU architecture (e.g., amd64, arm64, etc.)
ARG TARGETARCH
ARG TARGETOS

ENV GOARCH=$TARGETARCH
ENV GOOS=$TARGETOS

ENV CGO_ENABLED=0
ENV GOBIN=/usr/local/bin

COPY . .

#  dependencies https://github.com/golang/go/issues/48332

# The versions of these tools and their transitive dependencies are controlled via go.mod.
# To update the version of any tool installed here, run
#
#   go get <tool>@<commit-sha-or-tag>
#   go mod vendor
#
# Then submit a PR with the changes to `go.mod`, `go.sum`, and `vendor`.

# Get GCR credential helper
RUN go install github.com/GoogleCloudPlatform/docker-credential-gcr/v2

# Get Amazon ECR credential helper
RUN go install github.com/awslabs/amazon-ecr-credential-helper/ecr-login/cli/docker-credential-ecr-login

# Get ACR docker env credential helper
RUN go install github.com/chrismellard/docker-credential-acr-env

RUN \
  --mount=type=cache,target=/root/.cache/go-build \
  --mount=type=cache,target=/go/pkg \
  make out/executor out/warmer

# Generate latest ca-certificates
FROM debian:bookworm-slim AS certs
ENV HTTP_PROXY=http://aisw_proxy:aisw_proxy1688@na131t-swdevops-proxy01.eng.t-head.cn:3128 \
    HTTPS_PROXY=http://aisw_proxy:aisw_proxy1688@na131t-swdevops-proxy01.eng.t-head.cn:3128 \
    http_proxy=http://aisw_proxy:aisw_proxy1688@na131t-swdevops-proxy01.eng.t-head.cn:3128 \
    https_proxy=http://aisw_proxy:aisw_proxy1688@na131t-swdevops-proxy01.eng.t-head.cn:3128
RUN sed -i 's|http://deb.debian.org/debian|http://mirrors.tuna.tsinghua.edu.cn/debian|g' /etc/apt/sources.list.d/debian.sources && \
    sed -i 's|http://deb.debian.org/debian-security|http://mirrors.tuna.tsinghua.edu.cn/debian-security|g' /etc/apt/sources.list.d/debian.sources && \
    apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# use musl busybox since it's staticly compiled on all platforms
FROM busybox:musl AS busybox

FROM scratch AS kaniko-base-slim

# Create kaniko directory with world write permission to allow non root run.
# BuildKit's RUN --mount=from is not implemented by kaniko, which would run the
# command on the bare scratch rootfs and fail with "busybox: executable file not
# found". Copy just the busybox binary in and drive its applets by absolute path
# (/busybox mkdir ...), since the applet symlinks are not copied and would not be
# found on PATH. Remove the binary in the same RUN so it does not persist.
COPY --from=busybox /bin/busybox /busybox
RUN ["/busybox", "sh", "-c", "/busybox mkdir -p /kaniko && /busybox chmod 777 /kaniko && /busybox rm -f /busybox"]

COPY --from=certs /etc/ssl/certs/ca-certificates.crt /kaniko/ssl/certs/
COPY files/nsswitch.conf /etc/nsswitch.conf
ENV HOME /root
ENV USER root
ENV PATH /usr/local/bin:/kaniko
ENV SSL_CERT_DIR=/kaniko/ssl/certs
ENV DOCKER_CONFIG /kaniko/.docker/

FROM kaniko-base-slim AS kaniko-base

COPY --from=builder --chown=0:0 /usr/local/bin/docker-credential-gcr /kaniko/docker-credential-gcr
COPY --from=builder --chown=0:0 /usr/local/bin/docker-credential-ecr-login /kaniko/docker-credential-ecr-login
COPY --from=builder --chown=0:0 /usr/local/bin/docker-credential-acr-env /kaniko/docker-credential-acr-env

# NOTE: upstream copies an empty /kaniko/.docker from the builder here. When this
# image is built by kaniko itself, /kaniko/.docker is already the pod's read-only
# kaniko-secret mount, so writing it fails with "chown /kaniko/.docker: read-only
# file system". The directory is empty anyway and is provided at runtime by the
# secret mount, so it is intentionally not created in the image.

ENV DOCKER_CONFIG /kaniko/.docker/
ENV DOCKER_CREDENTIAL_GCR_CONFIG /kaniko/.config/gcloud/docker_credential_gcr_config.json
WORKDIR /workspace

### FINAL STAGES ###

FROM kaniko-base AS kaniko-warmer

COPY --from=builder /src/out/warmer /kaniko/warmer

ENTRYPOINT ["/kaniko/warmer"]

FROM kaniko-base AS kaniko-executor

# The executor binary cannot be installed at /kaniko/executor when this image is
# built by kaniko: RootDir is "/" (see SELF-HOSTING NOTE at the top of this file),
# so /kaniko/executor is the running builder process itself and writing it fails
# with "text file busy" (ETXTBSY). Install it under a different name and point
# ENTRYPOINT at it. On rebuild this name MUST match the kaniko-slim stage below.
COPY --from=builder /src/out/executor /kaniko/kaniko-executor

ENTRYPOINT ["/kaniko/kaniko-executor"]

FROM kaniko-executor AS kaniko-debug

ENV PATH /usr/local/bin:/kaniko:/busybox

COPY --from=builder /src/out/warmer /kaniko/warmer

COPY --from=busybox /bin /busybox
# Declare /busybox as a volume to get it automatically in the path to ignore
VOLUME /busybox

RUN ["/busybox/mkdir", "-p", "/bin"]
RUN ["/busybox/ln", "-s", "/busybox/sh", "/bin/sh"]

FROM kaniko-base-slim AS kaniko-slim

# kaniko-slim is the default pushed stage (no --target is passed by
# image-template). On rebuild this path MUST match the kaniko-executor stage
# above — see SELF-HOSTING NOTE at the top of this file.
COPY --from=builder /src/out/executor /kaniko/kaniko-executor

ENTRYPOINT ["/kaniko/kaniko-executor"]