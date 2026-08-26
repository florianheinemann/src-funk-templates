# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file learning app for the German SRC (Short Range Certificate) radio
exam. `index.html` is the entire application: CSS, JavaScript and all content
are inline. There is no build step, no dependency manager, no test runner and
no server. Open the file in a browser and it runs.

Only Google Fonts is loaded from the network; everything else is self-contained
and works offline.

## Controls

| | |
|---|---|
| click / tap, space, `→` | reveal, then advance |
| `←` | step back (browse mode only) |
| switch, top left | **Üben** (drill) ↔ **Blättern** (browse) |
| dot, top right | theme: auto → light → dark |

The counter next to the mode switch shows position in the deck, plus the round
number once past the first round in drill mode.

## The card data

All eleven cards live in one flat `CARDS` array at the top of the script in
`index.html`. Each entry is `{ titel, abschnitte }`, and `abschnitte` holds
exactly what appears on screen — there is no reference layer and no fallback.

Two states, and the difference matters:

| | Effect |
|---|---|
| key present (a string, `""` included) | row is shown; `""` renders the label with an empty cell |
| key absent | the row does not exist for this card |

The empty string is not a placeholder for missing content. It is used where a
field is deliberately not spoken — no announcement, no closing word — and the
empty cell under the label *is* that statement. Do not "helpfully" fill those
with `keine` or `entfällt`, and do not drop the key instead.

### Editing the cards

`CARDS` is a JSON literal embedded in HTML, so hand-editing is error-prone. The
established workflow is a Python script that bracket-matches the array out of
the file, parses it, mutates it, and writes it back:

```python
import json
p = 'index.html'; s = open(p, encoding='utf-8').read()
start = s.index('const CARDS = ') + len('const CARDS = ')
i, depth = start, 0
while True:
    if s[i] == '[': depth += 1
    elif s[i] == ']':
        depth -= 1
        if depth == 0: break
    i += 1
C = json.loads(s[start:i + 1])
# ... mutate C ...
open(p, 'w', encoding='utf-8').write(
    s[:start] + json.dumps(C, ensure_ascii=False, indent=2) + s[i + 1:])
```

Adding or removing a card means editing this array; there is no separate deck
list any more.

### Shortening wording

Normalising notation and shortening prose are separate operations; mixing them
is how wording drifts. Doing both at once once changed the meaning of 19
placeholders before anyone noticed.

There is no reference copy of the original wording any more, so a condensed
field cannot be checked against anything afterwards. Change wording
deliberately and one thing at a time.

## The notation system

One notation across every field and template, rendered by `render()` into
colour-coded spans:

| Written | Meaning | Class / colour |
|---|---|---|
| `GROSSBUCHSTABEN` | spoken verbatim | none, `--lead` |
| `<Sentence case>` | fill-in | `.fill`, `--fill` |
| `3x` | say three times | `.rep`, `--mute` |
| `( )` | optional | `.opt`, `--mute` |
| `oder` | alternative | `.alt`, `--mute` |
| `Option N:` | one of several whole variants | `.alt`, `--mute` |

`render()` tracks parenthesis depth so text inside `( )` is tinted too, while a
`<placeholder>` nested in parentheses keeps its own colour. Anything added to
the notation must be handled there.

## Row structure

`SECTIONS` defines both the set of rows and their order. `CORE` marks the rows
that make up the spoken call (their labels take the accent colour); `NOTE`
marks commentary rows (rendered dimmer).

Rows size to their own content, so a row sits at a different height from card to
card. That is deliberate: no cell may hide text behind a scroller of its own.
The whole table scrolls as one unit.

## Modes

`browse` (a boolean) switches between two behaviours that share the same
rendering path:

- **Üben** (drill) — shuffled deck, card covered until tapped, back-navigation
  disabled on purpose so you cannot peek at a card already answered.
- **Blättern** (browse) — documented order, wraps around, always revealed,
  `←` steps back.

`setMode(which, keep)` with `keep: true` preserves the card currently on screen
across a mode switch. Mode and theme persist via `localStorage`, wrapped in
try/catch because storage throws when the file is opened from a `data:` URL.

## Theming

Four CSS blocks define the palette: `:root`, the `prefers-color-scheme: dark`
media query guarded for `data-theme="auto"`, and explicit `[data-theme="light"]`
/ `[data-theme="dark"]` overrides. **A new colour token must be added to all
four** or the manual toggle breaks in one direction.

## Verifying changes

There is no test suite. Verify in a browser and measure rather than eyeball;
bugs here have repeatedly been invisible by sight and obvious in numbers.

Open `index.html` in the browser pane and drive it with injected JavaScript:
walk all eleven cards, collect values, and assert on the aggregate — e.g. count
cells still containing `[` brackets, check that no hidden row kept the previous
card's wording, or that no `Meldung` still ends in `OVER`/`OUT`.

For a refactor that must not change what is displayed, snapshot every cell —
text plus span classes — before and after and compare. A rendered checksum that
matches is proof; "looks right" is not.

Note that the preview pane serves the file as a `data:` URL, so `localStorage`
is unavailable there; persistence cannot be verified in the pane and must be
checked by opening the file directly.
