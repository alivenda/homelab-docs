# Kubernetes backend for Woodpecker

**Date:** 2026-01
**Status:** Active

## Context

Woodpecker CI supports two main backends for running pipeline steps: Docker
(mounts `/var/run/docker.sock` into the agent) and Kubernetes (spins up
short-lived pods in the cluster).

The Docker backend requires the agent container to mount the host's Docker
socket — a privileged escape path that gives any pipeline step root access to
the host.

## Decision

Use the Kubernetes backend (`WOODPECKER_BACKEND: kubernetes`) instead of the
Docker backend.

## Consequences

- No `/var/run/docker.sock` mount in the agent. Pipeline steps run as
  unprivileged pods in the `woodpecker` namespace with their own RBAC.
- Image builds can't use `docker build`. BuildKit (rootless) builds OCI images
  directly inside a build pod — no daemon, no socket mount, no privileged
  container.
- Step pods use a dedicated `woodpecker-ci-scratch` StorageClass and ephemeral
  volumes, sized at 1 GB per step.
- Heavy CI jobs are pinned to nodes with `workload=heavy` through the backend's
  `WOODPECKER_BACKEND_K8S_POD_NODE_SELECTOR`.
