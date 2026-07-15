# Logan Clicker — Long-Term Progression & Rebalance Spec (v6 design)

Target: the existing v5 single-file game (HUD/scene shell, 15 buildings, 119 upgrades,
4 Logan tiers, 6 worlds, 18-node skill tree, Transcendence + perks). This doc is the
implementation spec for the v6 rebalance.

---

## 1. Core design diagnosis

The current game breaks at high lifetime earnings for one precise reason:

**Transcendence reward is polynomial and its payout is uncapped-linear.**

- `glasses = floor(cbrt(ever / 1e7))` — at 1e24 (Sp) lifetime this is ~464,000 glasses.
- Each glass gives a flat +2% CPS, so the permanent multiplier is `1 + 0.02·g` ≈ **×9,300**.
- Building costs are static (15 → 2.6e16). A ×9,300 head start crosses the entire
  building ladder in seconds. Post-reset play is a cutscene, not a game.

Secondary problems:

1. **Everything permanent stacks multiplicatively with no cap** — glasses ×
   achievements (+60%) × pets (×3.4) × portals (×7.6–11) × tiers × skills (×~4). One
   knob (glasses) dwarfs the rest, so no post-reset decisions exist.
2. **One reset layer, one currency, one use.** Glasses do nothing but passively
   multiply. There is nothing to *spend*, so a reset changes no rules.
3. **No tempo loop.** The only reset is the big one, so the player either grinds one
   giant run or trivializes everything.

Fix strategy (three rules applied everywhere below):

- **Gains are logarithmic** in currency (`(log10(x) − offset)^p`), so doubling a
  prestige payout costs ×300–×1,600 income, not ×8.
- **Payouts are spent, not stacked.** Prestige currencies buy automation, unlocks,
  and rule changes; passive multipliers get softcaps.
- **Three reset layers with different jobs:** tempo (Rebirth), meta (Transcendence),
  rule-change (Singularity Collapse).

---

## 2. New prestige layer design

| | 🔷 **Rebirth** | 👓 **Transcendence** (rework) | 🕳️ **Singularity Collapse** (new) |
|---|---|---|---|
| **Job** | Tempo loop | Meta-progression | Rule-changing |
| **Cadence** | every 15–45 min | every 2–6 h | every 2–5 days |
| **Resets** | Bank, buildings, shop upgrades, run buffs, combo | Rebirth layer + Logan tiers + shard balance + Forge + world resources/projects | Everything below + glasses, Legacy, skills, worlds/portals |
| **Persists** | Tiers, worlds, skills, glasses, Legacy, relics, achievements | Worlds stay unlocked; skills, glasses, Legacy, relics, achievements, Dark layer | Dark Logans, Singularity upgrades, relics, challenge completions, achievements, lifetime stats |
| **Currency** | Rebirth Shards 🔷 | Golden Glasses 👓 (new formula) | Dark Logans 🕳️ |
| **Buys** | **The Forge**: automation & tempo (auto-buyers, autoclick, warm starts, building momentum, upgrade memory) | **Legacy Tree**: offline caps, starting tiers, world attunement, challenge access, "keep Forge automation through Transcend" | **Singularity Core**: production *exponent*, relic slots, Mythic synthesis, new worlds, auto-Rebirth rules |
| **Unlocks at** | 1e6 run Logans | 1e9 lifetime (existing players already qualify) | 1e21 (Sx) lifetime |
| **New mechanic it opens** | Automation as gameplay | Challenges + relics (via Legacy) | Exponent scaling, Mythic Logans, Chronos & Mirror worlds |

Psychology: Rebirth is the snack (frequent, low-stakes, spends into quality-of-life),
Transcendence is the meal (plan a build, buy meta structure), Collapse is the new
game+ (changes what the numbers *mean*).

---

## 3. Exact formulas

All formulas are double-safe (log-based, no bignum needed below 1e308).

### Helpers

