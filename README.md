# Silo on the Spainip K3s server

This repository packages the official PGSTY Silo Helm chart for the single-node `spainip-k3s` server and provides a Fleet bundle for the `spainip` Fleet workspace. The workload namespace is `silo`.

## Chart and release

The chart in [`charts/silo`](charts/silo) is based on PGSTY's official chart `7.0.3` and adds image digest support plus native Silo liveness, readiness, and startup probes. It is published as chart version `7.0.4` to the repository's GitHub Pages Helm repository whenever chart files change on `main`.

Silo is pinned to `docker.io/pgsty/silo:RELEASE.2026-10-26T00-00-00Z` and the release's multi-architecture image digest. The official chart's standalone mode is used with one replica, a `local-path` PVC, and the server's Traefik ingress class and `acmeresolver` TLS resolver.

The API and UI hostnames are independently configurable:

- API: `ingress.hosts` in [`charts/silo/values.yaml`](charts/silo/values.yaml), with the Fleet override in [`fleet/silo/values.yaml`](fleet/silo/values.yaml).
- Web UI: `consoleIngress.hosts` in those same files.

Defaults are `silo-api.spainip.es` and `silo.spainip.es`. Add corresponding DNS records to the server's public IPv4 and IPv6 addresses before activation. The chart creates separate API and console `Ingress` resources, both assigned to Traefik with HTTPS and the shared ACME resolver.

## Root credentials Secret

The chart references an existing Kubernetes Secret and does not generate or store root credentials in Helm values. Before activating the Fleet registration, create the `silo` namespace and then create `silo-root-credentials` in it. The Secret must contain keys `rootUser` and `rootPassword`.

The following commands prompt for the credentials, place them briefly in a mode-0600 temporary file, create/update the cluster Secret, and remove the temporary file. Run them only when preparing a deployment; they have not been run as part of this change.

```bash
kubectl create namespace silo --dry-run=client -o yaml | kubectl apply -f -

umask 077
SILO_SECRET_FILE="$(mktemp)"
read -r -p "Silo root username: " SILO_ROOT_USER
read -r -s -p "Silo root password: " SILO_ROOT_PASSWORD
printf '\n'
printf 'rootUser=%s\nrootPassword=%s\n' "$SILO_ROOT_USER" "$SILO_ROOT_PASSWORD" > "$SILO_SECRET_FILE"
kubectl create secret generic silo-root-credentials \
  --namespace silo \
  --from-env-file="$SILO_SECRET_FILE" \
  --dry-run=client -o yaml | kubectl apply -f -
rm -f "$SILO_SECRET_FILE"
unset SILO_ROOT_USER SILO_ROOT_PASSWORD SILO_SECRET_FILE
```

Do not commit the Secret or its values. Keep the root credentials in the protected secret-management process and rotate them there.

## Fleet registration and activation

[`fleet/silo/fleet.yaml`](fleet/silo/fleet.yaml) tells Fleet to install chart `silo` version `7.0.4` from `https://spainip.github.io/my-silo/`. The registration descriptor [`fleet/silo-gitrepo.yaml`](fleet/silo-gitrepo.yaml) targets the existing Fleet workspace `spainip`, uses the existing `github-auth` Secret, and tracks only `fleet/silo`.

The registration descriptor is prepared here but has not been added to `spainip/infra` and has not been applied. Do not register or reconcile it until the root Secret, DNS names, and published chart are ready and deployment is explicitly requested.

## Why this chart, not the Operator

The official Silo chart values cover standalone mode, API and console Services and hostnames, Traefik ingress annotations, an existing credentials Secret, and PVC storage class selection. This package keeps that chart and adds native health probes and immutable image digest support. The PVC has Helm's `resource-policy: keep` annotation so uninstalling the release does not remove the claim and its local data; review the PVC and PV before any manual cleanup. See the [official chart](https://github.com/pgsty/silo/tree/main/helm/silo) and [Silo Kubernetes documentation](https://silo.pgsty.com/operations/deployments/).

The MinIO Operator can manage Silo through a Tenant resource, but Silo's documentation identifies its final `v7.1.1` release as an archived compatibility baseline. It requires Kubernetes 1.30 or later and adds a separate controller and Tenant CRDs. For this single-node standalone deployment, the Operator adds an unnecessary dependency; it is not included.
