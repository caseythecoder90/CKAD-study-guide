# Commands reference

> Quick-reference for the kubectl, kind, and shell commands you'll use most. Organized by resource type, with cross-cutting sections (setup, generation, debugging) at the top. For *why* a command works the way it does, see the chapter notes.

## How to use this file

- `Ctrl+F` for the resource (Pods, Deployments, Services, ...) to jump to that section.
- Each resource section follows the same order: **Generate → Inspect → Edit → Delete**.
- Assumes `k=kubectl`, `$do=--dry-run=client -o yaml`, `$now=--force --grace-period=0` are set (chapter 00).
- For deeper per-topic reference, see [`commands/`](commands/) — one focused file per resource. This page is the single-file global view that mirrors what's in there.

---

## 1. Setup at the start of every fresh shell / exam

```bash
# Aliases (the exam pre-sets k=kubectl and bash autocomplete — verify with: alias k)
echo 'export do="--dry-run=client -o yaml"' >> ~/.bashrc
echo 'export now="--force --grace-period=0"' >> ~/.bashrc
source ~/.bashrc

# Vim config for YAML (no tabs, 2-space indent, line numbers)
echo "set expandtab tabstop=2 shiftwidth=2 number" > ~/.vimrc

# Confirm context and namespace before doing anything
kubectl config current-context
kubectl config view --minify | grep namespace
```

---

## 2. Imperative commands & dry-run YAML generation (the exam time-saver)

> Typing YAML by hand is slow. Imperative commands create resources in one line, and `--dry-run=client -o yaml` turns those same commands into a manifest you can edit. Use this pattern for almost every "create a thing" question. Full reference: [`commands/imperative.md`](commands/imperative.md).

### The two imperative entry points

| Command | What it creates |
|---|---|
| `kubectl run` | **Pods only** (since 1.18 — does not create Deployments anymore) |
| `kubectl create` | Everything else: deployments, services (via `expose`), configmaps, secrets, jobs, cronjobs, namespaces, roles, ... |

### Path A — one-shot imperative (no file)

Fastest. Use when every required field has a flag.

```bash
k run nginx --image=nginx
k create deployment web --image=nginx --replicas=3
k create configmap app-cfg --from-literal=ENV=prod
k expose deployment web --port=80 --type=NodePort
k create namespace dev
```

### Path B — generate YAML, edit, apply

When the question needs fields imperative flags don't cover (env vars on a Deployment, probes, volumes, resources, securityContext).

```bash
# 1. Generate template
k <run|create> <args> $do > thing.yaml

# 2. Edit
vim thing.yaml

# 3. Validate (optional but cheap)
k apply -f thing.yaml --dry-run=client
k apply -f thing.yaml --dry-run=server   # stricter, hits the API server

# 4. Apply
k apply -f thing.yaml

# 5. Verify
k get <resource> <name>
k describe <resource> <name>
```

### What `--dry-run=client -o yaml` actually does

- `--dry-run=client` — kubectl builds the spec **without sending it to the API server**. Nothing is created.
- `-o yaml` — print the would-be-sent object as YAML.
- Combined with `> file.yaml`, you get a fully-formed manifest (apiVersion, kind, metadata, spec all populated) ready to edit.

`--dry-run=server` does the same but the API server also validates (catches missing namespaces, unknown fields, admission rejections). Still no persist.

### Exam-time strategy

1. Read the whole question — note namespace, labels, image version, env vars.
2. Pick the entry point: `run` for a Pod, `create <kind>` for everything else, `expose` for a Service.
3. Try **pure imperative first**. Fewer chances to mis-indent YAML.
4. If a required field has no flag, switch to **`$do > file.yaml`**, edit, apply.
5. Always pass `-n <ns>` or set the default namespace first — forgetting this is the most common silly mistake.
6. Verify with `k get` and `k describe` (check the `Events:` section).

---

## 3. Inspecting & debugging (works for any pod)

