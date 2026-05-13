# Imperative commands & dry-run YAML generation

> **Why this matters on the exam.** Hand-typing YAML is slow and easy to get wrong. Imperative commands create resources in one line, and `--dry-run=client -o yaml` turns those same commands into a working manifest you can edit. This pattern is the single biggest time-saver on the CKAD.

---

## The two imperative entry points

| Command | What it creates |
|---|---|
| `kubectl run` | **Pods only** (since 1.18 — it no longer creates Deployments) |
| `kubectl create` | Everything else: deployments, services (via `expose`), configmaps, secrets, jobs, cronjobs, namespaces, roles, ... |

`kubectl expose` is a third entry point, but it only creates a **Service** from an existing pod/deployment.

---

## Two ways to use imperative

### 1. One-shot create (no file)

Fastest when every required field has a flag. Use it for throwaway resources, or when the question doesn't ask for anything beyond image/port/labels/replicas.

```bash
k run nginx --image=nginx
k create deployment web --image=nginx --replicas=3
k create configmap app-cfg --from-literal=ENV=prod
k expose deployment web --port=80 --type=NodePort
```

### 2. Generate YAML, edit, apply (the time-saver)

When the question needs fields the imperative flags don't cover — env vars on a Deployment, volume mounts, probes, resource limits, securityContext — generate a starter file and edit just the bits you need.

```bash
# 1. Generate
k run nginx --image=nginx --port=80 $do > pod.yaml

# 2. Edit
vim pod.yaml      # add volumeMounts, probes, env, resources, ...

# 3. (Optional) validate before applying
k apply -f pod.yaml --dry-run=client     # client-side schema check
k apply -f pod.yaml --dry-run=server     # API-server validation, still no persist

# 4. Apply
k apply -f pod.yaml

# 5. Verify
k get pod nginx
k describe pod nginx
```

`$do` is the alias `--dry-run=client -o yaml`. Set it once at the start of the exam (see `setup.md`).

---

## What `--dry-run=client -o yaml` actually does

- `--dry-run=client` — the **kubectl client** builds the object spec and **does not send it to the API server**. Nothing is created.
- `-o yaml` — print the would-be-sent object as YAML instead of a confirmation line.
- `> file.yaml` — redirect the YAML into a file you can edit and `apply` later.

The output is a fully-formed manifest: `apiVersion`, `kind`, `metadata`, `spec` are all populated with the correct values for that resource. You skip all the boilerplate.

> `--dry-run=server` does the same thing but also runs the spec through the API server's validation (catches missing namespaces, unknown fields, admission webhook rejections). Still doesn't persist. Useful when client-side check passes but `apply` mysteriously fails.

---

## Imperative cheat sheet by resource

### Pods (`k run`)

```bash
k run nginx --image=nginx
k run nginx --image=nginx --port=80
k run nginx --image=nginx --env="ENV=prod" --env="LOG=debug"
k run nginx --image=nginx --labels="app=web,tier=frontend"
k run nginx --image=nginx -n dev
k run busybox --image=busybox --command -- sleep 3600
k run nginx --image=nginx --restart=Never              # explicit — stays a Pod
k run tmp --image=busybox --rm -it --restart=Never -- sh   # debug shell, auto-deletes on exit
```

### Deployments

```bash
k create deployment web --image=nginx
k create deployment web --image=nginx --replicas=3
k create deployment web --image=nginx --port=80
k create deployment web --image=nginx -- /bin/sh -c 'while true; do echo hi; sleep 5; done'
```

### Services — almost always via `expose`

`expose` requires the **target resource to already exist**; it reads selector labels from the deployment/pod and builds the Service from them.

```bash
k expose deployment web --port=80 --target-port=8080            # ClusterIP (default)
k expose deployment web --port=80 --type=NodePort
k expose deployment web --port=80 --type=LoadBalancer
k expose pod nginx --port=80 --name=nginx-svc
```

If you need a Service without a backing workload (rare on CKAD):

```bash
k create service clusterip my-svc --tcp=80:8080
k create service nodeport my-svc --tcp=80:8080 --node-port=30080
```

### ConfigMaps

```bash
k create configmap app-cfg --from-literal=ENV=prod
k create configmap app-cfg --from-literal=ENV=prod --from-literal=LOG_LEVEL=debug
k create configmap app-cfg --from-file=config.properties
k create configmap app-cfg --from-file=app.conf=./local-file.conf       # rename in the CM
k create configmap app-cfg --from-env-file=app.env
```

### Secrets

```bash
k create secret generic db --from-literal=user=admin --from-literal=pass=s3cr3t
k create secret generic tls-files --from-file=cert.pem --from-file=key.pem
k create secret tls my-tls --cert=path/to/cert --key=path/to/key
k create secret docker-registry regcred \
  --docker-username=alice --docker-password=p4ss --docker-email=a@b.c
```

### Jobs & CronJobs

```bash
k create job pi --image=perl -- perl -Mbignum=bpi -wle "print bpi(100)"
k create cronjob backup --image=busybox --schedule="0 */6 * * *" -- echo backup

# Manually trigger a CronJob's pod template as a one-off Job
k create job --from=cronjob/backup manual-backup
```

### Namespaces

```bash
k create namespace dev
```

### RBAC (less common on CKAD but supported)

```bash
k create role pod-reader --verb=get,list,watch --resource=pods
k create rolebinding alice-pods --role=pod-reader --user=alice
k create clusterrole node-reader --verb=get,list --resource=nodes
k create clusterrolebinding alice-nodes --clusterrole=node-reader --user=alice
```

### ServiceAccount

```bash
k create serviceaccount build-bot
```

---

## Exam-time strategy

1. **Read the whole question first.** Note: namespace, labels required, image (and version!), env vars, ports, replicas.
2. **Pick the right entry point.**
   - Pod → `kubectl run`
   - Anything else → `kubectl create <kind>`
   - Service → `kubectl expose <existing-thing>`
3. **Try pure imperative first.** If every required field has a flag, skip YAML entirely — fewer chances to mis-indent.
4. **If imperative can't express a field**, generate with `$do > thing.yaml`, edit, `apply`.
5. **Always set the namespace** on the command (`-n <ns>`) or switch context first (`k config set-context --current --namespace=<ns>`). Forgetting `-n` is the most common silly mistake.
6. **Verify** every resource with `k get` and `k describe` before moving on. Check `Events:` at the bottom of `describe` for image pull / scheduling issues.

---

## Common pitfalls

- `kubectl run` does **not** create a Deployment anymore. It creates a single Pod. Use `kubectl create deployment` if you need a Deployment.
- Bare `--dry-run` is deprecated. Always specify `--dry-run=client` or `--dry-run=server`.
- `kubectl expose` reads selector labels from the target. If the target has no labels (or the wrong ones), the Service will have no endpoints.
- `--from-literal` can be repeated; you don't need separate `create configmap` calls per key.
- `$do` is **not a default alias** in the exam — you set it yourself in `~/.bashrc` at the start. See `setup.md`.
- When generating a YAML for a Deployment and then editing replicas/image, remember to also fix `metadata.name` if you copy-paste the file as a template for a second resource.

---

## See also

- `setup.md` — set up `$do`, `$now`, vim, aliases at the start of the exam
- `pods.md`, `deployments.md`, `services.md`, ... — full per-resource reference
- `../commands.md` — global single-file reference
