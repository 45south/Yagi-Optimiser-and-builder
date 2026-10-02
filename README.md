# Yagi Optimiser (NEC2++ + Differential Evolution)

**v1.0.042** — Dave Headland — https://github.com/45south

Searches element lengths, spacings, and (optionally) stacking heights for
a Yagi — multiple reflectors, one driven element, multiple directors —
to maximise gain, front-to-back ratio, and sidelobe suppression while
holding VSWR near a target feed impedance, across a frequency range.
Uses NEC2++ (`libnecpp`) as the EM solver — the same method-of-moments
math EZNEC is built on — not EZNEC itself, since EZNEC has no
scripting/automation interface.

Two ways to use it:
- **`yagi_gui.exe`** — a native Windows dialog: fields and dropdowns
  instead of hand-editing config files, a live log while it runs, preset
  buttons for common goals. Windows/MinGW only.
- **`yagi_optimize.exe`** — the same engine from the command line, with
  a plain-text `.cfg` file. Builds on Linux/macOS too, not just Windows.

The GUI doesn't do any of the antenna math itself — it just writes a
`.cfg` file from the form and runs the console tool as a child process,
streaming its output live. Both give identical results for identical
settings.

## Documentation

- **This file** — build instructions, quick start, how it works
- **(USER GUIDE.pdf)** — day-to-day usage, workflows, what
  the output means
- **(FIELD REFERENCE.pdf)** — every setting in both
  the GUI and the `.cfg` file, in detail, with recommended values —
  the authoritative reference; if this README and that file ever
  disagree, trust `FIELD_REFERENCE.pdf`

## Quick start

**GUI:** run `yagi_gui.exe`, fill in the fields (or pick a **Priority
preset** to start from a sensible combination), click **Run Optimiser**.
Watch the log. **Reset to Defaults** clears everything back to sensible
starting values; fields showing in blue are ones you've changed from
their default, so you can see at a glance what's been customised.

**Command line:**
```
yagi_optimize.exe yagi_example.cfg
```
Edit `yagi_example.cfg` (plain `key = value` text) for your design —
every key is documented in `FIELD_REFERENCE.pdf`.

Don't double-click `yagi_optimize.exe` in File Explorer — it needs the
config file as an argument, which only works from a command line (or via
the GUI, which always supplies one for you).

## What comes out

Printed progress every 10 generations — current score, worst-case
gain/F-B/VSWR/sidelobe across your frequency band, and boom length — plus
two files: `<out_prefix>.txt` (the same report) and `<out_prefix>.nec` (a
plain-text NEC2 deck of the winning design).

The `.nec` file opens directly in the free tool **4nec2**, and in EZNEC
Pro/2+ via File > Open with "NEC" selected as the file type. It's also
what `seed_file` reads back in if you want to refine that design
further — including files exported from EZNEC itself (**not** `.EZ`,
which is a proprietary binary format; use File > Save Description As,
choose NEC as the file type).

Both output files are checkpointed periodically during a run, not only
at the end — safe to stop early (Cancel in the GUI, Ctrl+C at a
terminal) and still have a usable, up-to-date result.

## Features

A quick map of what's available — see `FIELD_REFERENCE.pdf` for the full
detail on every one of these:

- **Multiple reflectors, correctly stacked.** More than one reflector is
  always arranged as a real design would (stacked at different heights,
  sharing one boom position) — this is automatic based on how many
  reflectors you ask for, not a separate setting. Each symmetric pair
  gets its own independent spacing, so tightly-coupled inner pairs and
  widely-spaced outer pairs are both representable.
- **Seed from an existing design** (yours or from EZNEC) and refine it,
  instead of always searching from scratch.
- **Hard floors and ceilings**, not just weighted preferences — a
  minimum gain, minimum F/B, minimum sidelobe suppression, or VSWR
  ceiling that the search treats as real constraints, holding firm even
  when other objectives are pushed hard.
- **Wire loss** — model real conductor resistivity (copper, a couple of
  aluminium alloys, or your own value) instead of assuming perfect
  conductors.
- **A first-director-only gap range**, separate from the general
  element spacing — for designs that deliberately use one unusually
  tight gap near the feed while keeping the rest of the boom
  conventional.
- **An automatic aggressive-then-subtle search ramp** (`de_F_end`/
  `de_CR_end`) — broad exploration early in a run, fine polishing late,
  within a single run instead of needing two manually-seeded passes.
- **Multi-threaded** (OpenMP) — uses all available CPU cores by default;
  cap it with `max_threads` if running more than one instance at once.
- **Wider bandwidth via a phased driven pair** (`n_driven`) — a second
  driven dipole phased against the first, connected by a transmission-
  line strap, for a modest bandwidth gain over a single driven element.
  See `FIELD_REFERENCE.pdf`'s "Driven Pair" section.
- **A DL6WU-style seed generator** — a starting geometry approximating
  the well-known DL6WU long-Yagi taper, for a given director count and
  frequency, ready to hand straight to `seed_file`.

## How it works

All-free-space model (no ground) — standard for comparing Yagi designs
on gain/F-B/impedance; add height/ground effects separately once you've
picked a design (real-ground modelling has its own quirks, like a
genuine gain null at exactly 0° elevation — see `USER_GUIDE.pdf` if you
hit that in EZNEC). Elements are modelled as single wires; the driven
element (or, for `n_driven >= 2`, the driven chain — see
`FIELD_REFERENCE.pdf`) is fed at its centre. Search is a differential-
evolution algorithm: a population of candidate designs is repeatedly
mutated and the better performer kept, over many generations — this
handles a noisy, non-convex, multi-objective search far better than
brute-force gridding.

Score per candidate combines weighted gain, F/B, and sidelobe
suppression, minus penalties for VSWR above target, pattern squint, boom
length outside your limits, and any floor/ceiling violations. There's no
knowable "perfect score" to compare against — it's a relative signal for
whether a run is still improving, not an absolute measure (see
`USER_GUIDE.pdf` if this comes up).

Gain/F-B/sidelobe are evaluated at the pattern's *actual* peak elevation
for each design, found by a small scan (±50° from horizon) rather than
assumed to sit at the horizon — this matters specifically for
stacked-reflector and multi-driven-element designs, which aren't
vertically symmetric and can have a genuinely tilted peak.
