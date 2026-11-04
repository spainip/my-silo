# Spainip Silo Helm chart

This chart is based on the official [PGSTY Silo chart](https://github.com/pgsty/silo/tree/main/helm/silo), upstream chart version `7.0.3` at commit `69fcf50cd2e0854af0150413df5fd66a5d74becc`. Spainip's chart version is `7.0.4`; it pins Silo `RELEASE.2026-10-26T00-00-00Z`, adds optional image digest support, and configures Kubernetes probes using Silo's native health command. The upstream chart license is included in this directory.

See the repository [README](../../README.md) for the Fleet bundle, ingress host configuration, root Secret setup, and activation steps.
