# msa-infrastructure

Kubernetes cluster infrastructure for running [msa-templates](https://github.com/ftechmax/msa-templates) based services.
Each component consists of a `base` and one or more environment specific `overlays`, following the standard
Kustomize base/overlay pattern.

## Components

| Directory         | Purpose                                                                                                   |
| ----------------- | --------------------------------------------------------------------------------------------------------- |
| `istio/`          | Istio (ambient profile) control plane, Gateway API CRDs, and north-south `Gateway` in `istio-ingress`.    |
| `cert-manager/`   | [cert-manager](https://cert-manager.io/) for certificate issuance.                                        |
| `sealed-secrets/` | [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) controller.                      |
| `reloader/`       | [Stakater Reloader](https://github.com/stakater/Reloader) restarts workloads on ConfigMap/Secret change.  |
| `rabbitmq/`       | RabbitMQ cluster + messaging-topology operators, admin user, management UI.                               |
| `opentelemetry/`  | OpenTelemetry Collector receiving OTLP (traces/metrics/logs) for services to export to.                   |
| `registry/`       | In-cluster OCI image registry (NodePort `32000`) for local development.                                   |
| `k3s/`            | [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller) + a k3s upgrade `Plan`. |

## Prerequisites

- A running Kubernetes cluster
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/)
- [`helm`](https://helm.sh/docs/intro/install/)

## Repository layout

```
<component>/
├── base/               # reusable manifests
├── crd/                # CRDs / operators that must be installed first
└── overlays/
    └── local/          # standard development environment overlay
```

### CRDs

A `crd/` directory holds the CRDs and operators that must be deployed before the
resources depending on them. Apply these first.

## Deploying

Order matters: apply CRDs/operators and Istio before the workloads that depend
on them.

```sh
# 1. CRDs / operators
kubectl apply -k k3s/crd
kubectl apply -k rabbitmq/crd

# 2. Platform: cert-manager, sealed-secrets, reloader
kubectl apply -k cert-manager/overlays/local
kubectl apply -k sealed-secrets/overlays/local
kubectl apply -k reloader/overlays/local

# 3. Networking
kubectl kustomize --enable-helm istio/overlays/local | kubectl apply -f -

# 4. Messaging / observability / registry
kubectl apply -k rabbitmq/overlays/local
kubectl apply -k opentelemetry/overlays/local
kubectl apply -k registry/overlays/local
```

## Observability

Services export telemetry over OTLP to the collector in the `observability`
namespace at `otel-collector.observability:4317` (gRPC) or `:4318` (HTTP). The
default pipeline receives traces, metrics, and logs and writes them to the
collector's own logs via the `debug` exporter — there is **no external backend
configured**. Add exporters (e.g. `otlphttp` to Tempo/Loki, `prometheus`) to
`opentelemetry/base/configmap.yaml` and wire them into the pipelines to ship
telemetry to a real backend.

## Service mesh

The cluster runs Istio in the **ambient** profile. A namespace joins the mesh
when it carries the `istio.io/dataplane-mode: ambient` label; its pods then get
transparent L4 mTLS through ztunnel.

Namespaces that carry application or data-plane traffic are meshed so that
traffic between them is authenticated and encrypted.

| Namespace         | Meshed | Reason                                                                          |
| ----------------- | ------ | ------------------------------------------------------------------------------- |
| `istio-ingress`   | yes    | Terminates north-south traffic at the `Gateway`; the entry point into the mesh. |
| `observability`   | yes    | Receives OTLP telemetry from every service.                                     |
| `rabbitmq-system` | yes    | Message bus for the services; in-mesh so publishers/consumers get mTLS.         |
| `registry`        | no     | Images aren't confidential.                                                     |
| `istio-system`    | no     | The mesh control plane itself must not be enrolled in its own data plane.       |
| `kube-system`     | no     | Core Kubernetes and sealed-secrets.                                             |
| `system-upgrade`  | no     | Runs privileged node-upgrade jobs that bypass normal pod networking.            |
| `reloader-system` | no     | Only calls the Kubernetes API server.                                           |

mTLS is currently **PERMISSIVE**: meshed pods negotiate mTLS with each other but
still accept plaintext, so out-of-mesh clients such as the registry reached
over its NodePort keep working.

## Accessing services

Web UIs are exposed through the Istio `Gateway` in `istio-ingress` (HTTP on
port 80) via `HTTPRoute`s on the `*.kube.local` domain:

| Service             | Hostname              |
| ------------------- | --------------------- |
| RabbitMQ management | `rabbitmq.kube.local` |

Point these hostnames at the gateway's ingress IP, e.g. in `/etc/hosts`:

```
<gateway-ip> rabbitmq.kube.local
```

The image registry is reached directly on the node via NodePort `32000`
(`localhost:32000` on a single-node cluster).

## Secrets

The `local` overlays include **plaintext** demo credentials for convenience on local development clusters.
Use the deployed Sealed Secrets controller with [`kubeseal`](https://github.com/bitnami/sealed-secrets#kubeseal) to encrypt secrets for production use.
