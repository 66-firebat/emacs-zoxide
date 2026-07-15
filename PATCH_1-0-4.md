# PATCH 1-0-4 — Embark actions for zoxide directories

## Goal

Add embark keybindings `+` and `-` inside the zoxide consult minibuffer to manipulate directory scores on the fly:

| Key | Action | Effect |
|-----|--------|--------|
| `+` | `zoxide add <path>` | Boost the directory's frecency score by 1 |
| `-` | Remove and re-add with reduced score | Demote the directory by a fixed amount |

After each action, the candidate list refreshes live in the minibuffer so you see the updated rankings immediately.

---

## How zoxide scoring works

- Each directory gets a **frecency score** starting at 1
- Every `cd` into it (or `zoxide add`) **increments by 1**
- The `--score <N>` flag on `zoxide add` sets the **increment amount** (default 1)
- The **aging algorithm**: when total scores exceed `_ZO_MAXAGE` (default 10,000), all scores are divided so total ≈ 90% of max. Entries below 1 after aging are pruned.
- **No native decrement** exists in zoxide 0.9.9 (`-i`/`-d` flags from PR #422 are not available in this release)

## Proposed implementation

### 1. New defcustoms

```elisp
(defcustom zoxide-add-amount 5
  "Amount added to a directory's score on embark `+'.
Passed as `--score' to `zoxide add'."
  :type 'integer
  :group 'zoxide)

(defcustom zoxide-subtract-amount 5
  "Fixed amount subtracted from a directory's score on embark `-'.
The directory is removed and re-added at `max(1, current - this)'."
  :type 'integer
  :group 'zoxide)
```

### 2. Embark action functions

```elisp
(defun embark-zoxide-add (candidate)
  "Boost the score of the selected zoxide directory by `zoxide-add-amount'.
Runs `zoxide add --score <amount>' to increment its frecency."
  (interactive)
  (zoxide-run nil "add" "--score" (number-to-string zoxide-add-amount) candidate)
  (embark--restart))
```

```elisp
(defun embark-zoxide-subtract (candidate)
  "Demote the selected zoxide directory by `zoxide-subtract-amount'.
Parses the current score, removes the entry, then re-adds it
with a reduced score of `max(1, current - zoxide-subtract-amount)'."
  (interactive)
  (let* ((raw (zoxide-run nil "query" "-ls" candidate))
         (first-line (car (split-string raw "\n" t)))
         (current-score (and first-line
                             (car (zoxide-parse-score-line first-line))))
         (new-score (max 1 (- (or current-score 1)
                              zoxide-subtract-amount))))
    (zoxide-run nil "remove" candidate)
    (zoxide-run nil "add" "--score" (number-to-string new-score) candidate)
    (message "Zoxide: %s score reduced from %s to %s"
             candidate (or current-score "?") new-score))
  (embark--restart))
```

**How subtract works step by step:**

1. Run `zoxide query -ls <candidate>` to get the current line
2. Parse the score number from the output (e.g. `1528.0`)
3. Run `zoxide remove <candidate>` to delete the entry
4. Run `zoxide add --score <new-score> <candidate>` to re-add it at `max(1, current - zoxide-subtract-amount)`
5. `embark--restart` refreshes the candidate list with the updated database

**Why `max(1, ...)`?** — Scores below 1 are pruned by zoxide's aging algorithm. Keeping it at minimum 1 ensures the entry survives until the next aging cycle.

**Why no timestamp manipulation?** — `zoxide add` sets the timestamp to "now", so the re-added entry appears recent. This is acceptable — the recency boost offsets the score reduction slightly, preventing the directory from completely disappearing from rankings. If the user really doesn't want it, they can press `-` multiple times until the score drops far enough, or just remove it entirely via a future `embark-zoxide-forget` action.

### 3. Bind them in embark's zoxide-path keymap

```elisp
(defvar-keymap embark-zoxide-path-map
  :doc "Keymap for embark actions on zoxide-path candidates."
  :parent embark-general-map
  "+" #'embark-zoxide-add
  "-" #'embark-zoxide-subtract)
```

### 4. Register the keymap for the `zoxide-path` category

```elisp
(add-to-list 'embark-keymap-alist '(zoxide-path . embark-zoxide-path-map))
```

This tells embark: "When the minibuffer category is `zoxide-path`, use this keymap."

### 5. Where to put the code

Can live in `keybinds.el` near the zoxide dispatch function, or in a separate `zoxide-embark.el` file.

---

## Live refresh with `embark--restart`

After running `zoxide add` or the subtract workaround, the async pipeline still holds the old candidate list. `embark--restart`:

1. Cancels the current consult session cleanly
2. Re-runs `grease-zoxide-travel` / `eat-zoxide-travel`
3. The async pipeline executes `zoxide query -ls <current-input>` again
4. Returns updated results with the modified scores

The minibuffer contents (the user's typed input) are preserved across the restart.

---

## Edge cases

| Scenario | Behavior |
|----------|----------|
| `+` on an already-ranked dir | Score increments, may move up in rankings |
| `+` on a dir not in zoxide | Added with score 1, appears on refresh |
| `-` on a dir not in zoxide | `zoxide query -ls` returns empty → `new-score` defaults to `max(1, 0-5)` = 1 → add with score 1 (creates entry) |
| `-` on a dir with score ≤ `zoxide-subtract-amount` | `max(1, 5-5)` = 1 → reset to minimum |
| Multiple `-` presses | Each press reduces further — score 1 is the floor |
| `-` on a dir with score 1 | `max(1, 1-5)` = 1 → stays at 1 (no change) |
| No embark installed | `zoxide-path` keymap is never loaded, no error |
| `zoxide query -ls` parsing fails | `current-score` is nil → `new-score` defaults to 1 |

## Future possibilities

- **Custom increment amount**: Use `C-u 10 +` to boost by 10 instead of 1
- **Forget action**: `zoxide remove` shortcut for complete removal
- **Timestamp reset**: Modify the database timestamp to also affect recency ranking
- **Annotate scores in real-time**: After each action, re-annotate candidates without a full restart
