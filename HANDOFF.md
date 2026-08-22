# Handoff Packet — D4WWCalc

Written from a cloud Claude Code session to hand off to a local one. The reason for
the switch: this cloud session's network egress is blocked (org policy) for every
Diablo site — maxroll, fextralife, icy-veins, d4builds, wowhead, the Blizzard
forums, all of it. A local session uses your own machine's network instead, so it
should just reach those pages normally.

**First message to send the new local session:** paste this whole file, or just say
"read HANDOFF.md and continue."

---

## 1. Get set up

```bash
git clone https://github.com/phantomdenied/D4WWCalc.git
cd D4WWCalc
git checkout main
```

Both `main` and `claude/user-friendly-redesign-d2rysn` point at the same commit
right now (`7d794ec`) — fully in sync, nothing to reconcile. `main` is simplest to
just keep working on directly; the feature-branch name is a holdover from earlier
in the project and doesn't need to keep existing separately.

No build step, no dependencies, no package.json. It's one file:

```
index.html   — the entire app (~1,260 lines: CSS in <style>, JS in <script>)
HANDOFF.md   — this file
.github/workflows/pages.yml — GitHub Pages deploy
```

Open `index.html` directly in a browser to run it locally — no server needed.

**Live URL:** https://phantomdenied.github.io/D4WWCalc/ (auto-deploys on push to
`main` via the Actions workflow — see §5 for one loose end there.)

---

## 2. What this is

A Diablo IV Season 14 ("Death Awakening") Whirlwind Barbarian DPS calculator.
Single-page, no framework, dark-themed. User enters their build (skill %, gear,
Paragon totals, multipliers) and it computes DPS with a full per-hit breakdown.

## 3. Architecture (read this before editing)

Everything lives in `index.html`'s `<script>` tag, roughly top to bottom:

- **Game data / constants** — `KNOWN_UNIQUES`, `BREAKPOINTS` (Whirlwind attack-speed
  frame table), `CONDITIONS`, `STAT_DEFS`, `GEAR_SLOTS`, `AFFIX_CATS`,
  `defaultMultipliers()`. Every constant here is meant to carry a `prov` tag
  (`game` / `table` / `community` / `unverified`) — see §4, this is the part that
  most needs your local browser access.
- **State** — `build` is the single source of truth (one object: gear, stats,
  scenario, multipliers). `blankBuild()` / `blankGear()` / `blankPower()` construct
  empty state. `SAMPLE` is a **hardcoded example build**, deep-cloned on load via
  `loadSample()` — don't shallow-copy it, that was a real bug that got fixed (see
  the "Fixed alongside it" note in the in-app Data Confidence panel).
- **Engine (pure functions)** — `rollup(b)` folds gear affixes + gear-bound Unique
  powers into one totals object; `computeDamage(b)` runs the actual formula;
  `validate(b, r)` produces the warning/error list. These take `build` in and
  return numbers out — no DOM access, easy to unit-test if you ever add tests.
- **Render** — `render()` and friends repaint the DOM from `build`. `scheduleRender()`
  debounces via `requestAnimationFrame`.
- **Snapshots** — localStorage-backed compare feature (`snaps`, `LS_KEY`).

### The one architectural rule that matters most

**A Unique/Mythic item's power lives on its gear slot, not in the Multipliers
list.** This was a real bug fixed this session: gear slots and the Multipliers
panel used to be two disconnected systems — equipping an item did nothing unless
you *also* separately toggled a same-named row elsewhere, and the shipped sample
build itself proved it (two "equipped" Mythics contributed zero damage). Now:
`KNOWN_UNIQUES` registry → a slot's `power: {id, ...}` field → `resolveGearPower()`
→ folded into `gearMults` inside `rollup()`. If you add a new Unique's mechanic,
follow that pattern — don't add it as a bare Multipliers row with no gear-slot tie,
that's the exact mistake already made and reverted once.

---

## 4. What actually needs your local browser — the real task list

Everything below is either a placeholder (`prov:'unverified'`, defaults to 0) or a
value carried over from an earlier version of this tool that was **never
independently verified against a live tooltip** (`prov:'community'`). None of it
is fabricated to fill a gap — it's honestly flagged in the UI (colored dot per row)
and in the in-app "Data confidence" disclosure panel at the bottom of the page.
Your job with real browser access is to close these out one at a time.

### Unverified — currently 0, no trustworthy value at all
- **Aspect of Channeling** (`defaultMultipliers`, id `channel`)
- **Aspect of Anger Management** (id `anger`)
- **Aspect of Crushing** (id `crushing`)

### Community, not re-verified this pass — has *a* number, unconfirmed
- **Ramaladni's Magnum Opus** — computed as `maxFury × perFury`, currently
  `150 × 0.425`. **This is the item you originally pointed at**:
  `https://diablo4.wiki.fextralife.com/Ramaladni's+Magnum+Opus` — get the exact
  current tooltip text/formula from there.
- **Ring of Starless Skies** — currently `10% per stack, 0–5 stacks, capped 50%`.
- **Gohr's Devastating Grips** — flat `82%`.
- **Tuskhelm of Joritz the Mighty** — flat `42%`.
- **Berserking** — flat `25%`.
- **Wrath of the Berserker** — flat `30%`.
- **Strength coefficient** (`strPer10`, Setup panel) — genuinely contested between
  sources found via search (`10 Str = 1%` vs `10 Str = 0.8%`). It's exposed as a
  user-editable input rather than hardcoded specifically because of this. Worth
  nailing down definitively if you can find an authoritative source.

### Structurally absent, not just unverified
Paragon Legendary Node values and Glyph scaling aren't modeled at all — there was
never network access to pull them. If you want them in, decide first whether they
belong as new `STAT_DEFS` entries (additive bucket) or new `KNOWN_UNIQUES`-style
gear-bound multipliers, based on what the actual tooltip math turns out to be.

### Suggested first move
Start with Ramaladni's Magnum Opus since that's the page already in hand. Once you
've got the real formula, update `KNOWN_UNIQUES.rmo` in `index.html` (search for
`const KNOWN_UNIQUES`), change `prov` to `'game'` once it's confirmed, and update
the "Data confidence" disclosure text at the bottom of the page (search for
`Data confidence — what's verified`) to reflect it. Same pattern for each item
after that.

---

## 5. Loose end: GitHub Pages has two deploy mechanisms active

Pages was already live via the classic "deploy from a branch" method before this
session touched anything. This session *also* added `.github/workflows/pages.yml`
(Actions-based deploy) without realizing the classic one already existed — first
attempt failed outright, but a later push succeeded on both simultaneously. Both
currently report success on every push to `main`, which is redundant but not
visibly broken. If you have a spare minute: check **Settings → Pages → Source** in
the repo. If it says "GitHub Actions," the classic path is vestigial and you could
clean it up. If it says "Deploy from a branch," the Actions workflow is the
redundant one — safe to delete `.github/workflows/pages.yml` at that point, but
there was never enough certainty to do that blind, so it was left alone.

---

## 6. What NOT to reintroduce

- Don't split this back into "Basic Mode" / "Advanced Mode" — that was the original
  shape of the tool and it caused the engine and UI to disagree with each other.
  There is one build model now; keep it that way.
- Don't add a Unique's power as a flat Multipliers row disconnected from a gear
  slot (see §3).
- Don't shallow-copy `SAMPLE` or any nested build object — deep clone
  (`JSON.parse(JSON.stringify(...))`) or you'll reintroduce the mutation bug.
- Don't invent a number to fill a gap. If it's not confirmed, leave it at 0 and
  tag it `unverified` — that's the whole point of the provenance system.
