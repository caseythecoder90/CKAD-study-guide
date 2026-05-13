# Services

`kubectl expose` is the imperative path for Services and is almost always what you want — it reads the selector labels from an existing Deployment/Pod, so the Service automatically targets the right backends.

## Generate

```bash
# From a Deployment
k expose deployment web --port=80 --target-port=8080 $do > svc.yaml

# From a Pod
k expose pod nginx --port=80 --target-port=80 $do > svc.yaml
```

## Service type

Default is `ClusterIP` when `--type` is omitted.

```bash
k expose deployment web --port=80 --type=ClusterIP $do > svc.yaml
k expose deployment web --port=80 --type=NodePort $do > svc.yaml
k expose deployment web --port=80 --type=LoadBalancer $do > svc.yaml
```

## When you have no backing workload (rare on CKAD)

```bash
k create service clusterip my-svc --tcp=80:8080
k create service nodeport my-svc --tcp=80:8080 --node-port=30080
k create service loadbalancer my-svc --tcp=80:8080
k create service externalname my-svc --external-name=example.com
```

## Inspect

```bash
k get svc
k get svc -o wide
k get endpoints <svc-name>             # which pods this Service actually points at
k describe svc <name>
```

If `endpoints` shows `<none>`, the Service's selector doesn't match any Pod's labels — the most common cause of "my Service doesn't work."

## Delete

```bash
k delete svc <name>
k delete -f svc.yaml
```

## NodePort gotchas

- NodePort range is `30000–32767`. Outside that range, the API server rejects the manifest.
- A NodePort Service also has a ClusterIP — you don't lose the in-cluster DNS entry.
- The port is open on **every node**, not just the node running the pod. kube-proxy routes the traffic.

## ClusterIP vs NodePort

| | ClusterIP | NodePort |
|---|---|---|
| Reachable from inside the cluster | yes (via Service DNS / ClusterIP) | yes |
| Reachable from outside | no | yes — `<NodeIP>:<NodePort>` |
| Default | ✓ | — |
| Use case | service-to-service traffic | dev/test external exposure, behind an Ingress |

## See also

- `imperative.md` — `k expose` and `k create service` patterns
- `deployments.md` — the typical backend for a Service
- `08-services-nodeport.md`, `09-services-clusterip.md` — concept chapters