```bash
# Logs
k logs <pod>
k logs <pod> -f                      # follow
k logs <pod> -c <container>          # multi-container pod
k logs <pod> --previous              # logs from a crashed previous container
k logs -l app=web                    # by label selector

# Shell into a container
k exec -it <pod> -- bash
k exec -it <pod> -c <container> -- sh
k exec <pod> -- <command>            # one-shot, non-interactive

# Describe — Events at the bottom shows scheduling, image pull, probe failures
k describe pod <name>
k describe deployment <name>

# Full live spec (everything Kubernetes stores about the resource)
k get pod <name> -o yaml
k get pod <name> -o yaml > pod.yaml  # extract for editing

# Wide / labeled output
k get pods -o wide                   # adds NODE + IP columns
k get pods --show-labels
k get pods -l app=web                # filter by label

# Watch live
k get pods -w

# Cluster events in chronological order
k get events --sort-by=.metadata.creationTimestamp
```

### Discovery — when you forget a field name or resource type

```bash
k explain pod                        # top-level fields
k explain pod.spec                   # drill in
k explain pod.spec.containers
k explain pod.spec.containers --recursive   # full nested tree

k api-resources                      # every resource type known to the cluster
k api-resources | grep -i ingress    # find a resource by partial name
```

---

## 4. Cluster & context

```bash
# kind (local lab)
kind create cluster --name <name> --config <file>
kind delete cluster --name <name>
kind get clusters

# Switch context / namespace
kubectl config get-contexts
kubectl config use-context <ctx>
kubectl config set-context --current --namespace=<ns>

# Confirm where you are
kubectl config current-context
kubectl config view --minify | grep namespace
```

---

## 5. Pods

```bash
# Generate (kubectl run — pods only)
k run nginx --image=nginx $do > pod.yaml
k run nginx --image=nginx --port=80 $do > pod.yaml
k run nginx --image=nginx --labels="app=web,tier=frontend" $do > pod.yaml
k run nginx --image=nginx --env="ENV=prod" $do > pod.yaml
k run nginx --image=nginx -n <namespace> $do > pod.yaml
k run busybox --image=busybox --command -- sleep 3600 $do > pod.yaml

# Imperative create (no file)
k run nginx --image=nginx
k run nginx --image=nginx --port=80

# Inspect
k get pods
k get pods -A                        # all namespaces
k get pods -o wide
k get pods --show-labels
k get pod <name> -o yaml

# Edit (only certain fields — see "What you can edit" below)
k edit pod <name>

# Extract live spec to a file (when you don't have the original YAML)
k get pod <name> -o yaml > pod.yaml

# Delete
k delete pod <name>                  # by name
k delete -f pod.yaml                 # by manifest
k delete pod -l app=web              # by label selector
k delete pods --all                  # all pods in current namespace
k delete pod <name> $now             # force-delete (only when stuck)
```

**Editable fields on a live pod** (everything else needs delete-and-recreate):

- `spec.containers[*].image`, `spec.initContainers[*].image`
- `spec.activeDeadlineSeconds`
- `spec.tolerations` (additions only)
- `spec.terminationGracePeriodSeconds` (lower only)

---

## 6. Deployments

```bash
# Generate
k create deployment web --image=nginx --replicas=3 $do > deploy.yaml

# Imperative
k create deployment web --image=nginx --replicas=3

# Inspect
k get deployments
k get deploy web -o yaml
k describe deploy web

# Scale
k scale deployment web --replicas=5

# Edit (deployments are fully mutable — controller rolls out new pods)
k edit deployment web
k set image deployment/web nginx=nginx:1.25     # update image only

# Rollout
k rollout status deployment/web
k rollout history deployment/web
k rollout undo deployment/web                   # roll back to previous revision
k rollout undo deployment/web --to-revision=2   # roll back to a specific one
k rollout restart deployment/web                # rolling restart (re-pulls image)

# Delete (removes deployment + replicaset + pods)
k delete deployment web
k delete -f deploy.yaml
```

---

## 7. Services

