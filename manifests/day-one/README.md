# Day-One Manifests

Committed Kubernetes artifacts for the Day-1 workshop sequence (Foundations -> POC -> Stable). Every command below runs from the **repository root** so the relative paths resolve verbatim.

Design contract: [`docs/tdd/day-one-reproducibility.md`](../../docs/tdd/day-one-reproducibility.md) (vote DKT-V2, approved). All values here trace to that TDD's canonical value table — do not edit values in isolation.

## Prerequisites (operator)

- The workshop image `docker.io/altf4llc/fem-kubernetes:v1` must be published to Docker Hub **before** the workshop. It is built and pushed by `.github/workflows/build-image.yml` (publishes `:v1` plus `:<git-sha>` on every build). The `:<git-sha>` tag is the immutable per-commit escape hatch — always pullable even after `:v1` is overwritten on rebuild.
- `kind`, `kubectl`, and `curl` installed on the student/instructor machine.

## File layout

| File | Stage | Form |
|---|---|---|
| `kind-cluster.yaml` | Foundations | 1 control-plane + 2 workers; host `30080` -> NGF data-plane NodePort `30080` |
| `deployment.yaml` | POC (seg 8) | POC-era: no probes, plaintext DB env, namespace `default` |
| `k8s/base/deployment.yaml` | Stable | matured: probes + CNPG creds, namespace `app` |
| `k8s/base/service.yaml` | Stable | `sample-app` ClusterIP on `8080` |
| `k8s/base/configmap.yaml` | Stable | `db-config` (DB_HOST/DB_PORT/DB_NAME) |
| `k8s/base/postgres-cluster.yaml` | Stable | CloudNativePG `Cluster` (initdb `appdb`/`appuser`) |
| `k8s/base/gateway.yaml` | Stable | `Gateway` (class `nginx`, host `sample-app.local`) |
| `k8s/base/httproute.yaml` | Stable | `HTTPRoute` -> `sample-app:8080` |
| `k8s/base/kustomization.yaml` | Stable | collects the base into one applyable unit |

## Two `deployment.yaml` files — distinct by design, not duplicates

There are two committed Deployment manifests, and the difference between them **is the workshop's thesis**: a POC Deployment maturing into a probed, operator-backed Stable base.

- **`deployment.yaml`** is the POC-era form reached at the seg-8 "Apply is declarative" slide. It has no probes, wires the database with plaintext env (`DB_HOST=postgres`, `DB_PASSWORD=demo-not-a-real-password`), and lives in `default`. It is runnable **at that moment**, against the ephemeral `postgres:16` already standing from the POC.
- **`k8s/base/deployment.yaml`** is the matured end-state applied in the Stable stage. It adds liveness/readiness/startup probes on `/healthz`, pulls credentials from the CloudNativePG-generated `postgres-app` Secret, reads non-secret config from the `db-config` ConfigMap, points at the durable `postgres-rw` Service, and lives in namespace `app`.

They share the `app: sample-app` label and the `:v1` image but differ in DB wiring and probes. If you diff them, the diff is the lesson — the POC form cannot carry the matured form's probe targets or `secretKeyRef: postgres-app` because, at seg 8, the namespace, the probes segment, the Gateway, and CloudNativePG do not exist yet. Applying the matured form there would leave the Pod unable to resolve a Secret that has not been created. So the seg-8 file is deliberately the simpler shape, and the Stable base is what students `apply -k` once the supporting resources exist.

## How students apply each

### POC (seg 4-8, namespace `default`)

The POC stands up Postgres and the app imperatively, then introduces the first declarative manifest:

```bash
kubectl apply -f manifests/day-one/deployment.yaml
```

The POC app is exposed as a `NodePort` Service (the deliberate "crude front door" anti-pattern). On `kind`, a raw NodePort is not reliably reachable from the host, so reach the app with a port-forward:

```bash
kubectl port-forward service/sample-app 8080:8080
curl localhost:8080/counter
```

