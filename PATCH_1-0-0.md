# PATCH 1-0-0 — Consult integration + frecency ranking fix

## Problem Statement

`zoxide-travel` currently suffers from two issues:

### 1. Ranking is broken

`zoxide-open-with` calls `zoxide-query` → `zoxide-query-with "-l"` → `zoxide query -l`.

The `-l` flag lists ALL directories from zoxide's database, but **without a query string to score against, zoxide cannot rank them by frecency**. The results appear in filesystem/insertion order rather than by frequency + recency.

Zoxide's ranking engine only activates when a query string is provided:
```bash
zoxide query -ls "project"   # → all matches sorted by frecency, with scores
zoxide query -l              # → all entries, NO ranking, NO scores
```

The `-s` flag adds a frecency score (float) as a prefix on each line:
```
1528.0 /home/user/project
   4.0 /home/user/other
```

### 2. Plain `completing-read` without consult integration

`zoxide-open-with` uses plain `completing-read`, which means:
- No live preview of the selected directory
- No async filtering (candidates are fetched once upfront)
- No consult category (no `consult-buffer` source compatibility)
- No narrowing (can't filter by prefix like `d ` for directories)

## Proposed Changes

### A. Score display with configurable padding

Each zoxide entry is displayed as:

```
spacer 1528.0  /home/user/project
spacer    4.0  /home/user/other
```

- **Score** is right-justified within a fixed-width field, configurable via `zoxide-score-width` (default 6 chars — enough for scores like `99999.0`)
- **Padding** between score and path is configurable via `zoxide-score-path-padding` (default 4 spaces)
- The **path** is the actual candidate value passed to the callback
- The display string is built from `zoxide query -ls <input>` output, parsing each `<score> <path>` line into a propertized string for vertico/consult

```elisp
(defcustom zoxide-show-scores t
  "When non-nil, display the frecency score to the left of each path.
When nil, only the path is shown (scores are still used for ranking
behind the scenes)."
  :type 'boolean
  :group 'zoxide)

(defcustom zoxide-score-width 6
  "Width of the score field when displaying zoxide results.
The score is right-justified within this width.
Has no effect when `zoxide-show-scores' is nil."
  :type 'integer
  :group 'zoxide)

(defcustom zoxide-score-path-padding 4
  "Spaces between the score and the path in zoxide results.
Has no effect when `zoxide-show-scores' is nil."
  :type 'integer
  :group 'zoxide)
```

### B. Async builder using `zoxide query -ls <input>`

Replace `zoxide-query` / `zoxide-query-with` with an async builder that:

1. **On each input change**: shells out to `zoxide query -ls <input>` — returns frecency-sorted matches with scores
2. **Empty input**: `zoxide query -ls` (no query) — returns ALL entries with scores, ranked globally

Each line of output is parsed as `SCORE  PATH`, then formatted as:
```elisp
(propertize (format "%s  %s"
              (propertize score 'face 'zoxide-score-face)
              path)
            'zoxide-score score
            'zoxide-path path)
```

The **candidate value** passed to consult/completing-read is the **path** alone. The display string includes the score.

### C. Consult integration

Replace `completing-read` with `consult--read` using:

- **`:prompt "zoxide: "`**
- **`:category 'zoxide-path`** — registers a new category so Marginalia can annotate with file size/type
- **`:require-match t`**
- **`:sort nil`** — defer sorting to zoxide's frecency engine
- **`:async`** — use consult's async builder that shells out to `zoxide query -ls <input>` on each keystroke without blocking Emacs
- **`:state** — `consult--file-preview` if available, otherwise nil (no custom preview)
- **`:lookup** — extract the path from the candidate (stored as a text property on the display string)

### D. New consult source for `consult-buffer`

Optionally, add a `consult-buffer` source so zoxide paths appear alongside buffers, bookmarks, etc.:

```elisp
(defcustom zoxide-consult-source
  '( :name     "Zoxide"
     :narrow   ?z
     :category 'zoxide-path
     :face     'consult-file
     :items    (lambda () (zoxide-query))
     :action   (lambda (path) (funcall zoxide-travel-callback-function path)))
  "Consult source for zoxide directories.")
```

## Implementation Details

### Parsing `zoxide query -ls` output

```elisp
(defun zoxide-parse-score-line (line)
  "Parse a single LINE from `zoxide query -ls` into (SCORE . PATH).
LINE format is \"<score> <path>\" where score is right-justified."
  (when (string-match (rx bol (group (+ (any digit ?.)))
                          " "
                          (group (+ any)))
                      line)
    (cons (string-to-number (match-string 1 line))
          (match-string 2 line))))
```

### Formatting with score

```elisp
(defun zoxide-format-entry (score path &optional score-width padding)
  "Format a zoxide entry with SCORE right-justified before PATH.
When `zoxide-show-scores' is nil, only the path is returned.
SCORE-WIDTH controls the score field width (default `zoxide-score-width').
PADDING is the number of spaces between score and path (default `zoxide-score-path-padding')."
  (if (not zoxide-show-scores)
      path
    (let* ((sw (or score-width zoxide-score-width))
           (pad (or padding zoxide-score-path-padding))
           (score-str (format (format "%%%d.1f" sw) score)))
      (concat
       (propertize score-str 'face 'zoxide-score-face)
       (make-string pad ?\s)
       path))))
```

### Async builder for consult

```elisp
(defun zoxide-consult-builder (input)
  "Return a list of (display . candidate) cons cells for INPUT.
INPUT is the user's typed query.  Calls `zoxide query -ls' for ranking.
Each entry shows the score left of the path."
  (let* ((args (if (or (not input) (string-empty-p input))
                   '("query" "-ls")
                 (list "query" "-ls" input)))
         (raw (zoxide-run nil args))
         (lines (remove "" (split-string raw "\n" t))))
    (mapcar (lambda (line)
              (pcase (zoxide-parse-score-line line)
                (`(,score . ,path)
                 (cons (zoxide-format-entry score path) path))
                (_ nil)))
            lines)))
```



### Score face

```elisp
(defface zoxide-score-face
  '((t (:inherit font-lock-comment-face)))
  "Face for the frecency score in zoxide results."
  :group 'zoxide)
```

### Updated `zoxide-travel` using async consult

```elisp
;;;###autoload
(defun zoxide-travel ()
  "Open a path from zoxide, ranked by frecency with consult.
Shows the frecency score to the left of each path.
The callback is controlled by `zoxide-travel-callback-function'."
  (interactive)
  (if (require 'consult nil t)
      (let ((candidate
             (consult--read
              :prompt "zoxide: "
              :category 'zoxide-path
              :require-match t
              :sort nil
              :lookup (lambda (selected)
                        (when selected
                          (get-text-property 0 'zoxide-path selected)))
              :state (when (fboundp 'consult--file-preview)
                       (consult--file-preview))
              :async (lambda (input)
                       (let ((args (if (or (not input) (string-empty-p input))
                                       '("query" "-ls")
                                     (list "query" "-ls" input))))
                         (zoxide-parse-results
                          (zoxide-run nil args)))))))
        (when candidate
          (funcall zoxide-travel-callback-function candidate)))
    ;; Fallback to old completing-read
    (zoxide-open-with nil zoxide-travel-callback-function t)))
```

## Edge Cases & Considerations

| Concern | Handling |
|---------|----------|
| `zoxide` not installed | `zoxide-executable` returns nil → error with actionable message |
| Empty database | `zoxide query -ls` returns empty → show "no zoxide entries yet" |
| Consult not installed | Fall back to `completing-read` (existing behavior) |
| Very large database | Async builder — consult debounces and runs async via `zoxide-run` |
| Score formatting with decimals | `%.1f` format — one decimal place, configurable |
| Path contains trailing spaces | `string-match` regex uses `(+ any)` which is greedy to end of line |
| User presses C-g | consult handles abort gracefully |
| No preview needed | `consult--file-preview` used if available, otherwise no preview state |

## Configurable Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `zoxide-travel-callback-function` | `#'find-file` | What to do with the selected path (configurable) |
| `zoxide-show-scores` | `t` | Show/hide the frecency score next to each path |
| `zoxide-score-width` | `6` | Width of the right-justified score field |
| `zoxide-score-path-padding` | `4` | Spaces between the score and the path |

## Migration

- `zoxide-travel` and `zoxide-travel-with-query` are the main entry points changed
- `zoxide-open-with` is kept for backward compatibility (used by third-party callers)
- `zoxide-query` / `zoxide-query-with` remain unchanged for other users
- The new async builder is named `zoxide-consult-builder` to avoid name collisions
