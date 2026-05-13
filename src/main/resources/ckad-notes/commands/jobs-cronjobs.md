# Jobs & CronJobs

## Job (runs once to completion)

```bash
k create job pi --image=perl -- perl -Mbignum=bpi -wle "print bpi(100)" $do > job.yaml
k create job pi --image=perl -- perl -Mbignum=bpi -wle "print bpi(100)"     # imperative
```

## CronJob (recurring on a cron schedule)

```bash
k create cronjob backup --image=busybox --schedule="0 */6 * * *" -- echo backup $do > cron.yaml
k create cronjob backup --image=busybox --schedule="*/5 * * * *" -- /backup.sh
```

## Trigger a CronJob manually

Creates a one-off Job from the CronJob's pod template — useful for testing without waiting for the next schedule tick.

```bash
k create job --from=cronjob/backup manual-backup
```

## Inspect

```bash
k get jobs
k get cronjobs                       # short: cj
k describe job <name>
k describe cronjob <name>
k logs job/<name>                    # logs from the pod the job ran
```

## Delete

```bash
k delete job <name>
k delete cronjob <name>              # stops future scheduled runs
```

## Key spec fields

- `spec.completions` — total successful pod completions required.
- `spec.parallelism` — how many pods run in parallel.
- `spec.backoffLimit` — retries before the Job is marked Failed (default 6).
- `spec.activeDeadlineSeconds` — hard timeout for the whole Job.
- `spec.ttlSecondsAfterFinished` — auto-delete completed/failed Jobs after N seconds.

CronJob adds:

- `spec.schedule` — cron expression.
- `spec.concurrencyPolicy` — `Allow` (default), `Forbid`, `Replace`.
- `spec.successfulJobsHistoryLimit` / `failedJobsHistoryLimit` — how many old Jobs to retain.

## See also

- `imperative.md` — full `k create job` / `k create cronjob` syntax
- `debugging.md` — `k logs job/<name>` and reading describe output
