# Namespaces

## Create

```bash
k create namespace dev
k create namespace dev $do > ns.yaml      # template
```

## Set as default for the current context

```bash
k config set-context --current --namespace=dev
```

After this, every command runs against `dev` unless overridden with `-n`.

## Cross-namespace queries

```bash
k get pods -n kube-system
k get pods -A                              # all namespaces
k get pods --all-namespaces                # same thing, long form
```

## Inspect

```bash
k get namespaces                           # short: ns
k describe ns dev
```

## Delete

Removes the namespace and **every resource inside it**. Destructive — confirm before running.

```bash
k delete namespace dev
```

If a namespace is stuck in `Terminating`, it's usually a finalizer on a contained resource — investigate `k get ns dev -o yaml` and `k api-resources --namespaced=true -o name | xargs -n1 k get -n dev` rather than force-deleting.

## See also

- `cluster-context.md` — switching the default namespace per context
- `07-namespaces.md` — concept chapter on namespace types, DNS naming, resource quotas
