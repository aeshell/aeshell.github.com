---
date: '2026-06-02T18:00:00+02:00'
draft: false
title: 'Fuzzy History Search'
weight: 8
---

Æsh Readline includes an interactive fuzzy history search inspired by [fzf](https://github.com/junegunn/fzf) and [fzf.fish](https://github.com/PatrickF1/fzf.fish). Press **Ctrl+R** to open a visual, multi-result fuzzy finder that filters and ranks history entries as you type.

## Overview

When you press `Ctrl+R`, a search UI appears below the prompt:

```
demo$ _
> query_
  3/20
> 06-02 12:30 │ git status
  06-02 12:28 │ git commit -m "initial commit"
  06-02 12:25 │ mvn clean test -pl readline
  06-02 12:20 │ grep -rn "FuzzyMatch" src/
```

- **Query line**: type to filter results using fuzzy matching
- **Info line**: shows the number of matches out of total unique entries
- **Results**: ranked by relevance, with timestamps and a `│` separator
- **Selected entry**: highlighted with a cyan `>` pointer and reverse video

## Key Bindings

| Key | Action |
|-----|--------|
| Printable characters | Filter results by fuzzy matching |
| `Backspace` | Delete last character from query |
| `↑` / `↓` | Navigate the result list |
| `Enter` | Insert the selected entry into the command line |
| `Esc` | Cancel and return to the prompt |
| `Ctrl+R` (while in search) | Toggle between relevance and chronological sort |

When you press `Enter`, the selected command is placed in the command line buffer for editing -- it is **not** executed immediately. Press `Enter` again to execute it.

## Fuzzy Matching

The fuzzy matcher is a Java port of [fzf's matching algorithm](https://github.com/junegunn/fzf/blob/master/src/algo/algo.go). Characters in the query must appear in order in the history entry, but they don't need to be contiguous. The scoring rewards:

- **Consecutive matches**: characters matching in sequence score higher
- **Word boundary matches**: characters at the start of words (after spaces, `/`, `-`, `_`, `.`) get bonus points
- **camelCase transitions**: matching at `lower→UPPER` boundaries scores well
- **First character significance**: the first query character gets double bonus

For example, the query `gp` would rank `git push origin main` higher than `grep -rn pattern` because `g` and `p` both appear at word boundaries.

### Scoring Schemes

The fuzzy matcher supports three scoring presets matching fzf's `--scheme` option:

| Scheme | Use Case | Description |
|--------|----------|-------------|
| `HISTORY` | Command history (default for Ctrl+R) | Equalized boundary bonuses -- whitespace and delimiters are equally important |
| `DEFAULT` | General-purpose | Whitespace boundaries get slight extra bonus |
| `PATH` | File paths | Path separators get extra bonus, beginning of input treated as after a delimiter |

## Features

### Pre-populated Query

If you have text in the command line when you press `Ctrl+R`, it is used as the initial query. For example, if you type `mvn` and then press `Ctrl+R`, the search immediately filters to maven commands.

### Deduplication

Duplicate history entries are automatically removed. Only the most recent occurrence of each unique command is shown.

### Timestamps

Each entry shows when it was added to history (format: `MM-dd HH:mm`). Timestamps are tracked by both `InMemoryHistory` and `FileHistory`, and are persisted to the history file across sessions. Legacy history files (from older versions without timestamps) use the file's modification time as a fallback.

### Sort Toggle

Press `Ctrl+R` while the search is open to toggle between:
- **Relevance sort** (default): best fuzzy matches first
- **Chronological sort**: most recent entries first, filtered by query

## Configuration

### Using the Fuzzy Search (Default)

The fuzzy search is bound to `Ctrl+R` by default in both Emacs and Vi modes. No configuration is needed.

### Reverting to the Old Search

If you prefer the traditional inline incremental search (`(reverse-i-search)`), you can rebind `Ctrl+R`:

```java
EditMode editMode = EditModeBuilder.builder(EditMode.Mode.EMACS)
        .addAction(Key.CTRL_R.getKeyValues(), "reverse-search-history")
        .build();
```

Both actions are always available:
- `"fuzzy-reverse-search-history"` -- the new fzf-style search
- `"reverse-search-history"` -- the traditional inline search

### Forward Search

`Ctrl+S` continues to use the traditional forward incremental search (`forward-search-history`). The fzf-style search does not have a separate forward mode -- use the sort toggle (`Ctrl+R` while in search) to switch to chronological order instead.

## Using the Fuzzy Matcher Programmatically

The fuzzy matching library is available in the `org.aesh.readline.fuzzy` package for use beyond history search -- for example, custom command completion or menu filtering.

### Single Match

```java
import org.aesh.readline.fuzzy.FuzzyAlgo;
import org.aesh.readline.fuzzy.FuzzyResult;
import org.aesh.readline.fuzzy.FuzzyScheme;

FuzzyAlgo algo = new FuzzyAlgo(FuzzyScheme.DEFAULT);

int[] text = "FooBarBaz".codePoints().toArray();
int[] pattern = "fbb".codePoints().toArray();

FuzzyResult result = algo.match(false, text, pattern, true);
if (result.isMatch()) {
    System.out.println("Score: " + result.score);
    System.out.println("Matched positions: " + Arrays.toString(result.positions));
}
```

### Filtering a List

```java
import org.aesh.readline.fuzzy.FuzzyScorer;
import org.aesh.readline.fuzzy.FuzzyScheme;

FuzzyScorer scorer = new FuzzyScorer(FuzzyScheme.HISTORY);

List<int[]> entries = ...; // list of code point arrays
int[] query = "gcp".codePoints().toArray();

List<FuzzyScorer.ScoredEntry> results = scorer.scoreAll(entries, query, false);
for (FuzzyScorer.ScoredEntry entry : results) {
    System.out.println(new String(entry.text, 0, entry.text.length)
            + " (score=" + entry.match.score + ")");
}
```

### Incremental Narrowing

For interactive search where the user types one character at a time, use `narrow()` to avoid re-scanning the full list:

```java
// First keystroke: full scan
List<FuzzyScorer.ScoredEntry> results = scorer.scoreAll(entries, "g".codePoints().toArray(), false);

// Second keystroke: narrow previous results
results = scorer.narrow(results, "gc".codePoints().toArray(), false);

// Third keystroke: narrow further
results = scorer.narrow(results, "gcp".codePoints().toArray(), false);
```

## Performance

The fuzzy matcher is optimized for interactive use. JMH benchmark results:

| Scenario | Time |
|----------|------|
| Single match (3-char pattern, typical command) | ~340 ns |
| 100 history entries, 3-char query | ~4 us |
| 500 history entries, 3-char query | ~21 us |
| 1,000 history entries, 3-char query | ~42 us |

Even with 1,000 history entries, a full fuzzy search completes in under 50 microseconds -- well within the latency budget for interactive keystroke processing.

### Performance Design

- **Reusable buffers**: `FuzzyAlgo` pre-allocates working arrays and reuses them across queries
- **ASCII fast path**: the `asciiFuzzyIndex` method quickly eliminates non-matching entries in O(n) before running the O(nm) scoring algorithm
- **V1/V2 algorithm selection**: for large inputs (`N*M > 8192`), the faster V1 greedy algorithm is used instead of the optimal V2 Smith-Waterman algorithm
- **Lazy narrowing**: `FuzzyScorer.narrow()` re-scores only the previous result set when a character is appended

## Algorithm Details

The implementation ports fzf's two fuzzy matching algorithms:

**V1 (Greedy, O(n))**: Forward scan finds the first subsequence match, backward scan shortens it. Fast but may not find the highest-scoring alignment.

**V2 (Modified Smith-Waterman, O(nm))**: Examines all possible alignments to find the optimal score. Uses a score matrix with gap penalties and boundary bonuses.

Scoring constants (matching fzf):

| Constant | Value | Description |
|----------|-------|-------------|
| `SCORE_MATCH` | 16 | Per matched character |
| `SCORE_GAP_START` | -3 | First gap character penalty |
| `SCORE_GAP_EXTENSION` | -1 | Subsequent gap character penalty |
| `BONUS_BOUNDARY` | 8 | Match at word boundary |
| `BONUS_CAMEL_123` | 7 | camelCase or number boundary |
| `BONUS_CONSECUTIVE` | 4 | Consecutive matching characters |
| `BONUS_FIRST_CHAR_MULTIPLIER` | 2x | First pattern character bonus multiplier |

The gap penalty is tuned so that boundary bonuses are cancelled when the gap exceeds ~8 characters (approximately the average word length).
