# Noise Floor

A quiz game over Electrical Engineering course content where every answer
takes you deeper into the ocean. Built to study for exams, and to give
friends something to compete over on breaks.

- **Daily dive**: 7 prompts from every course, the same for everyone that
  day. The right answer isn't enough: the **rarer** your correct answer,
  the deeper it takes you. "Name a logic gate": *AND* sinks you 281 m,
  *XNOR* 578 m.
- **Course dive**: every prompt in one course, then every accepted answer
  and a study note.
- **Precision dive** (exam prep): calculation problems with random
  component values. Rarity doesn't matter here, accuracy does: the
  closer your number, the deeper you go. Each problem ends with the
  worked solution.

Points become depth at 15.6 m per point, so a perfect daily dive
(7 x 100 points) lands exactly on the bottom of the Challenger Deep,
10,935 m down. In the browser you pilot an ROV on a tether from a
cable-laying ship at dusk, drawn as glowing line art like an
oscilloscope display. Every answer plays a dive you watch happen, past a
depth ruler, sea life and real landmarks (the Titanic at 3,800 m, the
trench walls of the hadal zone), with synthesized sound. The HUD's scope
shows your signal sinking toward the noise floor as you go deeper.

The daily dive is a new set of 7 every day, never the same set twice,
and you get one attempt (the timer is always on; reloading resumes the
attempt and the question you left counts as 0). Untimed course and
precision dives have a restart button.

## Playing in the browser

Open `web/dist/index.html`, or the GitHub Pages site once it's set up
(see "Publishing"). It's one self-contained file with no server: answer
counts are kept in the browser's local storage, so rarity reflects the
dives taken on that device. The daily set is the same for everyone.

## Leaderboard

The copy hosted on claude.ai has a shared leaderboard (Today and
All-time). A finished solo daily dive posts its points and depth once,
under your claude.ai name or a nickname you set on the board. Course
dives, exam prep and pass-and-play games don't post. Each player can
only write their own entry, but scores are self-reported, so it's a
board for friends, not a competition with prizes. The local file and
the GitHub Pages copy have no server, so they show a note instead of
the board.

## Playing in the terminal

```sh
cmake --preset default && cmake --build build/default
build/default/noisefloor daily                         # today's 7 prompts
build/default/noisefloor daily --players Arya,Sam,Lee  # pass-and-play
build/default/noisefloor course ECE3330                # every prompt in one course
build/default/noisefloor exam ECE3331                  # calculation problems
build/default/noisefloor courses                       # list courses
```

| Flag | Meaning |
|---|---|
| `--date YYYY-MM-DD` | play another day's set (`daily` only) |
| `--players a,b,c` | take turns on one machine; each answer is pushed off screen before the next player looks |
| `--time <s>` | per-answer timer, default 30 (90 for `exam`), `0` turns it off |
| `--stats <path>` | answer counts file: read before the game and updated after, so rarity reflects your group's real answers |
| `--count N`, `--seed N` | `course` and `exam`: how many prompts or problems, and a fixed shuffle |
| `--banks <dir>` | load prompt banks from somewhere else |

**Rules.** In the daily and course dives, type any correct answer. Wrong
answers cost 5 seconds of air, and you can keep trying until it runs
out. Small typos are forgiven ("metastabilty" counts as metastability),
and the game tells you what it counted your answer as. If a typo is
equally close to two answers, it asks you to type it again instead of
guessing. In a precision dive you get one answer per problem, typed with
SI prefixes if you like (`4.7k`, `330 n`, `2.2 MHz`, `15 µs`).

At the end, each player gets a line to paste in the group chat:

```
Arya  ▂▃▁▃▃▂▂  224 pts  3,499 m
```

## Courses

| Code | Course | Prompts | Problem types |
|---|---|---|---|
| ECE 2205 | Electric Circuits I | 10 | 8 |
| ECE 2231 | Introduction to Electronics | 10 | 9 |
| ECE 2233 | Circuits and Systems | 10 | 8 |
| ECE 2277 | Digital Logic Systems | 11 | 6 |
| ECE 3330 | Control Systems | 11 | 7 |
| ECE 3331 | Introduction to Signal Processing | 10 | 9 |
| ECE 3375 | Microprocessors and Microcomputers | 10 | 8 |

Problem types include voltage dividers, RC time constants and charging,
reactance, resonance, RC cutoff, impedance and phase, Q factor, diode and
zener currents, LED resistors, rectifier ripple, BJT bias points, g_m and
common-emitter gain, MOSFET drain current, aliasing, DFT bins, dB
conversions, ADC step size, SQNR and output codes, binary/hex/two's
complement, maximum clock frequency, address lines and ranges, UART
transfer time, timer overflow and PWM frequency, execution time, percent
overshoot, settling and peak time, and steady-state error.

