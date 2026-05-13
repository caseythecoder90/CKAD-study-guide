# Deployments

## Generate

```bash
k create deployment web --image=nginx --replicas=3 $do > deploy.yaml
k create deployment web --image=nginx --port=80 $do > deploy.yaml
```

## Imperative create

```bash
k create deployment web --image=nginx
k create deployment web --image=nginx --replicas=3
```

## Inspect

```bash
k get deployments              # short: deploy
k get deploy web -o yaml
k describe deploy web
k get rs                       # the ReplicaSets owned by Deployments
k get pods -l app=web          # pods this Deployment is managing
```

## Scale

```bash
k scale deployment web --replicas=5
k scale deployment web --replicas=0     # drain to zero without deleting
```

## Edit

Deployments are **fully mutable** — the controller rolls out new pods on change.

```bash
k edit deployment web
k set image deployment/web nginx=nginx:1.25      # update just the image
k set env deployment/web ENV=staging             # update an env var
k set resources deployment/web --limits=cpu=200m,memory=256Mi
```

## Rollout

```bash
k rollout status deployment/web                  # wait/show progress
k rollout history deployment/web                 # list revisions
k rollout history deployment/web --revision=3    # detail a revision
k rollout undo deployment/web                    # roll back to previous
k rollout undo deployment/web --to-revision=2    # roll back to a specific revision
k rollout restart deployment/web                 # rolling restart (re-pulls image)
k rollout pause deployment/web                   # pause rollout mid-update
k rollout resume deployment/web                  # resume
```

## Delete

Removes the Deployment, its ReplicaSets, and the Pods underneath.

```bash
k delete deployment web
k delete -f deploy.yaml
```

## See also

- `imperative.md` — full imperative flag list
- `services.md` — `k expose deployment` to put a Service in front of it
- `pods.md` — inspecting the pods a Deployment owns
