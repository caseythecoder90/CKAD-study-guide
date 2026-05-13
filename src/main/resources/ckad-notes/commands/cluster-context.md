# Cluster & context

## `kind` (local lab)

```bash
kind create cluster --name <name> --config <file>
kind delete cluster --name <name>
kind get clusters
```

## Contexts

```bash
kubectl config get-contexts
kubectl config use-context <ctx>
kubectl config current-context
```

## Namespace as default for a context

Avoids typing `-n <ns>` on every command.

```bash
kubectl config set-context --current --namespace=<ns>
```

## Confirm where you are

```bash
kubectl config current-context
kubectl config view --minify | grep namespace
```

## See also

- `setup.md` — `.bashrc` / `.vimrc` start-of-shell setup
- `namespaces.md` — creating and deleting namespaces
