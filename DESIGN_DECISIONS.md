# Noise Floor design decisions

Every non-obvious choice gets its reason written down when it's made,
both for future-me and for explaining the project in an interview.

## 1. One C++ core, two front ends

All the game logic lives in `noisefloor_core`. The terminal CLI and the
browser page are both thin shells around it. The browser runs the same
C++, compiled to WebAssembly (decision 8), not a JavaScript rewrite.
So the tests cover what players actually run, and a fix lands in both
versions at once.

## 2. Answer matching: normalize, then a length-scaled typo budget

Players type fast under a timer, so "metastabilty" should count. But
forgiving typos is the riskiest part of the game. A matcher that's too
loose quietly credits an answer the player didn't mean, which is worse
than rejecting a typo.

- **Normalize first** (`normalize()`): lowercase, drop apostrophes,
  turn every other punctuation mark into a space, collapse spaces. This
  handles the differences that aren't really typos: "K-map" vs "k map",
  "De Morgan's" vs "de morgans", "G(s)" vs "g s".
- **Exact alias matches always win.** A valid answer can never be
  "corrected" into a different one.
- **Edit distance is optimal string alignment** (Levenshtein plus
  adjacent swaps), because swapped letters ("teh") are the most common
  typing error and plain Levenshtein charges them 2. OSA differs from
  full Damerau-Levenshtein only in rare cases ("ca" -> "abc"); a test
  pins that case so the choice is visible.
- **The budget scales with the alias length**: 0 typos under 5
  characters, 1 up to 8, 2 from 9. Short answers are where one letter
  changes the meaning: "PI" vs "PD", "VOH" vs "VOL", "lead" vs "lag".
  So they must be exact.
- **A tie is rejected, not guessed.** If the input is equally close to
  two answers ("exclusive xor": one edit from both XOR and XNOR), the
  player is asked to retype, with no time penalty, because it wasn't
  their mistake.

Checked by: hand-worked distances; a randomized property test (5000
random one-edit pairs must have distance exactly 1, and OSA must never
exceed an independently written plain Levenshtein); and a content test
over the real banks that tries every adjacent swap in every alias and
requires the result to be the same answer or rejected, never a different
answer.

## 3. Popularity: a hand-made prior, replaced by data as it arrives

Rarity needs to know how often each answer is given, and on day one
nobody has answered anything. That's the cold-start problem. Each answer
carries a hand-estimated weight (its prior share is `weight /
total_weight`), and recorded answers are blended in as a Dirichlet prior
would:

```
share = (count + a * prior_share) / (total + a),   a = kPriorStrength = 25
```

This is the posterior mean of a Dirichlet-multinomial model where the
prior is worth `a` imaginary answers. With no data it's exactly the
prior. After a few dozen real answers the two have about equal say, and
after hundreds the data dominates. `a = 25` is about one class section.
That's enough that one friend's odd answer doesn't swing everything, and
few enough that a study group's real habits show up within a week.

Two consequences, both deliberate:

- Counts for answers no longer in the bank (after a rename) still count
  toward `total`, so they can't make the remaining answers look more
  popular than they are.
- The weights in the shipped banks are **my estimates**, not survey
  data. The model is designed so that being wrong about them only matters
  for the first few days.

Checked by: hand-computed shares (e.g. NOR at `(5 + 25*0.2)/40 = 0.25`);
a second route to the same numbers, where the prior is added as literal
counts (strength 100 makes them whole numbers) and estimated with no
prior, which must agree; and the shares of all answers summing to 1.

## 4. Points are information content: 10 per bit

`points = clamp(round(10 * log2(1/share)), 10, 100)`. An answer half the
class gives is 1 bit of surprise (10 points), one in 32 is 5 bits (50),
one in 1024 is 10 bits (100). A linear score like `100 * (1 - share)`
barely separates the interesting end: 3% and 0.5% would score 97 and 99.5,
nearly the same, when the second is six times rarer. On a log scale the
gap is 25 points (51 vs 76). Log-scale scoring is also something an EE student
already knows from Shannon information in the comms courses. The 10-point
floor means any correct answer is worth something, and the 100-point cap
stops a brand-new prompt's tiny-weight answers from deciding a whole game.

