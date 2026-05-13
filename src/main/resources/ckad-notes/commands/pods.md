# Pods

## Generate (template-then-edit)

```bash
k run nginx --image=nginx $do > pod.yaml
k run nginx --image=nginx --port=80 $do > pod.yaml
k run nginx --image=nginx --labels="app=web,tier=frontend" $do > pod.yaml
k run nginx --image=nginx --env="ENV=prod" $do > pod.yaml
k run nginx --image=nginx -n <namespace> $do > pod.yaml
k run busybox --image=busybox --command -- sleep 3600 $do > pod.yaml
```

## Imperative create (no file)

```bash
k run nginx --image=nginx
k run nginx --image=nginx --port=80
k run nginx --image=nginx --restart=Never               # explicit — stays a Pod
k run tmp --image=busybox --rm -it --restart=Never -- sh    # ephemeral debug shell
```

See [`imperative.md`](imperative.md) for the full imperative strategy.

## Inspect

```bash
k get pods
k get pods -A                        # all namespaces
k get pods -o wide                   # adds NODE + IP columns
k get pods --show-labels
k get pods -l app=web                # filter by label
k get pod <name> -o yaml             # full live spec
k describe pod <name>                # human summary; Events at the bottom
k get pods -w                        # watch live
```

## Edit

```bash
k edit pod <name>
```

**What you can actually edit on a live Pod** (everything else needs delete-and-recreate, or edit the parent Deployment):

- `spec.containers[*].image`, `spec.initContainers[*].image`
- `spec.activeDeadlineSeconds`
- `spec.tolerations` (additions only)
- `spec.terminationGracePeriodSeconds` (lower only)

When you need to change something not in that list:

```bash
k get pod <name> -o yaml > pod.yaml
k delete pod <name>
# edit pod.yaml, then:
k apply -f pod.yaml
```

## Delete

```bash
k delete pod <name>                  # by name
k delete -f pod.yaml                 # by manifest
k delete pod -l app=web              # by label selector
k delete pods --all                  # all pods in current namespace
k delete pod <name> $now             # force-delete (only when stuck Terminating)
```

## Gotchas

- A deleted Pod immediately reappears → it's managed by a Deployment/ReplicaSet. Delete the controller or scale to 0:
  ```bash
  k delete deployment <name>
  k scale deployment <name> --replicas=0
  ```
- Pods are mostly immutable. If `k edit` rejects your change, you need delete-and-recreate.

## See also

- `imperative.md` — `k run` flags + dry-run YAML generation
- `deployments.md` — managing Pods through a controller
- `debugging.md` — logs, exec, describe, events
