# 10 — Formatting kubectl output

> An aside, but one of the highest-leverage things you can learn for the exam. Every `kubectl get`, `describe`, and most other read commands accept an output flag — `-o` (or `--output`) — that completely changes what comes back. The default human-readable table is fine for browsing, but on the exam you'll want JSON/YAML to extract a single field, JSONPath to script a check, or `wide` to see which node a pod landed on. Knowing the right format saves seconds, and seconds add up.

---

## 1. The flag

```bash
kubectl get <resource> [name] -o <format>
kubectl get <resource> [name] --output <format>
```

`-o` is the short form. They're identical.

---

## 2. The main formats

### `default` (table) — no `-o` at all

The human-readable summary table. Columns differ per resource. Good for browsing, useless for scripting.

```bash
k get pods
# NAME    READY   STATUS    RESTARTS   AGE
# web-0   1/1     Running   0          3m
```

### `-o wide` — table plus extra columns

Adds the most-commonly-needed extra info. For pods that means `NODE`, `IP`, `NOMINATED NODE`, `READINESS GATES`. For services, `SELECTOR`. For nodes, `INTERNAL-IP`, `OS-IMAGE`, `KERNEL-VERSION`, `CONTAINER-RUNTIME`.

```bash
k get pods -o wide
# NAME    READY   STATUS    RESTARTS   AGE   IP           NODE       ...
# web-0   1/1     Running   0          3m    10.244.1.5   worker-2   ...

k get svc -o wide
# Adds the SELECTOR column — instantly shows which pod labels a Service targets
```

When you're debugging "which node is this pod on?" or "what's the pod IP?", `-o wide` answers it without dropping to `-o yaml`.

### `-o yaml` — full live spec as YAML

The complete object including the `status` subresource and Kubernetes-added fields (UID, resourceVersion, managedFields, etc.). This is what you want for **extracting a working manifest from a live resource**.

```bash
k get pod web-0 -o yaml
k get pod web-0 -o yaml > pod.yaml         # save for editing
k get deploy web -o yaml | less
```

### `-o json` — full live spec as JSON

Same content as YAML, JSON-structured. Use this when you need to pipe into `jq` for non-trivial filtering — `jq` is more flexible than `jsonpath`.

```bash
k get pod web-0 -o json
k get pods -o json | jq '.items[].metadata.name'
k get pods -o json | jq '.items[] | select(.status.phase=="Running") | .metadata.name'
```

### `-o name` — just `<kind>/<name>`

One line per resource, in `kind/name` form. Perfect for piping into `xargs` or another `kubectl` command.

```bash
k get pods -o name
# pod/web-0
# pod/web-1

# Delete every pod with a label
k get pods -l env=test -o name | xargs k delete
```

### `-o jsonpath='<expr>'` — one or more fields via JSONPath

The exam favourite. Pulls out exactly the field(s) you want, no jq required. Wrap the expression in single quotes so the shell doesn't expand `$`.

```bash
# Single value
k get pod web-0 -o jsonpath='{.status.podIP}'
k get svc web -o jsonpath='{.spec.clusterIP}'
k get node worker-2 -o jsonpath='{.status.capacity.cpu}'

# Decode a Secret value (base64 round-trip)
k get secret db-creds -o jsonpath='{.data.password}' | base64 -d

# Iterate over a list with {range ...}{end}
k get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Every container image across every pod (a classic exam-ish task)
k get pods -A -o jsonpath='{.items[*].spec.containers[*].image}'
```

JSONPath inside kubectl is a **subset of standard JSONPath** with kubectl-specific quirks (no `?()` filter expression, no recursive descent in some versions). For anything complex, use `-o json | jq`.

### `-o jsonpath-file=<file>` — JSONPath expression from a file

Same as `-o jsonpath` but reads the expression from a file. Useful when the expression is long.

```bash
echo '{.items[*].metadata.name}' > query.jsonpath
k get pods -o jsonpath-file=query.jsonpath
```

### `-o go-template='<tmpl>'` — Go templates

More powerful than JSONPath (real conditionals and loops), but more verbose. Rarely needed on the exam — reach for `jq` first.

```bash
k get pods -o go-template='{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}'
```

### `-o go-template-file=<file>`

Same as above, template loaded from a file.