## How scoring works

**Rarity (daily and course dives)**

1. **Estimate how popular the answer is.** Each answer in a bank has a
   hand-estimated popularity weight. Once real answers are recorded, they
   are blended in, and the prior's influence fades as data builds up:
   `share = (times given + 25 * prior share) / (total answers + 25)`.
2. **Score its information content.** `points = 10 * log2(1 / share)`,
   clamped to 10..100. Every halving of an answer's popularity is worth 10
   more points.

Everyone in one game is counted before anyone is scored. If three
friends all say NAND, NAND gets less rare for all three, and answer order
doesn't matter.

**Accuracy (precision dive)**

100 points within 0.1% of the exact answer, 33 fewer for every tenfold
more error (67 at 1%, 33 at 10%), down to 20 at 25%. Anything further
off, or with the wrong sign, scores 0.

The reasoning behind all three formulas is in
[`DESIGN_DECISIONS.md`](DESIGN_DECISIONS.md) (#3, #4, #9 and #10).

## Prompt banks

One plain-text file per course in [`banks/`](banks/):

```
course ECE2277 | Digital Logic Systems

prompt dl-gate | Name a logic gate.
note NAND and NOR are "universal": either one on its own can build every other gate.
answer 26 | AND | and gate
answer 4 | XNOR | exclusive nor | xnor gate | equivalence gate
```

- `answer <weight> | <display name> | <aliases...>`. The weight is the
  popularity prior (roughly "out of 100 classmates, how many say this?").
- Aliases are compared after normalizing case, punctuation and spacing,
  so `K-map` already covers `k map`; listing both is an error.
- Each prompt needs at least 6 answers. Rarity means little with fewer.
- Prompt ids must be unique across all banks, because they key the
  answer counts.

To add a course, add a `.bank` file. The tests load every bank, so a
duplicate alias or a too-short prompt fails CI rather than a game. A
test also checks that swapping any two adjacent letters in any alias
never gets a *different* answer credited.

## Tests

`build/default/noisefloor_tests` (also run by `ctest`, on Linux, macOS
and Windows, and under ASan/UBSan in CI). Every exam problem type is
checked against an independent recomputation of its answer over 150
random problems each: a different formula or method where one exists
(complex-number impedances, bisection for the −3 dB point, a search over
the step response for overshoot, reading the bit pattern back out of the
question). Also covers the normalizer and edit distance (hand-worked
cases plus a randomized one-edit property test), the bank parser's error
messages, scoring (hand-computed shares and points, and the prior checked
a second way as literal pseudo-counts), the daily selection (published
FNV-1a and SplitMix64 test vectors, results pinned against an independent
Python implementation, uniformity, and stability when a prompt is added),
and whole scripted games with a fake clock.

## Building the browser version

```sh
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk && ./emsdk install latest && ./emsdk activate latest && source ./emsdk_env.sh
cd /path/to/noisefloor && web/build_web.sh
```

This compiles `src/` plus `web/bindings.cpp` to WebAssembly, packs the
`banks/` files into it, and inlines everything with `web/page.html` and
`web/app.js` into `web/dist/index.html`. Rebuild after changing any of
those, including adding a bank.

## Publishing

`.github/workflows/pages.yml` deploys the committed `web/dist/index.html`
to GitHub Pages on every push to `main`. One-time setup in the repo:
Settings -> Pages -> Source: "GitHub Actions". The game is then at
`https://<your-username>.github.io/<repo-name>/`.

## Layout

```
include/noisefloor/, src/   the game logic (matching, scoring, depth, daily set, exam problems, play loops)
cli/main.cpp                terminal version
banks/                      one .bank file per course
tests/                      Catch2 tests, including checks on the real banks
web/                        bindings.cpp (C++ -> JS), page.html, app.js, build_web.sh
web/dist/index.html         the built browser version (committed, deployed)
```

## Roadmap

- More courses from the progression sheet (ECE 3332 Electric Machines,
  ECE 3336 Electromagnetic Theory, ECE 4437 Communications, ...).
- **Circuit prompts** checked by a circuit simulator: "Build a low-pass
  filter with a cutoff between 900 Hz and 1.1 kHz." Any circuit that
  meets the spec counts, and uncommon topologies score more.
- Share answer counts through the leaderboard's database too, so rarity
  reflects the whole friend group (today the board stores scores only).
- A small server to share answer counts across a whole class instead of
  one browser or one `--stats` file.
