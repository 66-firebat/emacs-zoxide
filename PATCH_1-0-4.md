# PATCH 1-0-4 — Embark actions for zoxide directories

## Goal

Add direct keybindings `C-i` and `C-d` inside the zoxide consult minibuffer to manipulate directory scores on the fly:

| Key | Action | Effect |
|-----|--------|--------|
| `C-i` | `zoxide add --score <N> <path>` | Increment by `zoxide-add-amount` (default 5) |
| `C-d` | Remove + re-add at reduced score | Decrement by `zoxide-subtract-amount` (default 5) |

After each action, the candidate list refreshes in-place by re-querying zoxide and swapping `vertico--candidates` directly — no timer, no `abort-recursive-edit`, no `embark--restart`.

---

## How zoxide scoring works

- Each directory gets a **frecency score** starting at 1
- Every `cd` into it (or `zoxide add`) **increments by 1**
- `zoxide add --score <N>` adds `N` to the existing score (NOT setting it absolutely)
- `zoxide remove` deletes the entry entirely
- The **aging algorithm**: when total scores exceed `_ZO_MAXAGE` (default 10,000), all scores are divided so total ≈ 90% of max. Entries below 1 after aging are pruned.
- **No native decrement** exists in zoxide 0.9.9

## Implementation

### 1. Defcustoms (in `zoxide.el`)

```elisp
(defcustom zoxide-add-amount 5
  "Amount added to a directory's score on C-i."
  :type 'integer
  :group 'zoxide)

(defcustom zoxide-subtract-amount 5
  "Amount subtracted from a directory's score on C-d.
The directory is removed and re-added at `max(1, current - this)'."
  :type 'integer
  :group 'zoxide)
