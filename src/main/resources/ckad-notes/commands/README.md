# Commands — per-topic reference

This directory splits the global `../commands.md` into focused files, one per resource type or task area. Use this when you're studying or drilling a single topic; use `../commands.md` when you want a single page to Ctrl+F across everything.

Both stay in sync — when a notes chapter adds new commands, update **both** the relevant file here and the global reference.

## Index

| File | What's in it |
|---|---|
| [`imperative.md`](imperative.md) | **The exam time-saver.** Imperative one-liners + `--dry-run=client -o yaml` to generate manifests. Read this first. |
| [`setup.md`](setup.md) | Aliases (`k`, `$do`, `$now`), `.bashrc`, vim config, context/namespace verification |
| [`cluster-context.md`](cluster-context.md) | `kind` clusters, contexts, switching namespace defaults |
| [`pods.md`](pods.md) | Pod create/inspect/edit/delete, which fields are mutable |
| [`deployments.md`](deployments.md) | Deployment generation, scaling, rollout, rollback, restart |
| [`services.md`](services.md) | `expose`, ClusterIP vs NodePort, endpoints |
| [`namespaces.md`](namespaces.md) | Create, default-switch, cross-namespace queries, delete |
| [`configmaps-secrets.md`](configmaps-secrets.md) | All `--from-*` variants, TLS shorthand, decoding |
| [`jobs-cronjobs.md`](jobs-cronjobs.md) | Job, CronJob, manual trigger from a CronJob |
| [`debugging.md`](debugging.md) | `logs`, `exec`, `describe`, `events`, `explain`, `api-resources` |
| [`vim.md`](vim.md) | Vim survival kit for editing YAML on the exam |

## Conventions

All files assume the aliases from `setup.md` are sourced:

- `k` = `kubectl`
- `$do` = `--dry-run=client -o yaml`
- `$now` = `--force --grace-period=0`
