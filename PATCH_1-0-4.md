# PATCH 1-0-4 — Embark actions for zoxide directories

## Goal

Add embark keybindings `+` and `-` inside the zoxide consult minibuffer to manipulate directory scores on the fly:

| Key | Action | Effect |
|-----|--------|--------|
| `+` | `zoxide add <path>` | Boost the directory's frecency score |
| `-` | `zoxide remove <path>` | Remove directory from zoxide database |

After each action, the candidate list refreshes live in the minibuffer so you see the updated rankings immediately.

---

## How Embark works with consult

Embark lets you define **category-specific actions** for candidates in a completing-read minibuffer. When you're in the zoxide consult buffer, candidates carry the category `zoxide-path` (set via `:category 'zoxide-path` in our `consult--read` call).

By default, pressing `C-.` in the minibuffer opens the embark actions menu. But we can also add **direct keybindings** that bypass the menu entirely — `+` and `-` become immediate actions.

---

## Proposed implementation plan

### 1. Define embark action functions

```elisp
(defun embark-zoxide-add (candidate)
  "Boost the score for the selected zoxide directory.
Runs `zoxide add' to increase its frecency."
  (zoxide-run nil "add" candidate))

(defun embark-zoxide-remove (candidate)
  "Remove the selected directory from the zoxide database.
Runs `zoxide remove' to delete its entry."
  (zoxide-run nil "remove" candidate))
```

### 2. Bind them in embark's zoxide-path keymap

```elisp
(defvar-keymap embark-zoxide-path-map
  :doc "Keymap for embark actions on zoxide-path candidates."
  :parent embark-general-map
  "+" #'embark-zoxide-add
  "-" #'embark-zoxide-remove)
```

### 3. Register the keymap for the `zoxide-path` category

```elisp
(add-to-list 'embark-keymap-alist '(zoxide-path . embark-zoxide-path-map))
```

This tells embark: "When the minibuffer category is `zoxide-path`, use this keymap."

### 4. Live refresh after action

After running `zoxide add/remove`, the async pipeline still has the old candidate list. We need to trigger a refresh so the updated rankings appear immediately.

There are two approaches:

**Option A: `embark--restart`** — Restarts the entire consult session, re-running the async pipeline from scratch. Simple but causes a brief flicker.

```elisp
(defun embark-zoxide-add (candidate)
  (interactive)
  (zoxide-run nil "add" candidate)
  (embark--restart))
```

**Option B: Pipeline flush + re-query** — Send `'flush` to the async pipeline, then re-send the current input string to trigger a new query. More elegant, no flicker.

```elisp
(defun embark-zoxide-add (candidate)
  (interactive)
  (zoxide-run nil "add" candidate)
  ;; Flush stale candidates and re-query with current input
  (let ((input (minibuffer-contents-no-properties)))
    (embark--restart)))  ;; simplest for now
```

`embark--restart` is the pragmatic choice — it re-runs the async pipeline, which re-executes `zoxide query -ls <input>` with the updated database.

### 5. Where to put the code

Can live in `keybinds.el` near the zoxide dispatch function, or in a separate `zoxide-embark.el` file.

---

## Edge cases

| Scenario | Behavior |
|----------|----------|
| `+` on an already-ranked dir | `zoxide add` increases its score, it may move up in the list |
| `+` on a dir not in zoxide | `zoxide add` adds it, it appears on next refresh |
| `-` on a dir not in zoxide | `zoxide remove` is a no-op, no error |
| `-` on a frequently-used dir | Removed from database, disappears from list on refresh |
| No embark installed | `zoxide-path` keymap is never loaded, no error |
| Multiple `+` presses in a row | Each press boosts the score independently |

---

## Future possibilities

- **Custom score value**: Instead of a plain `zoxide add`, use `zoxide add --set-score <N>` to set an arbitrary score
- **Numeric prefix**: Use `C-u 5 +` to boost by a specific amount
- **Annotate scores in real-time**: After each action, re-annotate the candidate list without a full restart