```js
// generic softcap: linear to cap, then power-p growth
function softcap(v, cap, p) { return v <= cap ? v : cap * Math.pow(v / cap, p); }

// log-gain: standard prestige payout shape
function logGain(x, offset, pow, scale = 1) {
  const L = Math.log10(x + 1) - offset;
  return L <= 0 ? 0 : Math.floor(Math.pow(L, pow) * scale);
}
```

### Rebirth Shards

```js
pendingShards = logGain(runLogans, 6, 1.5);   // run earnings, not lifetime
```

| Run earnings | Shards | Notes |
|---|---|---|
| 1e6 | 1 | first Rebirth (~25–40 min fresh) |
| 1e8 | 2 | |
| 1e10 | 8 | |
| 1e12 | 14 | doubling 14→28 needs 1e15.2 (×1,600 income) |
| 1e16 | 31 | |
| 1e20 | 52 | |
| 1e24 | 76 | |

"Time to reset" signal: show `+N 🔷` on the Rebirth button; the displayed *rate*
(`shards per hour this run`) starts falling once the run stalls — reset when pending
gain ≥ ~30% of current holdings.

### Golden Glasses (replaces `cbrt(ever/1e7)`)

```js
glassLevel = Math.floor(Math.pow(Math.max(0, Math.log10(ever + 1) - 6), 2) / 3);
gain = glassLevel - state.prestigeLevel;    // unchanged claim mechanic
```

| Lifetime | Glasses (total) |
|---|---|
| 1e9 | 3 |
| 1e12 | 12 |
| 1e15 | 27 |
| 1e18 | 48 |
| 1e21 | 75 |
| 1e24 | 108 |
| 1e30 | 192 |

Doubling glasses always costs ×~300+ lifetime income. The Sp-scale save that
currently holds ~464,000 glasses migrates to **~108** (see §9 migration).

### Dark Logans

```js
darkGain = logGain(ever, 20, 1.8);   // 0 until 1e21
```

| Lifetime | Dark Logans |
|---|---|
| 1e21 | 1 |
| 1e24 | 12 |
| 1e27 | 33 |
| 1e30 | 63 |

### Passive multiplier softcaps (the anti-runaway core)

```js
// glasses passive: gentle, log-shaped — the POWER is in the Legacy Tree, not here
glassMult = 1 + 0.5 * Math.log10(1 + glasses);          // 108 glasses → ×2.02

// total permanent product (pets × portals × tiers × skills × glassMult × achievements)
permanent = softcap(rawPermanent, 1e4, 0.5);            // ×1e6 raw → ×1e5 effective

// click power vs production (stops click builds from being strictly dominant or dead)
clickFromCps = softcap(cps * clickCpsPct, clickBase * 1000, 0.4);
```

### Prestige upgrade costs (all three shops)

```js
cost(n) = ceil(base * Math.pow(r, n));   // Forge r = 1.5, Legacy r = 1.7, Core r = 2.2
```

### Diminishing returns for stacked same-source bonuses

```js
// n copies of the same +b% effect: first 25 full strength, then decay
effStacks(n, cap = 25) = n <= cap ? n : cap + (n - cap) ** 0.6;
```

Applied to: tier-unit bonuses, achievement count, world-project stacks.

### Post-reset pacing guard

No artificial ramp. Recovery speed is governed by the softcapped permanent product:
target is **first 1e6 in ~2 minutes** after a Rebirth with automation, ~10 minutes
to first Rebirth after a Transcend. If ×1e5 effective permanent breaks that,
tighten the softcap exponent 0.5 → 0.45 (single tuning knob).

---

## 4. Logan tier system (Synthesis rework)

Rename buying tiers to **Synthesis**. Two changes make the ladder a system instead
of a stat-stick:

### Conversion consumes the tier below

```
Alien Logan  = 10,000 Logans          × 1.18^owned
Super Logan  = 25 Aliens + 2e6 Logans × 1.15^owned
Cosmic Logan = 25 Supers + 5e8 Logans × 1.15^owned
God Logan    = 25 Cosmics + 1e11 Logans × 1.15^owned
Mythic Logan = 10 Gods + 1,000 of every world resource   (unlocked via Singularity Core)
```

