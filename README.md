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

## Network policies

Every namespace this repository deploys, except `system-upgrade` (see
[Not covered](#not-covered)), carries Kubernetes `NetworkPolicy`s. A
`default-deny` policy isolates every pod in the namespace in both directions,
allowing only DNS out; per-workload policies then open the connections below.
Outbound traffic to the internet is denied. Kubelet health probes keep working,
because the CNI always allows traffic from a pod's own node.

In `kube-system` only the sealed-secrets controller is selected, and the
registry policy selects only the registry pod, so both are safe in a namespace
shared with other workloads.

| Namespace         | Workload                    | Inbound                                                                                                | Outbound                                  |
| ----------------- | --------------------------- | ------------------------------------------------------------------------------------------------------ | ----------------------------------------- |
| `istio-ingress`   | Gateway                     | Anyone on `80`, `443`, `15021`                                                                         | istiod on `15012`; any pod in the cluster |
| `istio-system`    | istiod                      | Gateway on `15012`; private ranges on `15017` (webhooks)                                               | API server on `6443`                      |
| `rabbitmq-system` | RabbitMQ                    | Allow-listed namespaces on `5672`; gateway + topology operator on `15672`; RabbitMQ on `4369`, `25672` | RabbitMQ on `4369`, `25672`               |
| `rabbitmq-system` | RabbitMQ operators          | Private ranges on `9443` (webhooks)                                                                    | API server on `6443`                      |
| `rabbitmq-system` | messaging-topology-operator | (as above)                                                                                             | RabbitMQ management on `15672`            |
| `observability`   | OpenTelemetry Collector     | Allow-listed namespaces on `4317`, `4318`                                                              | Nothing beyond DNS                        |
| `cert-manager`    | cert-manager                | Private ranges on `10250` (webhook)                                                                    | API server on `6443`                      |
| `reloader-system` | Reloader                    | Nothing                                                                                                | API server on `6443`                      |
| `registry`        | Registry                    | Private ranges on `5000`                                                                               | Nothing                                   |
| `kube-system`     | sealed-secrets controller   | Private ranges on `8080` (`kubeseal` via the API server)                                               | API server on `6443`                      |

The Istio policies live in `istio/network-policies/` rather than `istio/base/`,
because the base's label transformer would add `app.kubernetes.io/part-of` to
their pod selectors.

### Allowing application namespaces

AMQP and OTLP are denied until an application namespace is allow-listed. Each
environment overlay lists the namespaces in `client-namespaces.yaml`:

- `rabbitmq/overlays/local/client-namespaces.yaml` for AMQP
- `opentelemetry/overlays/local/client-namespaces.yaml` for OTLP

Add one rule per namespace, following the commented example in each file. A
rule with an empty or missing `from` allows every source, so never leave one
behind when removing a namespace.

### API server access and webhooks

The API server runs outside the pod network, on port `6443` of the node with
k3s. Rules for API server egress and for admission webhooks the API server
calls match the private address ranges (`10.0.0.0/8`, `172.16.0.0/12`,
`192.168.0.0/16`), so nothing can use those ports to reach the internet.

To narrow API server egress to the node network, uncomment and fill in the
commented patch in each local overlay: `istio`, `rabbitmq`, `cert-manager`,
`reloader`, and `sealed-secrets`. On a cluster whose API server listens on
another port, patch the same policies.

### Adding outbound dependencies

- An exporter added to the collector needs an egress rule for its backend in a
  policy selecting `app: otel-collector`.
- An ACME `ClusterIssuer`, such as Let's Encrypt, needs egress for the
  cert-manager controller to the ACME server and, for DNS-01, the DNS
  provider's API. Until that rule exists, ACME issuers can't register or issue.
- A new Gateway listener on another port needs that port added to the
  `gateway` policy.

### Not covered

- `system-upgrade`: its upgrade jobs download k3s from the internet.
- The k3s components in `kube-system` (CoreDNS, metrics-server,
  local-path-provisioner, `svclb`): CoreDNS forwards to upstream resolvers and
  the others are managed by k3s, so a mistake there takes the cluster down.

DNS stays allowed for every pod that resolves names, so a compromised
dependency could still leak data through DNS queries. Blocking that needs a
DNS-level filter, not a network policy.

The policies assume ztunnel is not running. With the ambient data plane
enabled, traffic between meshed pods arrives on port `15008` and kubelet probes
come from `169.254.7.127`, and both need extra rules.

The policies need a CNI that enforces them; k3s does this out of the box.

## Accessing services

Configure the HTTPS domain and TLS Secret name in
`istio/overlays/local/settings.yaml` (default: `kube.internal` and
`gateway-tls`). The Secret must exist in `istio-ingress` with a certificate
covering `*.kube.internal`. HTTP requests for the configured domain and
its subdomains redirect to HTTPS with status 301.

Web UIs are exposed through the Istio `Gateway` in `istio-ingress` (HTTPS on
port 443) via `HTTPRoute`s on the `*.kube.internal` domain. If you change the
domain, update the hostname in `rabbitmq/management/httproute.yaml` to match.

| Service             | Hostname                 |
| ------------------- | ------------------------ |
| RabbitMQ management | `rabbitmq.kube.internal` |

Point these hostnames at the gateway's ingress IP, e.g. in `/etc/hosts`:

```
<gateway-ip> rabbitmq.kube.internal
```

The image registry is reached directly on the node via NodePort `32000`
(`localhost:32000` on a single-node cluster).

## Secrets

The `local` overlays include **plaintext** demo credentials for convenience on local development clusters.
Use the deployed Sealed Secrets controller with [`kubeseal`](https://github.com/bitnami/sealed-secrets#kubeseal) to encrypt secrets for production use.
