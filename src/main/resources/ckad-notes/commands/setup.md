# Setup — start-of-shell / start-of-exam

Run these once at the start of every fresh shell (and at the very start of the exam) so the rest of the commands in these notes work as written.

## Aliases & env vars

```bash
# Confirm the exam-provided aliases are already set
alias k          # should show: alias k='kubectl'
# bash autocomplete on `k` is also pre-configured

# Add the two time-saver env vars
echo 'export do="--dry-run=client -o yaml"' >> ~/.bashrc
echo 'export now="--force --grace-period=0"' >> ~/.bashrc
source ~/.bashrc
```

Usage:

```bash
k run nginx --image=nginx $do > pod.yaml      # generate template
k delete pod stuck-pod $now                   # force-delete
```

## Vim config for YAML

YAML hates tabs. This `.vimrc` enforces 2-space indents and shows line numbers (useful when the exam tells you "the error is on line 23").

```bash
echo "set expandtab tabstop=2 shiftwidth=2 number" > ~/.vimrc
```

## Confirm where you are before doing anything

Two questions to answer before every task:

1. **Which cluster / context am I in?**
2. **Which namespace will my commands hit?**

```bash
kubectl config current-context
kubectl config view --minify | grep namespace
```

If the question says "in namespace `foo`", either:

```bash
# Set it as the default for this context (preferred when many commands follow)
k config set-context --current --namespace=foo

# Or pass -n foo on every command
k get pods -n foo
```

## See also

- `cluster-context.md` — switching contexts, `kind` lab clusters
- `imperative.md` — what to do with `$do` once it's set
- `vim.md` — vim keystroke cheat sheet for the exam
