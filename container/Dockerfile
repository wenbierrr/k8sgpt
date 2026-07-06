# Copyright 2023 The K8sGPT Authors.
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#     http://www.apache.org/licenses/LICENSE-2.0
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

FROM golang:1.26-alpine3.23 AS builder

ENV CGO_ENABLED=0
ARG VERSION
ARG COMMIT
ARG DATE
WORKDIR /workspace

COPY go.mod go.sum ./
RUN go mod download

COPY ./ ./

# --- Security patching ---
# 1) Drop unused integrations (keda, prometheus). We only use the core SRE
#    analyze/serve functionality. This removes github.com/docker/docker
#    (CVE-2026-41567 / CVE-2026-42306: no fixed version available),
#    github.com/docker/cli, github.com/containerd/containerd and
#    github.com/prometheus/prometheus from the linked binary entirely.
RUN rm -rf pkg/integration/keda pkg/integration/prometheus && \
    sed -i \
      -e '/k8sgpt\/pkg\/integration\/keda"/d' \
      -e '/k8sgpt\/pkg\/integration\/prometheus"/d' \
      -e '/"prometheus": *prometheus\.NewPrometheus(),/d' \
      -e '/"keda": *keda\.NewKeda(),/d' \
      pkg/integration/integration.go

# 2) Bump remaining vulnerable modules (still used by core code) to fixed versions:
#    x/crypto   -> v0.52.0 (CVE-2026-39827/28/29/30/32/35, CVE-2026-42508, CVE-2026-46595/97)
#    x/net      -> v0.55.0 (CVE-2026-25681, CVE-2026-27136, CVE-2026-33814, CVE-2026-39821, CVE-2026-42502)
#    otel       -> v1.43.0 (CVE-2026-29181, CVE-2026-24051, CVE-2026-39883)
#    go-jose/v4 -> v4.1.4  (CVE fixed in 4.1.4)
#    spdystream -> v0.5.1, containerd -> v1.7.33 (in case they remain linked)
RUN go get \
      golang.org/x/crypto@v0.52.0 \
      golang.org/x/net@v0.55.0 \
      github.com/go-jose/go-jose/v4@v4.1.4 \
      github.com/moby/spdystream@v0.5.1 \
      github.com/containerd/containerd@v1.7.33 \
      go.opentelemetry.io/otel@v1.43.0 \
      go.opentelemetry.io/otel/metric@v1.43.0 \
      go.opentelemetry.io/otel/trace@v1.43.0 \
      go.opentelemetry.io/otel/sdk@v1.43.0 \
      go.opentelemetry.io/otel/sdk/metric@v1.43.0
# --- End security patching ---

RUN go build -o /workspace/k8sgpt -ldflags "-X main.version=${VERSION} -X main.commit=${COMMIT} -X main.date=${DATE}" ./

FROM gcr.io/distroless/static AS production

LABEL org.opencontainers.image.source="https://github.com/k8sgpt-ai/k8sgpt" \
    org.opencontainers.image.url="https://k8sgpt.ai" \
    org.opencontainers.image.title="k8sgpt" \
    org.opencontainers.image.vendor='The K8sGPT Authors' \
    org.opencontainers.image.licenses='Apache-2.0'

WORKDIR /
COPY --from=builder /workspace/k8sgpt .
USER 65532:65532

ENTRYPOINT ["/k8sgpt"]