```bash
# Generate (from an existing deployment or pod)
k expose deployment web --port=80 --target-port=8080 $do > svc.yaml
k expose pod nginx --port=80 --target-port=80 $do > svc.yaml

# Specify a service type
k expose deployment web --port=80 --type=NodePort $do > svc.yaml
k expose deployment web --port=80 --type=LoadBalancer $do > svc.yaml
# (default is ClusterIP if --type is omitted)

# Inspect
k get svc
k get endpoints <svc-name>           # which pods this service points at
k describe svc <name>

# Delete
k delete svc <name>
```

---

## 8. ConfigMaps & Secrets

```bash
# ConfigMap
k create configmap app-config --from-literal=ENV=prod $do > cm.yaml
k create configmap app-config --from-literal=ENV=prod --from-literal=LOG_LEVEL=debug $do > cm.yaml
k create configmap app-config --from-file=config.properties $do > cm.yaml
k create configmap app-config --from-env-file=app.env $do > cm.yaml

# Secret (generic) — values are base64-encoded for you
k create secret generic db-creds --from-literal=username=admin --from-literal=password=s3cr3t $do > secret.yaml
k create secret generic tls-files --from-file=cert.pem --from-file=key.pem $do > secret.yaml

# Secret (TLS-specific shorthand)
k create secret tls my-tls --cert=path/to/cert --key=path/to/key $do > secret.yaml

# Inspect
k get cm
k get cm app-config -o yaml
k get secret
k get secret db-creds -o jsonpath='{.data.password}' | base64 -d   # decode a value
```

---

## 9. Jobs & CronJobs

```bash
# Job (runs once to completion)
k create job pi --image=perl -- perl -Mbignum=bpi -wle "print bpi(100)" $do > job.yaml

# CronJob (recurring; cron-format schedule)
k create cronjob backup --image=busybox --schedule="0 */6 * * *" -- echo "backup" $do > cron.yaml

# Inspect
k get jobs
k get cronjobs                       # short: cj
k logs job/<name>                    # logs from the job's pod

# Trigger a CronJob manually (creates a one-off Job from its template)
k create job --from=cronjob/backup manual-backup
```

---

## 10. Namespaces

```bash
# Create
k create namespace dev
k create namespace dev $do > ns.yaml

# Make a namespace your default for this context (saves typing -n constantly)
k config set-context --current --namespace=dev

# Run a one-off in a different namespace
k get pods -n kube-system
k get pods -A                        # all namespaces

# Delete (removes everything inside the namespace — destructive)
k delete namespace dev
```

---

## 11. Force-delete & common gotchas

```bash
# Pod stuck in Terminating
k delete pod <name> $now             # = --force --grace-period=0

# A deleted pod immediately reappears
# → It's managed by a Deployment/ReplicaSet. Delete the controller, or scale to 0:
k delete deployment <name>
k scale deployment <name> --replicas=0

# "I can't change the image on this pod" / "edit failed"
# → Pods are mostly immutable. Delete and recreate, or edit the parent Deployment.
k get pod <name> -o yaml > pod.yaml
k delete pod <name>
# edit pod.yaml, then:
k apply -f pod.yaml

# Lost track of which context / namespace you're in
k config current-context
k config view --minify | grep namespace
```

---

## 12. Vim quick reference

| Action | Command |
|---|---|
| Open file | `vim file.yaml` |
| Enter Insert mode | `i` |
| Leave Insert mode | `Esc` |
| Save | `:w` |
| Save and quit | `:wq` |
| Quit without saving | `:q!` |
| Jump to line N | `:N` |
| Go to top / bottom | `gg` / `G` |
| Search forward | `/word` then Enter (then `n` for next match) |
| Delete current line | `dd` |
| Copy current line | `yy` |
| Paste below cursor | `p` |
| Undo / Redo | `u` / `Ctrl+r` |

---

## See also

- [`commands/`](commands/) — per-topic focused files (start with [`imperative.md`](commands/imperative.md))
- `00-local-lab-setup.md` — full lab setup, vim survival kit, exam environment
- `03-pods.md` — pod concepts, lifecycle, multi-container patterns
- `04-pod-yaml.md` — YAML structure, real-world walkthrough, template-edit-apply workflow
