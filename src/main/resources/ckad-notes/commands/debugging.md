# Debugging & inspection

Works for any pod/resource. This is the workflow when something is "not working" and you need to figure out why.

## Logs

```bash
k logs <pod>
k logs <pod> -f                      # follow
k logs <pod> -c <container>          # multi-container pod
k logs <pod> --previous              # logs from the crashed previous container
k logs -l app=web                    # by label selector
k logs -l app=web --tail=50 --max-log-requests=10
k logs deployment/web                # logs from one pod the deployment owns
```

## Shell / one-off command in a container

```bash
k exec -it <pod> -- bash
k exec -it <pod> -c <container> -- sh
k exec <pod> -- <command>            # one-shot, non-interactive
k exec <pod> -- env                  # see the env the container actually has
```

## Describe — read the Events at the bottom

`Events:` shows scheduling decisions, image pull failures, probe failures, OOM kills.

```bash
k describe pod <name>
k describe deployment <name>
k describe svc <name>
k describe node <name>               # for "why isn't my pod scheduling?"
```

## Full live spec

```bash
k get pod <name> -o yaml
k get pod <name> -o yaml > pod.yaml  # extract for editing
k get pod <name> -o jsonpath='{.status.podIP}'
```

## Wider listings

```bash
k get pods -o wide                   # adds NODE + IP columns
k get pods --show-labels
k get pods -l app=web                # filter by label
k get pods -w                        # watch live
k get pods --sort-by=.status.startTime
```

## Cluster events (often the fastest way to diagnose)

```bash
k get events --sort-by=.metadata.creationTimestamp
k get events -n <ns> --sort-by=.lastTimestamp
k get events --field-selector type=Warning
```

## Discovery — when you forget a field name or resource type

```bash
k explain pod                        # top-level fields
k explain pod.spec                   # drill in
k explain pod.spec.containers
k explain pod.spec.containers --recursive    # full nested tree
k explain pod.spec.containers.env

k api-resources                      # every resource known to the cluster
k api-resources | grep -i ingress    # find a resource by partial name
k api-resources --namespaced=true    # namespaced resources only
```

## Common "why is X broken?" recipes

| Symptom | First thing to run |
|---|---|
| Pod stuck `Pending` | `k describe pod <name>` — Events show why it can't schedule |
| Pod `ImagePullBackOff` | `k describe pod <name>` — typo in image name, missing pull secret |
| Pod `CrashLoopBackOff` | `k logs <pod> --previous` — see what the previous container said before dying |
| Service has no endpoints | `k get endpoints <svc>` then `k describe svc <svc>` — selector doesn't match pod labels |
| `k edit` rejects a field | the field is immutable on this resource — extract YAML, delete, re-apply |
| Pod stuck `Terminating` | `k delete pod <name> $now` (force) — but investigate finalizers first |

## See also

- `pods.md` — what fields are mutable on a Pod
- `imperative.md` — generating a fresh YAML when delete-and-recreate is needed