### `-o custom-columns=<spec>` — custom table

Define your own table columns from JSONPath expressions. Great when `-o wide` doesn't show what you need but you don't want full YAML.

```bash
k get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName

# Header-free (good for scripting)
k get pods -o custom-columns=NAME:.metadata.name --no-headers
```

### `-o custom-columns-file=<file>`

Column spec loaded from a file:

```
NAME           NODE              IP
.metadata.name .spec.nodeName    .status.podIP
```

```bash
k get pods -o custom-columns-file=cols.txt
```

---

## 3. Modifiers that pair with the format flags

These aren't `-o` formats themselves, but they shape what shows up in the table or YAML.

### `--show-labels` — append a `LABELS` column

```bash
k get pods --show-labels
```

### `-L <label>[,<label>...]` — show specific label values as columns

```bash
k get pods -L app,tier
# NAME    READY   STATUS    AGE    APP   TIER
# web-0   1/1     Running   3m     web   frontend
```

### `--sort-by=<jsonpath>` — sort rows by a field

```bash
k get pods --sort-by=.metadata.creationTimestamp
k get pods --sort-by=.spec.nodeName
k get events --sort-by=.lastTimestamp
```

### `--no-headers` — drop the header row (for scripts)

```bash
k get pods --no-headers -o custom-columns=NAME:.metadata.name
```

### `-w` / `--watch` — stream changes after the initial list

```bash
k get pods -w
k get pods -o wide -w
```

### Filters that change the result set (not the format)

```bash
k get pods -l app=web                       # label selector
k get pods --field-selector status.phase=Running
k get pods -A                                # all namespaces
k get pods -n <ns>                           # one specific namespace
```

These compose with the `-o` flag — `k get pods -A -o wide -l app=web` is fine.

---

## 4. Exam-time patterns worth memorising

```bash
# "What IP is this pod on?"
k get pod <name> -o jsonpath='{.status.podIP}'

# "Which node is this pod on?"
k get pod <name> -o jsonpath='{.spec.nodeName}'
# or just
k get pod <name> -o wide

# "Decode the password from this Secret"
k get secret <name> -o jsonpath='{.data.password}' | base64 -d

# "Show me every image running anywhere"
k get pods -A -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u

# "Extract a live resource's spec to a file so I can edit it"
k get pod <name> -o yaml > pod.yaml

# "Sort pods by age, newest last"
k get pods --sort-by=.metadata.creationTimestamp

# "Watch a rollout in real time" (alternative to k rollout status)
k get pods -l app=web -w

# "Just the names, for scripting"
k get pods -l env=test -o name
```

---

## 5. Gotchas

- **Single-quote JSONPath expressions.** Bash will eat `$` and `{}` otherwise.
- `k describe` does **not** accept `-o`. It's a fixed human-readable format. For machine-readable detail use `-o yaml` or `-o json`.
- `-o yaml` includes `status` and `managedFields` — when you save the output to recreate the resource elsewhere, **strip those** (or use `--show-managed-fields=false`, available on recent kubectl) before re-applying.
- `--dry-run=client -o yaml` is the same `-o yaml` you use here, just on a *would-be* object instead of a live one. The output format machinery is shared. See `commands/imperative.md`.
- JSONPath in kubectl is missing some features of "real" JSONPath. If `?(@.field=="x")` filtering doesn't work, fall through to `-o json | jq`.

---

## 6. References

- **kubectl reference (the big one)** — <https://kubernetes.io/docs/reference/kubectl/>
- **kubectl quick reference / cheat sheet** — <https://kubernetes.io/docs/reference/kubectl/quick-reference/>
- **JSONPath support in kubectl** — <https://kubernetes.io/docs/reference/kubectl/jsonpath/>
- **`kubectl get` command spec** — <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/>
- **List all container images running in a cluster** (worked JSONPath example) — <https://kubernetes.io/docs/tasks/access-application-cluster/list-all-running-container-images/>
- **Go templates (text/template package)** — <https://pkg.go.dev/text/template>

---

## See also

- `commands.md` — global commands reference
- `commands/debugging.md` — `logs`, `exec`, `describe`, `explain`, `api-resources`
- `commands/imperative.md` — `--dry-run=client -o yaml` for generating manifests (the most common use of `-o yaml`)
