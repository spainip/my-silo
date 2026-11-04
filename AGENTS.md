# Spainip K3s deployment and ingress standards

This file records the deployment and Traefik ingress conventions verified on the `spainip-k3s` server on 2026-11-04. The live GitOps source of truth is [`spainip/infra`](https://github.com/spainip/infra), especially its `fleet/`, `infrastructure/`, and `platform/` directories. Treat this document as operator guidance; if it conflicts with reconciled configuration in `spainip/infra`, verify the live cluster and update this file.

## Deploying workloads

- Work directly on the `main` branch for all repository work. Commit changes to `main`; do not create a topic branch or require a pull request.
- Manage workload changes through Git and Rancher Fleet. Do not make lasting workload changes with `kubectl apply`, Helm CLI, or edits to live objects; Fleet will reconcile them away. Use `kubectl` for inspection and troubleshooting. Make the change in its owning Git repository and wait for Fleet to report the bundle as Ready before considering deployment complete.
- Package this repository's deployable workload as a Helm chart. Publish the chart to the repository's GitHub Pages Helm repository automatically whenever chart content changes, and use the published chart as the deployment source through Fleet. Keep deployment configuration pointed at the published chart and its version; do not deploy a separate copy of the workload manifests.
- Include a comprehensive `values.yaml` in the chart. It must expose and document the supported deployment settings with safe defaults, including image repository and digest, replica and update behavior, service and ingress settings, resource requests and limits, probes, persistence, pod security/context settings, and optional environment or secret references as applicable. Keep credentials and secret payloads out of chart values; reference existing Kubernetes Secrets instead.
- The central `spainip/infra` repository registers app repositories as Fleet `GitRepo` resources under `fleet/`. Each `GitRepo` selects a bundle path; put an app's Kubernetes manifests and optional `fleet.yaml` in the path that its registration tracks. The cluster's bootstrap registration tracks `fleet/` in `spainip/infra`.
- Keep each app's resources in its own namespace. Follow the existing app bundle conventions and reference existing cluster secrets by name. Do not commit secret payloads, credentials, or generated ACME account/certificate data. Use the protected secret-management process for secret creation and rotation.
- Pin container images by immutable digest (`image@sha256:...`) for deployed workloads. Use the existing registry pull secret where a private image requires it.
- Keep persistent data and node-local storage declarations in the infrastructure-owned configuration, with an explicit namespace, volume ownership, and retention policy. A workload's removal must account for its Fleet `keepResources` setting and retained PVC/PV/data; do not assume deleting a GitRepo deletes or preserves resources safely.
- Deploy configuration changes through commits to the owning repository and let Fleet reconcile them. Check the relevant Fleet GitRepo/bundle and resulting Kubernetes resources after the change. Diagnose drift at its source instead of hiding unexplained differences with comparison patches.

## Traefik ingress

Traefik is installed and upgraded by the Fleet bundle in `spainip/infra/fleet/traefik`. It runs in namespace `traefik`, uses the `traefik` ingress class, and exposes a dual-stack `LoadBalancer` on ports 80 and 443. Public DNS names point to the server's IPv4 and IPv6 addresses. K3s's bundled Traefik is disabled; use the Fleet-managed installation.

- Prefer Traefik's `traefik.io/v1alpha1` `IngressRoute` for new application routes. Put it in the application's namespace and route to a Kubernetes Service in that namespace. Current app routes use `entryPoints: [websecure]`, a `Host(...)` rule, and `tls.certResolver: acmeresolver`.
- The `web` entry point redirects HTTP to `websecure` globally. Do not create public HTTP-only application routes. Keep public applications on HTTPS and use the shared `acmeresolver` TLS-ALPN ACME resolver unless the service has an explicitly approved certificate arrangement.
- `Ingress` resources are supported for compatibility or chart-generated resources. Set `spec.ingressClassName: traefik`; use `websecure`, TLS enabled, and `acmeresolver` via the corresponding `traefik.ingress.kubernetes.io/router.*` annotations. Do not rely on the default ingress class: it is intentionally not default.
- Hostnames must be unique unless a deliberate host-and-path split is required. Use the exact intended DNS hostname and a Kubernetes Service port (not a Pod IP or container address). For path routing, make the match rule explicit and check that application base paths and path-prefix behavior agree.
- Traefik's Kubernetes CRD provider has `allowCrossNamespace: false`. Keep an `IngressRoute` and its referenced Service and Middleware in the same namespace. Define app-specific Middleware resources in that namespace too.
- Protect administrative and sensitive routes with authentication at the proxy or application layer. For Traefik BasicAuth, reference a Kubernetes Secret from a Middleware, set a suitable realm, and enable `removeHeader: true` when the upstream should not receive the credentials. Never publish an unauthenticated dashboard or management endpoint. The Traefik dashboard is separately protected by its namespace-local BasicAuth middleware.
- Do not edit Traefik chart values or global entry-point behavior for an individual app. App routes belong to the owning workload bundle; shared Traefik configuration belongs in `spainip/infra/fleet/traefik` and should be changed with care because it affects every ingress.

Example for a new public service (replace the namespace, host, route name, Service name, and port):

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: app
  namespace: app
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(`app.spainip.es`)
      services:
        - name: web
          port: 8080
  tls:
    certResolver: acmeresolver
```

The resolver, entry points, chart values, and ingress class are configured in `spainip/infra`; do not duplicate those cluster-level settings in each application bundle.