```

### 2. Action functions (in `embark.el`)

```elisp
(defun embark--zoxide-extract-path (&optional candidate)
  "Extract the path from a vertico candidate string.
Strips consult tofu characters and parses the score prefix."
  (unless candidate
    (setq candidate (vertico--candidate))
    (when (and candidate (fboundp 'consult--tofu-strip))
      (setq candidate (consult--tofu-strip candidate))))
  (or (cdr (zoxide-parse-score-line candidate)) candidate))

(defun embark--zoxide-refresh ()
  "Re-query zoxide and swap `vertico--candidates' in place.
No timer, no abort-recursive-edit — just replaces the candidate list
and re-renders vertico."
  (let* ((input (minibuffer-contents-no-properties))
         (args (if (or (not input) (string-empty-p input))
                   '("query" "-ls")
                 (list "query" "-ls" input)))
         (raw (apply #'zoxide-run nil args))
         (lines (remove "" (split-string raw "\n" t)))
         (new-candidates (delq nil (mapcar #'zoxide-consult-format lines))))
    (when (and (boundp 'vertico--candidates) vertico--candidates)
      (setq vertico--candidates new-candidates
            vertico--total (length vertico--candidates))
      (if (zerop vertico--total)
          (setq vertico--index -1)
        (when (>= vertico--index vertico--total)
          (setq vertico--index (max 0 (1- vertico--total)))))
      (vertico--prompt-selection)
      (vertico--display-count)
      (vertico--display-candidates (vertico--arrange-candidates)))))

(defun embark-zoxide-add (&optional candidate)
  "Boost the selected zoxide directory by `zoxide-add-amount'."
  (interactive)
  (setq candidate (embark--zoxide-extract-path candidate))
  (when candidate
    (zoxide-run nil "add" "--score" (number-to-string zoxide-add-amount) candidate)
    (embark--zoxide-refresh)
    (message "zoxide: %s +%d" candidate zoxide-add-amount))
  candidate)

(defun embark-zoxide-subtract (&optional candidate)
  "Demote the selected zoxide directory by `zoxide-subtract-amount'.
Reads the current score from the display string directly (NOT from
`zoxide query -ls <path>' — zoxide does fuzzy matching and would
return wrong results)."
  (interactive)
  (setq candidate (embark--zoxide-extract-path candidate))
  ;; We already parsed the score above in extract-path — the car
  ;; of zoxide-parse-score-line was the score. We lost it when
  ;; we took the cdr. Re-extract from the raw vertico candidate.
  (let* ((raw (vertico--candidate))
         (stripped (if (fboundp 'consult--tofu-strip)
                       (consult--tofu-strip raw) raw))
         (parsed (zoxide-parse-score-line stripped))
         (current-score (car parsed))
         (new-score (max 1 (- (or current-score 1) zoxide-subtract-amount))))
    (when candidate
      (zoxide-run nil "remove" candidate)
      (zoxide-run nil "add" "--score" (number-to-string new-score) candidate)
      (embark--zoxide-refresh)
      (message "zoxide: %s score reduced from %s to %s"
               candidate (or current-score "?") new-score)))
  candidate)
```

### 3. Direct keybindings via consult `:keymap` (in `zoxide.el`)

```elisp
(defvar-keymap zoxide-consult-map
  :doc "Additional keybindings for the zoxide travel minibuffer."
  "C-i" #'embark-zoxide-add
  "C-d" #'embark-zoxide-subtract)
```

Passed as `:keymap zoxide-consult-map` to `consult--read` in both `grease-zoxide-travel` and `eat-zoxide-travel`. These take priority over `vertico-map` so `C-d` in zoxide subtracts, while `C-d` in `consult-buffer` still kills buffers.

### 4. Embark menu access (secondary path, in `embark.el`)

```elisp
(defvar-keymap embark-zoxide-path-map
  :doc "Keymap for embark actions on zoxide-path candidates."
  :parent embark-general-map
  "+" #'embark-zoxide-add
  "-" #'embark-zoxide-subtract)

(add-to-list 'embark-keymap-alist '(zoxide-path . embark-zoxide-path-map))
```

Also accessible via `C-.` (embark-act) for discoverability.

### 5. Category bridge

The `:category 'zoxide-path` in `consult--read` is the bridge between consult and embark. Embark reads the category from completion metadata and looks up the keymap in `embark-keymap-alist`. No additional glue code needed.

---

## In-place refresh (no timer, no restart)

`embark--zoxide-refresh` uses the same pattern as `my/consult-kill-buffer`:

1. Reads current minibuffer input
2. Shells out to `zoxide query -ls <input>` (sync, instantaneous)
3. Parses output into formatted candidates
4. Swaps `vertico--candidates` directly
5. Re-renders with `vertico--prompt-selection` / `vertico--display-count` / `vertico--display-candidates`

No `abort-recursive-edit`, no `run-with-idle-timer`, no `embark--restart`. The minibuffer stays open the entire time.

---

## Indicator & vertico count

- **Async indicator** (`*`/`:`/`!`): Replaced with `zoxide--async-indicator-right` (in `zoxide--async-wrap`) that places the indicator at `(point-max)` with `after-string` — appears on the right.
- **Verico count** (`[1/2] `): Changed from default `"%-6s "` to brackets via `vertico.el`:

```elisp
(setq vertico-count-format '("%s " . "[%s/%s]"))
```

---

## Edge cases

| Scenario | Behavior |
|----------|----------|
| `C-i` on already-ranked dir | Score increments by `zoxide-add-amount`, may move up |
| `C-i` on dir not in zoxide | `zoxide add` adds it with score 1 |
| `C-d` on already-ranked dir | Score reduced by `zoxide-subtract-amount`, min 1 |
| `C-d` on dir with score 1 | Stays at 1 (floor) |
| Multiple `C-d` presses | Each reduces further until score 1 |
| `C-d` in consult-buffer | Still kills buffers (vertico-map, not zoxide-consult-map) |
| No embark installed | `zoxide-path` keymap won't be registered, functions load safely |
| `zoxide query -ls` parsing fails | `current-score` defaults to nil → `new-score` = 1 |