## 5. The daily set: a deal through every prompt, never the same set twice

Everyone playing "today" must get the same 7 prompts, with no server.
So the set is computed from the date alone, and every step has to give
bit-identical results on every compiler. The set also has to change
every day and never repeat.

- **Not `<random>`'s distributions.** `std::mt19937` is fully specified,
  but `std::uniform_int_distribution` and `std::shuffle` are not, and
  libstdc++, libc++ and MSVC give different results. FNV-1a (64-bit) and
  SplitMix64 are both a few lines long and specified exactly; the tests
  check them against their published test vectors.
- **An endless deal, like dealing cards.** Every prompt is dealt once,
  in a shuffled order, before any comes back; then the pool is
  reshuffled for the next cycle. Day `d` (days since 1970, computed with
  Howard Hinnant's `days_from_civil`) takes the next 7 prompts of that
  deal. With 72 prompts a cycle lasts about 10 days, so every prompt
  comes up about once every 10 days.
- **Two fixed halves, so a reshuffle can't bring a prompt straight
  back.** A plain reshuffle can put the last prompt of one cycle first in
  the next, on consecutive days. An earlier attempt held recently dealt
  prompts back in the next shuffle, but that depends on the previous
  cycle, which depends on the one before. The fix: split the pool once
  into two fixed halves (by a hash of each id), and deal a fresh shuffle
  of the first half then of the second in every cycle. A prompt then
  waits at least half the pool between appearances (at least 5 days with
  72 prompts), with no reference to earlier cycles.
- **Never the same set twice, as a rule.** Shuffling alone can't promise
  that: with 72 prompts, plain dealing repeats a set a few times in 30
  years. So from 2026-01-01 (`kScheduleStart`) the schedule remembers
  every set it has dealt, and a day that would repeat one slides its
  window along the deal until the set is new. With the real banks that
  happens 4 times in 30 years; on those days the swapped-in prompt can
  also appear the next day. The search is capped, so a pool too small to
  have enough sets accepts a repeat instead of hanging (the small test
  pool of 20 prompts runs dry after about a year, which is how the cap
  was found). Computing a date replays the schedule from the start, about
  300 small steps in 2026 and 11,000 in 30 years: microseconds in C++.
- **Only ids matter**, not the order bank files were loaded in (tested).
  The cost: adding a prompt reshuffles future days. Daily sets are only
  worth comparing among friends on the same version, which a shared site
  gives them.
- The `v2` in the shuffle's salt is a version: changing it on purpose
  re-rolls every day.

Checked by: hash/RNG test vectors; daily sets pinned against a separate
Python implementation (which also implements the never-repeat rule);
30 years of daily sets with no set repeated; every prompt dealt about
equally often with at most 4 short gaps in 10 years; the same rule on a
20-prompt pool where it does real work (69 swaps in 2026).

Practice mode doesn't need agreement between machines, so it uses a
plain partial Fisher-Yates with `SplitMix64::below()`. That function
uses rejection sampling, because `r % n` slightly favours small values.

## 6. Everyone is counted before anyone is scored

In a pass-and-play game, each prompt's answers are all added to the
counts, then everyone is scored. Scoring each player as they answered
would make the first player's answer look rarer than an identical later
one, purely because of turn order. It also makes the social part work:
three friends all saying "NAND" makes NAND less rare for all three.

## 7. The timer is checked when an answer is submitted

The terminal version reads whole lines, so it can't interrupt someone
mid-typing. Instead, the elapsed time plus penalties is checked when an
answer arrives: a correct answer typed after the limit scores 0. Each
wrong answer adds 5 s of penalty, as Krillion does. Guessing is allowed
but costs time. The clock is passed in as a function, so the tests play
whole timed games with a fake clock that ticks 1 s per read and check
the exact remaining time printed.

## 8. The browser version is the C++ compiled to WebAssembly

The page could have been written in JavaScript from scratch, but then
there would be two implementations of matching and scoring that could
drift apart, and only one of them tested. Instead `web/bindings.cpp`
exposes the core through embind, and `web/app.js` only draws screens,
runs the visible countdown and saves to local storage. Every rule
(what counts as a match, the points, the daily set, the share line)
comes from the same compiled code the tests check.

- **The banks are packed into the build** (`--embed-file banks@/banks`),
  so the browser loads the exact `.bank` files the content tests check
  through the same `load_bank_dir()`.
- **One self-contained file.** `SINGLE_FILE` inlines the WebAssembly, so
  the page needs no server and works from `file://`. The build sets
  `SINGLE_FILE_BINARY_ENCODE=0` to keep the inlined module as base64.
  Newer Emscripten defaults to packing raw bytes into a string, and an
  HTML parser replaces NUL bytes inside an inline `<script>`, which would
  corrupt the module. `build_web.sh` refuses to build if the generated
  code contains a NUL or a `</script`.
- **The timer runs in JavaScript** because it has to update the screen
  ten times a second. It uses the time limit and penalty from the C++
  `SessionOptions` defaults, so the two versions can't disagree.
- **Counts live in local storage.** Without a server, rarity reflects
  the games played in one browser. The daily set doesn't depend on
  counts, so friends still play the same 7 prompts and compare share
  lines.

## 9. Exam prep scores accuracy, not rarity

For studying, a niche answer isn't the goal. On an EE exam, most marks
come from getting a number right. So the precision dive flips the
scoring: generated calculation problems, scored on how close the answer
is.

- **Generators, not a fixed question list.** Each problem type picks
  random inputs and computes the answer, so a student can dive again
  and again without memorizing answers. Component values come from the
  E12 series (1.0, 1.2, 1.5, ... 8.2), like real parts, so the numbers
  look like a real circuit.
- **Points on a log scale of error.** `100 - 33.3 * log10(error / 0.1%)`:
  100 at 0.1% or better, 67 at 1%, 33 at 10%, 20 at the 25% tolerance,
  and 0 beyond it. An answer rounded to 4 significant figures (at most
  0.05% off) always scores full marks, and 3 figures (at most 0.5%) at
  least 77. Rounding to 2 figures in the middle of a calculation (up to
  5% off) can cost more than half the points, which is the habit worth
  training out before an exam. A sign error is more than 100% off, so it
  scores 0.
- **SI prefixes are parsed** (`4.7k`, `330 n`, `2.2MEG`, `5 µs`, `3
  microseconds`). The prefix is case-sensitive like SPICE (`m` is milli,
  `M` is mega), and whole prefix words are checked first so
  "microseconds" isn't read as milli.
- **The worked solution ends with the displayed answer**, and typing the
  displayed answer always scores 100 (both tested for every problem type).
- **Every problem type is checked independently.** The tests recompute
  each type's answer over 150 random problems with a separately written
  method, different from the generator's wherever one exists:
  complex-number impedances, bisection for resonance and the −3 dB
  point, a search over the closed-form step response for overshoot and
  peak time, RK4 integration for capacitor charging, numerical
  integration for RMS and capacitor energy, SQNR derived from first
  principles, and reading the bit pattern back out of the question text
  for the number-system problems. A test also fails if a problem type
  has no independent check.

## 10. Points become depth

Krillion's hook is the dive: every good answer takes you deeper into
the ocean. Noise Floor keeps its scores in points (so the formulas above
stay simple and testable) and shows them as depth:
`metres = points × 10,935 / 700`. A perfect daily dive (7 prompts × 100
points) lands exactly on the bottom of the Challenger Deep. Most dives
end in the midnight zone (1,000 to 4,000 m), and reaching the abyssal
zone takes a strong set of rare or exact answers.

The zone boundaries (sunlight, twilight, midnight, abyssal, hadal at 200,
1,000, 4,000 and 6,000 m) are the standard oceanographic ones, and
`ocean_zone()` assigns each boundary to the deeper zone. The landmarks
on the way down (the Titanic at 3,800 m, the deepest filmed fish at
8,336 m) are real depths, kept in the page script because they are
presentation, not scoring.

The browser is built as a game screen around that idea:

- **The ocean is the screen, always.** A ship at the surface, your
  vehicle on its tether, and a water column painted band by band
  with the colour of the depth each band shows, so deeper water really is
  darker. A HUD (depth, progress, score), a compact question card and an
  input dock float over it. An earlier version covered the ocean with
  big text panels and only showed it during the dive, so the game didn't
  feel like a dive at all.
- **Every answer plays a dive**: a wind-up (the sub noses down, points
  pop up), a descent eased in and out over 1.6 to 3.8 s depending on the
  distance, and a landing (a camera bump, the depth reached). A banner
  names each ocean zone as you cross into it, and the depth ruler labels
  every landmark you pass.
- **The camera lags behind the sub in proportion to its speed**, taken
  from the derivative of the easing curve, so at top speed the sub
  plunges down the screen and the camera catches up as it slows. Without
  the lag the sub sits still while the world scrolls, which reads as the
  ocean moving, not the sub diving.
- **Reduced motion shortens the dive instead of removing it.** The first
  version skipped the dive entirely when the operating system asked for
  reduced motion (Windows does whenever "Animation effects" is off), so
  players on those machines never saw it, and the dive is the game. Now
  it's 30% shorter, without the camera shake or the streaking snow.
- **Sound is synthesized with Web Audio**, no audio files: a splash
  entering the water, a filtered-noise whoosh for the descent, bubble
  blips, a sonar ping at each zone, a thud on landing, chimes for a
  locked-in answer, a buzz for a wrong one, ticks in the last five
  seconds of air, and a low hum that darkens with depth. It starts on the
  first click, as browsers require, and a mute button remembers its
  setting.

A dive with no points has no distance to travel, so its descent counts
as finished once the wind-up ends. An earlier version computed progress
as 0 forever in that case and never ended the scene. The page has no
automated browser test in this repo yet, so that fix was checked by
hand, in a headless browser, with a round where everyone passes.

## 11. Its own look, one daily attempt, and a restart button

- **The look.** An earlier version borrowed too much from the game that
  inspired it (a pixel-art boat and character over a blue pixel sea).
  The current one has its own identity, tied to the subject. You pilot
  an ROV on a tether from a cable-laying ship at dusk, among offshore
  wind turbines, and everything is drawn as glowing line art like a
  vector oscilloscope display. The HUD's scope shows a sine wave sinking
  into a fixed noise band as you dive: about 40 dB of SNR at the surface,
  falling linearly to 0 dB at the bottom of the Challenger Deep. That
  makes the name literal: the deepest dive reaches the noise floor.
- **One daily attempt.** The browser keeps a record per date:
  `playing` with the points so far, or `done` with the share lines. A
  finished day locks the button until midnight. A reload mid-dive resumes
  the attempt, and the question on screen when the page was left counts
  as 0, so reloading can't buy a fresh timer or a second look at a
  prompt. The daily dive always has the timer on. Without a server this
  is per browser: clearing site data or switching browsers gives another
  attempt, and only a server could prevent that.
- **Restart** appears only in untimed course and precision dives. A
  timed dive is a run against the clock, and the daily dive is one
  attempt. It takes two taps, so one stray tap can't throw away a dive.


## 12. A leaderboard that only knows the daily dive

- **Only the daily posts.** Everyone gets the same prompts on the same
  day with the same timer, so those scores are comparable. Course and
  exam dives are practice: people pick different courses and can
  restart, so ranking them would mean little.
- **One post per day, solo only.** A finished solo daily writes
  `boards/<player id>` with that day's points, depth and bars, and never
  overwrites a day already posted. Pass-and-play is skipped because one
  device can't say which friend scored what.
- **Stored by id, named at render time.** The entry keeps the player's
  opaque id and an optional nickname, never a copied name; the board
  looks up names when it draws, so a renamed account updates.
- **Access rules.** Anyone who can open the page can read the board;
  each player can write only their own document. The rules can't check
  the numbers themselves, so the board trusts its players. Real
  anti-cheat would need the server to score the answers.
- **No server, no board.** When the page runs from a file or GitHub
  Pages there's no database, and the board says so instead of showing a
  fake or empty list.
