
ARG ENVIRONMENT="dev"

FROM quay.io/konflux-ci/konflux-test:latest AS builder

ARG ENVIRONMENT

WORKDIR /tmp

COPY ./base/ ./base/
COPY ./overlay/ ./overlay/

RUN oc kustomize -o /tmp/manifests.yaml ./overlay/$ENVIRONMENT

FROM scratch

LABEL maintainer="Red Hat, Inc."
LABEL vendor="Red Hat, Inc."
LABEL url="https://www.redhat.com"
LABEL distribution-scope="private"

# Copy files to locations specified by labels.
COPY --from=builder /tmp/manifests.yaml /manifests.yaml
