# Podinfo on UDS Core or Zarf-only Kubernetes

Podinfo upstream: https://github.com/stefanprodan/podinfo
OCI application chart: oci://ghcr.io/stefanprodan/charts/podinfo, pinned to 6.9.2.
Zarf package version: 0.1.0. Requires Zarf Package Values support.

## Files

- zarf.yaml: OCI application chart and local integration chart; input mappings.
- package-values.yaml: package defaults including uds.enabled: true.
- helm-values.yaml: static settings for the upstream application chart.
- site-values.yaml: application overrides (replicas, message, color).
- zarf-only-values.yaml: sets uds.enabled: false.
- charts/uds-integration/: small Helm chart that conditionally renders a UDS Package.

## Build once

```bash
zarf package create . --confirm
```

Creation requires access to GHCR. The package includes the chart, local integration
chart, and image for offline deployment. Replace amd64 with arm64 if appropriate.

## UDS Core deployment (default)

Requires UDS Core with the Operator, ambient mesh, and tenant gateway.

```bash
zarf package deploy zarf-package-podinfo-values-demo-amd64-0.1.0.tar.zst \
  --values site-values.yaml --confirm
kubectl -n podinfo get packages.uds.dev podinfo
```

Open https://podinfo.uds.dev for the default local tenant domain; otherwise use
https://podinfo.<your-tenant-domain>. DNS must resolve to the tenant gateway.
app.host changes the hostname prefix. The integration configures ambient mesh and
tenant ingress. SSO and monitoring are not configured.

## Zarf-only deployment

Requires a Zarf-initialized Kubernetes cluster; UDS Core is not required.

```bash
zarf package deploy zarf-package-podinfo-values-demo-amd64-0.1.0.tar.zst \
  --values site-values.yaml --values zarf-only-values.yaml --confirm
```

Equivalent flag: --set-values uds.enabled=false

The integration Helm release renders no Kubernetes resources when disabled.
The upstream application Deployment and Service still render normally.

```bash
kubectl -n podinfo port-forward svc/podinfo 8080:9898
```

Open http://localhost:8080. A Zarf-only cluster has no UDS tenant gateway route.

## Values mapping

| Zarf package input | Helm chart | Chart value |
| --- | --- | --- |
| app.replicas | Podinfo | replicaCount |
| app.message | Podinfo | ui.message |
| app.color | Podinfo | ui.color |
| app.host | uds-integration | host |
| uds.enabled | uds-integration | enabled |

The integration chart uses an explicit Helm if .Values.enabled condition. Keep the
value boolean (false), not the quoted string "false". Its schema enforces this.
Use templates/ for the custom resource, not crds/. The UDS CRD is provided by Core.

This option is for choosing the target environment. Turning it off on an existing
UDS deployment can delete the Package CR and trigger UDS Operator cleanup; it is
not a migration procedure for removing Core from a running application.

## Validation

Pulled the pinned OCI chart and rendered the application defaults and site values.
Rendered the integration chart enabled and disabled and verified that only the
enabled render includes the UDS Package. Checked the service name, port, namespace,
and selector against the application chart. No Zarf build or live deployment run.

References:
https://docs.zarf.dev/ref/package-values/
https://helm.sh/docs/chart_template_guide/control_structures/
