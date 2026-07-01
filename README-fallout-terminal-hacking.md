# Fallout Terminal Hacking Minigame

A fully playable Fallout terminal hacking minigame written in pure Python. It replicates the classic mechanics from Fallout 3, 4, and New Vegas — pick the correct password out of a pool of decoy words within a limited number of attempts, using a "Likeness" score (matching characters in the correct position) to narrow down each guess.

## What I Learned (and Why It Matters)

Building this project reinforced core Python fundamentals — string comparison, list/set operations, control flow, and using the standard library (`random`, `sys`, `time`) to build something interactive without any external dependencies. Writing `get_likeness()` was a good exercise in using `zip()` to compare two sequences position-by-position, and structuring the game loop around a shrinking `attempts_left` counter reinforced how to manage game state cleanly. It's a good reminder that engaging gameplay can come from a small, well-designed set of rules rather than complex systems.

## Deployment Instructions

Save the script as `terminal_hack.py` and run it with `python3 terminal_hack.py` from any terminal with Python 3 installed — no external packages required. Each session randomly selects 8 candidate words and a secret password from the word pool; enter guesses exactly as they appear in the list to receive a Likeness score, and use that score to deduce the correct password before running out of attempts.

## What I Enjoyed Most

The most enjoyable part was recreating the "feel" of the in-game terminals — the line-by-line typing effect from `type_effect()` and the RobCo-style header text go a long way toward making a simple word-guessing game feel like it belongs in the Fallout universe. It was also satisfying to see how a single small function (the Likeness calculator) is really the entire core of the game's logic.
