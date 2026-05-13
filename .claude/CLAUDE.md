# Project context for Claude

## What this repo is

CKAD (Certified Kubernetes Application Developer) study notes and lab setup.

- Notes live in `src/main/resources/ckad-notes/`, numbered by chapter (`00-...`, `01-...`).
- Diagrams referenced from notes live in `src/main/resources/ckad-notes/diagrams/`, numbered.
- `src/main/resources/ckad-notes/commands.md` is the **global single-file command reference** (Ctrl+F across everything).
- `src/main/resources/ckad-notes/commands/` holds **per-topic focused command files** for quick navigation while studying one resource at a time.

---

## Commit & PR workflow (required for every change)

Every change — even a small notes edit — goes through this flow. Don't commit directly to `main`.

1. **Open a GitHub issue** describing what's being added or changed.
   ```bash
   gh issue create --title "Add notes for <topic>" --body "Covers <what>. Includes diagrams <N>-<M>."
   ```
   Capture the issue number from the output.

2. **Create a feature branch** off `main`:
   ```bash
   git switch -c notes/<chapter-slug>      # for new chapter notes
   git switch -c docs/<short-slug>         # for commands or meta docs
   git switch -c fix/<short-slug>          # for corrections
   ```

3. **Commit** on the branch with a clear, conventional message:
   ```bash
   git add <files>
   git commit -m "Add <topic> notes (chapters NN-MM)"
   ```
   No emojis. No co-author trailer unless explicitly asked. One concise subject line; longer body only if the change needs explanation.

4. **Push the branch** to `origin`:
   ```bash
   git push -u origin HEAD
   ```

5. **Open a PR** with `gh pr create`. The PR body **must** include a GitHub auto-close keyword referencing the issue, so merging the PR closes the issue automatically:
   ```bash
   gh pr create --title "Add <topic> notes" --body "$(cat <<'EOF'
   ## Summary
   - <what changed>

   Closes #<issue-number>
   EOF
   )"
   ```
   Auto-close keywords (any work, case-insensitive): `Closes #N`, `Fixes #N`, `Resolves #N`. The reference must be in the PR description or in a commit message that lands on the default branch.

6. **Merge the PR** (`gh pr merge --squash --delete-branch` is the default). The linked issue closes automatically when the merge reaches `main`.

### Quick sanity check before merging
- Issue linked in PR body via `Closes #N`? ✓
- Branch is feature branch, not `main`? ✓
- Commit message is informative (no "wip", no "update")? ✓

---

## Commands documentation rules

When a change introduces or surfaces new `kubectl`/`kind`/shell commands:

- **Always update `commands.md`** (the global reference). New commands belong in the relevant numbered section.
- **Always update the per-topic file** in `commands/` for the resource(s) touched.
- The dedicated **`commands/imperative.md`** is the exam-time reference for imperative one-liners + `--dry-run=client -o yaml` YAML generation — keep its examples in sync when new resources are covered.

If a notes chapter introduces a new resource type that has no focused command file yet, create one in `commands/` and add it to `commands/README.md` index.

---

## Style

- No emojis in notes or commit messages unless the user explicitly asks for them.
- Concise, exam-focused phrasing. No marketing fluff, no padding.
- Diagrams referenced by their numbered filename prefix from `diagrams/`.
- Prefer editing existing files over creating new ones; keep new files focused (one topic per file).
- When updating notes, also update `commands.md` and the matching `commands/<topic>.md` in the same PR so the references never drift.