Sacrificing units creates a real decision (keep 25 Supers' bonus vs. one Cosmic),
and synthesis efficiency (skills, Zorgon, Legacy) becomes a build axis:
`consumed = ceil(25 * synthEff)` with synthEff floor 0.4 (min 10 consumed).

### Tier Ranks — choose-one perks

Every 10 units of a tier = 1 Rank; each Rank offers a **pick-one-of-two** perk
(respec on Rebirth):

| Tier | Option A | Option B |
|---|---|---|
| Alien | +15% production | Golden Logans 5% more frequent |
| Super | Clicks +25% | Crits +3% chance |
| Cosmic | Buffs last +15% | World resources +20% |
| God | All synthesis 10% cheaper | +1 relic swap per run |
| Mythic | **+0.01 production exponent each** (softcap 10) | +1 Forge automation slot |

Bonuses per unit use `effStacks` (§3). Gates already in the game (world unlocks,
tier-gated shop upgrades, `sk-d` branch) now key off Ranks instead of raw counts.

---

## 5. Skill tree layout

Tree becomes **run-scoped keystones on a permanent chassis**: base nodes are
permanent (bought with SP as today), but each branch ends in a **keystone pair —
mutually exclusive, chosen per run** (swappable at Rebirth, free). That single rule
creates builds without punishing experimentation.

SP sources unchanged (+1/Logan Level, +3/Transcend) plus +5 per completed challenge.

Six branches × 9–10 nodes (existing 18 nodes keep their ids and become the first
rows of branches 1–4):

**🖐 Momentum (active)** — Iron Finger → Thunder Thumb → Fist of Logan → Thousand
Hands → *new:* Combo Memory (combo decays instead of resetting), Overcharge (clicks
during Click Storm extend it +0.2 s), Blister Pact (clicks ×3, CPS −20%)…
**Keystones:** *Stormcaller* (Click Storms ×2 power, golden Logans click-only spawns)
vs *Zen Finger* (no combo; every click counts as 25-combo).

**🏭 Industry (idle)** — Blueprints → Mass Production → Logistics → Loganomics →
*new:* Assembly Lines (auto-buyer +1/s), Compound Interest (+0.5%/min idle, cap 30%),
Union Contracts (buildings 15% cheaper, clicks −50%)…
**Keystones:** *Gray Goo* (Auto-Pokers self-replicate 1/min) vs *Monolith* (your
highest building counts double, all others −25%).

**🧬 Synthesis (evolution)** — Xeno Studies → Evolution Theory → Portal Attunement →
Godspark → *new:* Cheap Genes (synthesis consumes 20% fewer units), Rank Scholar
(+1 perk option per Rank), Gene Bank (keep 1 tier unit of each through Transcend)…
**Keystones:** *Hive Mind* (tier bonuses use cap 40 in `effStacks`) vs *Apex* (only
highest tier gives bonuses, ×3).

**🌀 Wayfarer (worlds)** — *new branch:* Passport (world resources +25%), Twin Portals
(2 visit bonuses at once), Resource Rush (resources accrue offline), Deep Roots
(world projects survive Transcend)…
**Keystones:** *Nomad* (visit bonus ×2 but changes world every 10 min) vs *Homebody*
(pick one world: its bonus ×3, others −50%).

**🦋 Legacy (prestige)** — *new branch:* Quick Molt (Rebirth +10% shards), Glass
Forward (start runs with 5% of previous run's buildings), Echo Chamber (first golden
each run guaranteed Frenzy)…
**Keystones:** *Perpetual* (auto-Rebirth at chosen threshold) vs *Big Bang* (manual
Rebirths only, +60% shards).

**🎲 Chaos (high-risk)** — *new branch, unlocks at 25 glasses:* Unstable Isotopes
(buffs ×1.5 power, ×0.5 duration), Logan Roulette (5% of clicks give ×100, 5% give 0),
Entropy Tax (production +50%, a random building disables 60 s every 5 min)…
**Keystones:** *Singularity Gambit* (all bonuses ×2, bank wiped if it exceeds
1000× your CPS — forced spending) vs *Stasis* (immune to Chaos downsides, +25% flat).

---

## 6. Worlds redesign

Worlds gain **unique resources that accrue only while visiting** (rate scales with
synergy-tier count), spent on **world projects** (persist through Rebirth, reset at
Transcend unless Deep Roots). Parking location becomes a per-phase decision.

| World | Identity | Resource | Specialty | Synergy tier | Late-game role |
|---|---|---|---|---|---|
| 🌍 Earth | home / balance | ☕ Motivation | starter, click projects | — | Motivation feeds every other world's tier-2 projects |
| 🔴 Mars | war / crits | 🎖 War Medals | active-click builds | Super | crit-exponent project (crit mult +0.5/stack) |
| 🔵 Neptunia | frost / patience | 🧊 Cryo Cells | idle & offline | Alien | offline production ×2 projects |
| 🟠 Vulcanis | fire / volatility | 🔥 Ember Cores | buff economy, unstable | Cosmic | buff-chaining; Ember projects risk: −20% CPS while charging |
| 🟢 Zorgon-7 | xeno / synthesis | 🛠 Alien Alloy | conversion efficiency | Alien | synthesis discount stacks; Mythic component |
| 🟣 The Void | dark / prestige | 🕳 Void Fragments | prestige amplification | God | +shard and +glass gain projects; Dark Logan component |
| ⏳ Chronos *(new, Singularity Core 5 🕳️)* | time | ⌛ Sand of Logan | automation | Mythic | auto-Rebirth tuning, tick-rate +, offline everything |
| 🪞 Mirror *(new, Core 12 🕳️)* | inverse | 🪞 Shards of Self | challenge world | all | every buff inverted; completing runs here = challenge medals |

Unlock pacing: Mars → early run 1; Neptunia/Vulcanis → mid Transcend era (tier
gates as today); Zorgon/Void → late Transcend era; Chronos/Mirror → Collapse era.
Portal passive changes from ×1.5 each to **×1.35 each, softcapped inside the global
permanent product** — worlds matter through resources now, not raw multipliers.

---

## 7. Replayability systems

| System | What | Why it re-freshens resets | Ties into | Cadence |
|---|---|---|---|---|
| **Challenges** (Legacy Tree unlock, 15 👓) | 8 rule-broken runs: *Pacifist* (no clicks), *Minimalist* (≤3 building types), *Blackout* (no goldens), *Freefall* (costs grow 1.30×), *Mute Logan* (no speech/no ticker… and no achievements bonus), *Inverted* (combo counts down), *Solo World* (Earth only), *Glass House* (bank caps at 100× CPS) | forces different build/branch each attempt | reward = **relics** + 5 SP each | one per Transcend cycle |
| **Relics** (3 slots, +1 per Core purchase) | equipment-like modifiers chosen at Rebirth: *Bronzed Thumb* (crits pierce softcap), *Pocket Portal* (2 visit bonuses), *Shard Battery* (+1 shard/10 min), *Cracked Hourglass* (offline ×3, online −10%)… | loadout = run identity | earned from challenges & Mythic synthesis | swap each Rebirth |
| **Rotating mutators** (auto, seeded weekly from date) | one ambient modifier per real-week: "golden logans travel", "buildings cost Motivation", "combo cap 200"… | login variety without FOMO (purely cosmetic-positive) | none required | weekly |
| **Contracts** (Void projects) | opt-in run goals ("reach 1e12 in 20 min") for bonus shards | short-run skill tests | Rebirth economy | per run, optional |
| **Automation presets** | save/load Forge + keystone + relic loadouts as named builds | one-click re-speccing makes experimenting cheap | Forge | always |

---

## 8. Balance targets

| Milestone | Target |
|---|---|
| First Rebirth | 25–40 min into a fresh game |
| Steady Rebirth loop | 15–45 min; automation (Forge) earned by rebirth 3–5 |
| First Transcend | 4–8 h play, ~2–3 rebirths deep |
| Transcend loop | 2–6 h; each should fund 1–2 meaningful Legacy nodes |
| First Collapse | 3–7 days after first Transcend |
| Collapse loop | 2–5 days |
| Run "productive" window | a run should stall (income growth < costs) ~10–15 min after its last purchase spike — that stall is the reset cue |

Power budget at mid-game (post-3rd Transcend):
**~40% run** (buildings + shop upgrades) / **~25% Forge** / **~20% Legacy + glasses**
/ **~15% relics + ranks**. No single permanent source should exceed ×100 effective
before Collapse-era exponents.

Early-game skip rules: Rebirth skips ~0 content (automation *plays* it instead),
Transcend skips to ~first Rebirth in 10 min, Collapse deliberately restores the
full arc once (that's its charm) with exponent power making it 3–4× faster.

Old content stays relevant via: challenges re-imposing early constraints, world
resources requiring visits at every era, and Mythic synthesis consuming low-tier
units forever.

---

## 9. Implementation plan

### State split (the important refactor)

```js
state = {
  run:  { logans, totalLogans, buildings[], tiers{}, shopUpgrades:Set, buffs, combo, runTime },
  temp: { shards, forge:{ id:level }, worldRes:{ mars:0,... }, projects:{}, keystones:[], relics:[3] },
  meta: { glasses, legacy:{ id:level }, skills:[], dimensions:[], challengesDone:[], relicsOwned:[] },
  deep: { darkLogans, core:{ id:level }, mythics, collapses },
  life: { everLogans, totalClicks, records, achievements, playTime, options }
}
```

### Reset as data, not code

```js
const RESET_LAYERS = {
  rebirth:   { clears: ["run"],                 grants: s => logGain(s.run.totalLogans, 6, 1.5) },
  transcend: { clears: ["run", "temp"],         grants: s => glassLevel(s.life.everLogans) - s.meta.glasses },
  collapse:  { clears: ["run", "temp", "meta"], grants: s => logGain(s.life.everLogans, 20, 1.8) },
};
function doReset(layer) { /* snapshot grant → wipe listed slices via fresh factory → apply persists hooks → recalc */ }
```

Each slice gets a `fresh()` factory so "what resets" is auditable in one place.

### Node / upgrade schema (one shape for Forge, Legacy, Core, skills)

```js
{ id, name, icon, desc, branch, cost: n => base * r**n, max, req: ["id"], excl: "groupId",
  fx: { kind: value } }   // all effects resolved in recalc(), same as today
```

`excl` implements keystone mutual exclusion. `recalc()` stays the single pipeline;
add `softcap()` at the permanent-product step and `effStacks()` at per-source steps.

### World schema addition

```js
{ id, ..., resource: { id, icon, ratePerSec: s => base * (1 + synergyTier(s)) },
  projects: [ { id, cost, fx, persistsThroughTranscend?: bool } ] }
```

### Numbers

Plain doubles + the log formulas are safe to ~1e300; no bignum library needed.
Centralize every tuning constant in one `BAL = {...}` object at the top of the file
so pacing fixes are one-line edits.

### Save migration (v5 → v6) — non-negotiable rules

- Same `loganClickerSave` key; keep the v1–v5 loader chain.
- Map old fields into slices (`logans→run`, `skills/dimensions/glasses→meta`, …).
- **Glasses conversion:** `meta.glasses = glassLevel(everLogans)` (recompute from
  lifetime, ignore inflated stored value); grant a one-time "Founder's Relic"
  (+10% all production, relic slot 1) as compensation so the nerf lands as a trade.
- Grant retroactive first-Rebirth unlock if `everLogans ≥ 1e6` so existing players
  see the new loop immediately.

### Build order (each step ships playable)

1. State split + reset-as-data + migration (no visible change).
2. Glasses formula swap + softcaps + Founder's Relic.
3. Rebirth layer + Forge (the tempo loop fixes the "instant recovery" complaint immediately).
4. Synthesis consumption + Ranks.
5. World resources + projects.
6. Skill tree branches 5–6 + keystones.
7. Challenges + relics.
8. Singularity Collapse + Core + Chronos/Mirror.
