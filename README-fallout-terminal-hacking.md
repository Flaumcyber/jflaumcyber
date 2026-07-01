# Fallout Terminal Hacking Minigame

Two recreations of the Fallout terminal hacking minigame, in one project. The first is a simple, pure-Python CLI version that replicates the core mechanic — guessing a password from a pool of decoys using a Likeness score. The second is a far more faithful recreation built with Python's `curses` library: five difficulty levels, a real two-column memory dump, an arrow-key cursor, and the authentic bracket mechanic where one bracket per puzzle fully replenishes your attempts and the rest remove a dud word.

## What I Learned (and Why It Matters)

**Basic version.** Building this reinforced core Python fundamentals — string comparison, list/set operations, control flow, and using the standard library (`random`, `sys`, `time`) to build something interactive without any external dependencies. Writing `get_likeness()` was a good exercise in using `zip()` to compare two sequences position-by-position, and structuring the game loop around a shrinking `attempts_left` counter reinforced how to manage game state cleanly.

**Advanced version.** This build pushed well past the original CLI version — instead of printing text and reading input with `input()`, `curses` gives full control over cursor position, screen redraws, and reverse-video highlighting, which is what it actually takes to reproduce the in-game feel of moving a cursor over a wall of text. Structuring the grid as a list of `GridLine` objects (each tracking its own word, bracket info, and removed/used state) made it much easier to reason about than a flat string, and separating "what's under the cursor" from "what happens when you press Enter" kept the input-handling loop simple. Building five difficulty tiers off one shared `WORD_LISTS` dictionary, keyed by word length, was a good exercise in designing configuration data so behavior scales without duplicating logic. Together, both versions were a useful reminder that engaging gameplay can come from a small, well-designed set of rules — and that the same core mechanic can be built at very different levels of fidelity depending on the tools you reach for.

## Deployment Instructions

**Basic version.** Save the script as `terminal_hack.py` and run it with `python3 terminal_hack.py` from any terminal with Python 3 installed — no external packages required. Each session randomly selects 8 candidate words and a secret password from the word pool; enter guesses exactly as they appear in the list to receive a Likeness score, and use that score to deduce the correct password before running out of attempts.

**Advanced version.** Save the script as `fallout_hacking_advanced.py` and run it from a real terminal (curses requires an actual TTY — it won't run in a browser or most IDE consoles): `python3 fallout_hacking_advanced.py --difficulty=3` (or `-d1` through `-d5`). Use the arrow keys to move the cursor across the two content columns, and press Enter on a highlighted word to guess it, or on a highlighted bracket to activate it. Run with `--help` for the full option list.

## What I Enjoyed Most

**Basic version.** The most enjoyable part was recreating the "feel" of the in-game terminals — the line-by-line typing effect from `type_effect()` and the RobCo-style header text go a long way toward making a simple word-guessing game feel like it belongs in the Fallout universe. It was also satisfying to see how a single small function (the Likeness calculator) is really the entire core of the game's logic.

**Advanced version.** Getting the bracket mechanic exactly right was the most satisfying part — making sure exactly one bracket per puzzle is reserved to fully replenish attempts (tracked with a single `replenish_bracket_used` flag) while every other bracket removes a dud, matching the real strategy players use of burning through guesses and then hunting the screen for a reset. Watching the cursor wrap cleanly between the two columns and seeing a dud word visibly turn into a row of dots after being removed made the whole thing feel like a genuine terminal, not just a word-guessing script.
