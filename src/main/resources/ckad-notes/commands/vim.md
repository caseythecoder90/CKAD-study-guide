# Vim quick reference

You'll be in vim a lot on the exam — to edit generated YAML. These are the keystrokes that matter.

| Action | Command |
|---|---|
| Open file | `vim file.yaml` |
| Enter Insert mode | `i` |
| Leave Insert mode | `Esc` |
| Save | `:w` |
| Save and quit | `:wq` |
| Quit without saving | `:q!` |
| Jump to line N | `:N` |
| Go to top / bottom | `gg` / `G` |
| Search forward | `/word` then Enter (`n` for next, `N` for previous) |
| Delete current line | `dd` |
| Copy (yank) current line | `yy` |
| Paste below cursor | `p` |
| Undo / Redo | `u` / `Ctrl+r` |
| Indent / outdent line | `>>` / `<<` |
| Visual select line | `V` then move cursor |
| Replace word under cursor | `cw` then type new word |

## Recommended `~/.vimrc` for the exam

```vim
set expandtab tabstop=2 shiftwidth=2 number
```

- `expandtab` — Tab key inserts spaces (YAML rejects tabs).
- `tabstop=2 shiftwidth=2` — 2-space indents, the Kubernetes YAML convention.
- `number` — show line numbers (so error messages like "line 23" are useful).

See `setup.md` for the one-liner to write this file at the start of the exam.