The NodePort stays on the slide as the anti-pattern; the port-forward is the reliable reach.

### Stable (seg 9-14, namespace `app`)

```bash
kubectl create namespace app
kubectl apply -k manifests/day-one/k8s/base
```

(The Stable slides build these resources up incrementally first; `apply -k` at the Kustomize segment applies the whole base as one unit.)

## NGINX Gateway Fabric NodePort pin (REQUIRED for `curl localhost`)

`kind-cluster.yaml` maps host port `30080` to node port `30080`. NGINX Gateway Fabric v2.x provisions its nginx data-plane Service dynamically per `Gateway`, and the NodePort deploy variant **auto-allocates** that NodePort to a random high port. Without pinning it to `30080`, the host `30080` -> `30080` mapping points at nothing.

The same data-plane Service uses `externalTrafficPolicy: Local`, so a NodePort only answers on a node that actually runs the data-plane Pod — and `kind` maps host `30080` on the **control-plane node only**. So the patch below also pins the data-plane Pod to the control-plane node: the `nodeSelector` targets the built-in `node-role.kubernetes.io/control-plane` label and the matching toleration lets it schedule past the control-plane `NoSchedule` taint. With both pins in place, `curl localhost:30080` works on both Docker Desktop and OrbStack — Docker Desktop forwards a cross-node hop and OrbStack does not, but with the Pod on the control-plane node there is no cross-node hop to forward.

After the NGF controller is up and at/before `Gateway` creation, pin the data-plane NodePort to `30080` and the data-plane Pod to the control-plane node:

```bash
kubectl patch nginxproxy nginx-gateway-proxy-config -n nginx-gateway --type=merge \
  -p '{"spec":{"kubernetes":{"deployment":{"pod":{"nodeSelector":{"node-role.kubernetes.io/control-plane":""},"tolerations":[{"key":"node-role.kubernetes.io/control-plane","operator":"Exists","effect":"NoSchedule"}]}},"service":{"nodePorts":[{"port":30080,"listenerPort":80}]}}}}'
```

Keep the `30080` here identical to the `containerPort` in `kind-cluster.yaml`. Then:

```bash
curl -H "Host: sample-app.local" http://localhost:30080/healthz
```

## Fallbacks

- **POC reach** is already port-forward (above) — no host-port mapping needed.
- **Gateway reach** is the combined pin above — `curl localhost:30080` works on both runtimes once the data-plane Pod is on the control-plane node. Only if that pin is skipped, or host port `30080` is occupied, fall back to a port-forward against the NGF data-plane Service:

  ```bash
  kubectl port-forward -n app svc/sample-app-nginx 8080:80
  curl -H "Host: sample-app.local" http://localhost:8080/healthz
  ```

  (The per-`Gateway` data-plane Service is named `<gateway>-nginx` in the `Gateway`'s namespace; find it with `kubectl get svc -A | grep nginx`.)
- **Host-port collision:** `kind-cluster.yaml` pins host port `30080`. If something already listens on `:30080`, the Gateway port-forward fallback above sidesteps it.

## Operator notes

- Pinned tool versions (from the TDD canonical table): CloudNativePG **v1.29.1**, NGINX Gateway Fabric **v2.6.3** (Gateway API v1.5.1), kind node image `kindest/node:v1.35.0`.
- The CloudNativePG `Cluster` pins no `storageClass`, so its PVC falls through to the cluster default — `standard` (kind's built-in local-path provisioner) on kind, `gp3` on EKS.
- The `256Mi` memory limit in `k8s/base/deployment.yaml` is a starting point for a trivial Bun server; confirm it does not `OOMKilled` on the clean-machine pass and bump if `kubectl describe pod` shows it.
- The data-plane NodePort `30080` is the single most environment-sensitive value — confirm reachability on a real cluster (the clean-machine pass is the gate). Pinning the data-plane Pod to the control-plane node (in the same patch) resolves the OrbStack cross-node case, so the one host curl works on both Docker Desktop and OrbStack.
