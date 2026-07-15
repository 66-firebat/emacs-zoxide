# PATCH 1-0-3 — Conditional zoxide travel dispatch

## Goal

Bind `Alt + z` to a single command that intelligently dispatches between the two zoxide-travel variants based on the current buffer type:

| Current buffer | Command |
|----------------|---------|
| `eat-mode` (eat terminal) | `eat-zoxide-travel` — cd into selected directory |
| Any other buffer | `grease-zoxide-travel` — open directory in Grease file manager |

## Implementation

### New dispatch function

A small wrapper in `keybinds.el` that checks the current major mode:

```elisp
(defun my/zoxide-travel-dispatch ()
  "Dispatch to `eat-zoxide-travel' or `grease-zoxide-travel' based on context.
In an eat terminal buffer, cd into the selected directory.
Otherwise, open the directory in Grease."
  (interactive)
  (if (derived-mode-p 'eat-mode)
      (call-interactively #'eat-zoxide-travel)
    (call-interactively #'grease-zoxide-travel)))
```

### Keybinding

Replace the current `Alt+g` → `grease-zoxide-travel` in `keybinds.el`:

| Key | Command |
|-----|---------|
| `Alt + z` | `my/zoxide-travel-dispatch` |
| `Alt + g` | (free — can be reused for something else) |

```elisp
(general-def :keymaps 'override
  ...
  "M-z" 'my/zoxide-travel-dispatch)
```

### Eat non-bound key consideration

`Alt+z` is not currently in the eat semi-char non-bound keys list. It needs to be added so it works from inside eat terminal buffers — otherwise eat will intercept it and send it to the shell.

```elisp
                 ("M-z" . [?\e ?z])
```

### Edge cases

| Scenario | Behavior |
|----------|----------|
| Eat buffer, no `zoxide` installed | Falls through to `completing-read` fallback, errors gracefully |
| Eat buffer, consult not installed | Falls through to `completing-read` fallback, then attempts eat cd |
| Non-eat buffer, same as before | Opens Grease as always |
| User invokes via `M-x` directly | Both commands still accessible individually via `M-x grease-zoxide-travel` / `M-x eat-zoxide-travel` |
| Eat buffer, `eat--send-string` not available | Errors with "Not in an eat terminal buffer" (handled by `eat-zoxide-travel`) |

## Migration

1. Add `my/zoxide-travel-dispatch` to `keybinds.el`
2. Change `"M-g" 'grease-zoxide-travel)` to `"M-z" 'my/zoxide-travel-dispatch)`
3. Add `("M-z" . [?\e ?z])` to the eat non-bound keys `dolist`
4. Remove the old `"M-g"` binding (or reassign it)
