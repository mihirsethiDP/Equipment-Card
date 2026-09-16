# Handoff: Equipment Card & Plant Drill-down (Goal 13)

**Date:** 16 Sep 2026 · **Pilot:** 2 Oct · **Design owner:** Mihir · **Current version:** Plant Drill-down v3

---

## Overview

A single card component that describes any node in a water-treatment plant, reached by drilling down
**Plant → Unit process → Unit → Equipment**. The card carries identity, live sensor readings, the
work filed against that node, and exactly one call to action for the most pressing thing. At equipment
level the card **composes itself from the equipment's type** rather than being a different card per type.

Primary user is a plant operator on a phone, standing at the equipment. Secondary users (Lead,
Senior Lead, non-operating client) reuse the same card with a different lens — **not built yet**, see
*Known gaps*.

---

## About the design files

The files in `prototypes/` and the standalone bundle in this folder are **design references written in
HTML** — prototypes that show intended look and behaviour. They are **not production code to copy**.

The task is to **recreate these designs in the target codebase's existing environment** (React, Vue,
React Native, whatever the app uses) with its established component library, routing, state management
and i18n. If no environment exists yet, choose the appropriate framework and implement there.

Specifically do not port:

- The prototype's in-memory `N` node graph — replace with the real asset API.
- The `L` string tables — replace with the app's i18n layer (the prototype ships `en` + `hi` inline).
- Inline style strings assembled in JS — replace with the codebase's styling approach.
- The state chips and "jump to" row in the prototype header — these are **demo scaffolding** for
  reviewing states, not product UI. Delete them.

### Fidelity

**High-fidelity.** Colours, type, spacing, copy, severity vocabulary, block order and interaction
behaviour are all final and should be recreated faithfully. The exact pixel geometry of the equipment
illustrations is **low-fidelity placeholder** — see *Assets*.

---

## Files

| File | What it is |
|---|---|
| `plant-drilldown-v3-standalone.html` | **Current design.** Self-contained, opens offline in any browser. Start here. |
| `prototypes/plant-drilldown-v3.dc.html` | Source of the current design (needs `support.js` beside it). |
| `prototypes/plant-drilldown-v2.dc.html` | Previous version — introduced type profiles. Kept for diffing. |
| `prototypes/plant-drilldown-v1.dc.html` | First drill-down: levels + roll-up, before type profiles. |
| `prototypes/equipment-card-v2.dc.html` | Equipment card in isolation, with designed **empty / offline / failure** states. |
| `prototypes/equipment-card-v1.dc.html` | First equipment card. Historical. |
| `prototypes/support.js` | Runtime the `.dc.html` prototypes need. Not part of the deliverable. |

Each prototype file renders **two artboards side by side**: desktop 1440×900 (`#2a`) and mobile
390×844 (`#2b`). They navigate independently so two states can be compared at once.

---

## Information architecture

```
Plant  (Kanpur STP)
└── Unit process        (Preliminary, Primary, Secondary, Tertiary, Sludge Handling)
    └── Unit            (Screen chamber, Blower group IU-G3, Aeration basin 1, …)
        └── Equipment    (Blower BL-2, Tank EQ-1, DO probe DO-1, Fine screen FS-1, …)
```

### Ownership and roll-up — the core rule

Every issue, task and maintenance record is **filed against exactly one node** and **rolls up** to every
ancestor. It is never duplicated and never re-parented.

- `Secondary Treatment` owns *"DO below target across both aeration basins"* — it spans two basins and
  the blower group, so it cannot belong to one machine.
- `Blower group IU-G3` owns the *duty changeover log* — a group task, not a blower task.
- `BL-2` owns its own discharge-pressure issue and its own PM.

In the card's work area, items filed at the node you are on appear under **"Filed here"**, and items
inherited from descendants under **"From below"**, each tagged with its owning level and name
(`Equipment · Aeration Blower BL-2`) and, when expanded, a button that navigates to that owner.

**Rolled-up lists are capped** at `BELOW_CAP = 4` items from below, with `+N more from below` /
`Show fewer`. A real plant has hundreds of assets; the prototype's 4-row list is not representative.
Server-side pagination or a severity floor is required — see *Known gaps*.

> ⚠️ **Scope discrepancy to resolve before build.** The config model currently allows issue/trigger
> scoping at **equipment only**. The design requires plant, unit-process, unit/group **and** equipment.
> This blocks the process-level issue above. Flagged as an ADR in *Known gaps*.

---

## Card profiles — how the card varies by equipment type

A **profile** is a declaration per equipment type: an ordered list of anchor blocks. Tabs are constant.
This is the mechanism that replaces "a different card per type".

```js
PROFILES = {
  rotating:   { anchor: ['attributes','health','duty','links','sensors'], sensorsOpen: false },
  vessel:     { anchor: ['links','sensors'],                              sensorsOpen: true  },
  instrument: { anchor: ['attributes','health','sensors'],                sensorsOpen: true  },
  fixed:      { anchor: ['attributes','health','links','sensors'],        sensorsOpen: false },
  group:      { anchor: ['rollup'] }   // plant / unit process / unit
}
```

| Profile | Applies to | Rationale |
|---|---|---|
| **Rotating machine** | Blower, sludge pump, dosing pump, scraper drive, grit classifier, centrifuge | Has wear, so health and attributes lead. Duty strip when it sits in a duty group. |
| **Passive vessel** | Equalization tank, sludge holding tank, chlorine contact tank | **No health score and no attributes block** — nothing rotates, no wear signal to score. Leads with what feeds it / what it feeds, then its readings. A vessel essentially *is* its readings, so sensors default open. |
| **Instrument** | DO probe, level probe | Health + attributes, **no links block** — a probe reads one thing. |
| **Fixed plant** | Coarse/fine screen, diffuser grid | Same as rotating minus duty. |
| **Group** (levels) | Plant, unit process, unit | Three-number roll-up strip instead of asset blocks. |

**Block order is the profile's order** — `order:` on a flex column. A block renders only if the profile
declares it **and** the data exists (`!specs`, `!health`, `!sensors.length`, `dutyPeers < 2` all collapse
it). Nothing is hidden by hand.

> ⚠️ **"Instrument" and "Fixed plant" are not in the locked archetype set AR-1…AR-7.** Screens and
> diffuser grids genuinely do not fit the seven. Needs an ADR: archetypes 8 and 9, or a trait model.

---

## Screens / views

### 1 · Level view (plant / unit process / unit) — desktop

**Layout.** Two columns. Left canvas `flex:1`, right drawer fixed `452px` with
`border-left:1px solid #E1E3E5` and `box-shadow:-14px 0 34px rgba(15,35,55,.12)`.

Left canvas, top to bottom:

- **Breadcrumb bar** — `padding:14px 24px`, `border-bottom:1px solid #E1E3E5`,
  `background:rgba(255,255,255,.75)`. Crumbs `12.5px/600`, current `700 #161616`, ancestors `#525252`,
  separator `/` in `#B9C2C9`. Every crumb navigates. Right side shows `Open card` button only when the
  drawer is closed.
- **Level header** — `padding:18px 24px 10px`. Eyebrow: level label, `10.5px/700`, `letter-spacing:.09em`,
  uppercase, `#8D8D8D`. Title `19px/700 #161616`. Right-aligned composition line `12px/600 #525252`
  (`"3 units · 6 equipment"`, or `"Nothing mapped yet"`).
- **Child tile grid** — `repeat(auto-fill, minmax(228px, 1fr))`, `gap:12px`. Tile: white,
  `1px solid #E1E3E5`, `border-radius:11px`, `padding:14px 15px`. Tile with a Major item below it gets
  `1.5px solid #F0A9A9`. Contents: name `14.5px/700`; health badge top-right (equipment only); sub-line
  `11.5px #8D8D8D`; type chip (equipment only, `10px/700` uppercase, `#1A4639` on `#E5F6F1`); then an
  **attention line** — a severity dot plus plain words: `"1 issue · 2 tasks due"` or `"Nothing pending"`
  in `#8D8D8D`. Tiles navigate down.

**Empty unit.** A unit with no equipment mapped (Aeration basin 2) shows a dashed card:
*"No equipment mapped to this unit yet"* + why its readings arrive at process level + a
**Map equipment** action.

### 2 · Equipment view — desktop

Left canvas replaces the tile grid with the **equipment illustration** and its live reading tags,
plus the profile chip under it. Everything else is identical.

### 3 · The card (right drawer, desktop · bottom sheet, mobile)

Fixed **header** (never scrolls):

- Row 1: `EN / हिं` toggle · level chip · profile chip · `⋯` · `✕` (34×34 desktop, **44×44 mobile**).
- Title `21px/700` (mobile `19px`). Place line `12.5px #525252` — the ancestor path joined with ` › `.
- Status row: a state dot + status text `14px/600`, `font-variant-numeric:tabular-nums`; right side
  relative timestamp `11px/600 #8D8D8D`. Dot is `#198038` running, `#8D8D8D` for stopped / idle / standby.
- **Offline banner** when live data is down (see *States*).
- **Next-PM line** — `12px/600 #5A3E9B` on `#F6F2FC`, `1px solid #E4DAF5`, `border-radius:7px`.
  Reads `Next PM · bearing greasing · Due in 14 days`. **Suppressed when that PM is already the CTA**,
  so the card never says the same thing twice.

Scrolling **anchor blocks**, in profile order. Each is white with `border-bottom:1px solid #E1E3E5`,
`padding:13px 20px` desktop / `12px 16px` mobile. Section labels are `10.5px/700`, `letter-spacing:.08em`,
uppercase, `#8D8D8D`.

- **Attributes** — collapsed by default, header is a disclosure. Collapsed shows a summary of the first
  two values (`"Kaeser · EB 291 C"`). Expanded: desktop `1fr 1fr` grid of label/value pairs; mobile chips.
- **Health** — score `22px/700` tabular + `/100`, band chip, `Why?` link, then a 6px track
  (`#EAEDEF`, radius 3) filled `linear-gradient(90deg,#3DA385,#F1C21B)`. `Why?` discloses the **basis and
  confidence** in a `#F7F9FB` inset. Band label and colour derive from **one** threshold function —
  `≥78 Good #0E6027/#E9F5ED`, `≥68 Fair #8A6100/#FFF6DB`, else `Watch #B75000/#FFF0E4`.
  No score → `—` in `#8D8D8D` with band `Not enough data` and a basis saying when it fills in.
- **Duty group** — only when ≥2 peers carry a duty role. Tappable cells, 44px min-height, each showing
  short name + role (`lead` / `running` / `standby` / `new`); the current asset is outlined
  `2px solid #183650`. Mobile: collapsed, summary `"BL-2 · running · 4 in group"`.
- **Linked equipment** — `Upstream — feeds this` / `Downstream — fed by this`, each a row of chips.
  Chip: `#F7F9FB`, `1px solid #D6DEE5`, `border-radius:8px`, a severity dot for the linked asset's own
  state, name, then a `›` chevron in `#8D8D8D`. **36px desktop, 44px mobile.**
  Deliberately **not** action-coloured — it must read as navigation, not control (safety finding U-11).
  Footnote: *"Flow path from the plant schematic. Tap any asset to open its card."*
  Mobile: collapsed, summary `"3 · upstream & downstream"`.
- **Sensors** — disclosure whose default open state comes from the profile. Each row: severity dot, name
  `13px/600`, meta `11px #8D8D8D`, right-aligned value `14px/700` tabular + expected range `10.5px/600`.
  Rows separated by `border-top:1px solid #F0F0F0`, `padding:10px 0` desktop / `12px 0` mobile.
  Tones: `ok #198038/#161616`, `warn #FF832B/#B75000`, `stale #C9CDD1/#8D8D8D`, `off #C9CDD1/#8D8D8D`.
  A stopped pump reads `0.0 A · stopped`, never a bare zero.
  **Mobile: sorted worst-first and capped at 3**, with a summary line (`"2 need attention"` /
  `"all normal"`) and `+N more sensors` / `Show fewer sensors`.
  A reading that physically belongs to another asset is attributed — EQ-1's level reads `via LP-1` —
  so the same number is never double-counted on two cards.
- **Roll-up strip** (levels only) — three cells on `#F7F9FB`, `1px solid #E1E3E5`, radius 9: open issues,
  tasks due, of them preventive. Numbers `21px/700` tabular, coloured `#DA1E28` / `#8A6100` / `#8D8D8D`
  when zero. Below: `Worst SLA: 1 h 05 m left · 1 covered`.

**Work area** — three tabs, constant on every node:

| Tab | Contains |
|---|---|
| **Issues** | Unplanned work. Open first, then resolved. |
| **Tasks** | Planned work **including preventive**. Overdue and due first, then last 30 days. |
| **History** | Events for this node and everything under it. |

Tab: `flex:1 1 0`, `min-width:0`, 38px desktop / **44px mobile**, `13px/700`, active
`#183650` with `border-bottom:3px solid #3DA385`, inactive `#8D8D8D` transparent. Count badge
`10px/700` white on `#DA1E28` (issues) / `#183650` (tasks).

**Horizon strip** — Tasks tab only. `All · Today · Tomorrow · This week`, 36px pills, active
`#183650` white. Filters the same queue in place, never navigates. Empty window gets its own copy,
not the tab's empty state.

**Row** — white, `1px solid #E1E3E5`, radius 10, `padding:12px 13px` desktop / `13px` mobile.
A Major unclaimed item gets `1.5px solid #183650`. A claimed item gets `opacity:.82` and no emphasis
border. Title `14px/600` (history `13px`). Right column stacks up to two chips: severity, then
`Covered` or `Preventive`. Meta line `12px #525252` assembles `when · who`, plus
`Preventive · every 500 run-hours` for PM rows and `Suresh is on it` when claimed. Rows with detail
expand in place behind a `1px solid #F0F0F0` divider.

**Footer CTA** (pinned, never scrolls):

- Active: full-width button, `background:#183650`, radius 9, `padding:16px 18px`, `15.5px/700` white,
  label left, `→` right.
- Calm: `#F1F8F4` / `1px solid #CDE7D8` / `#0E6027` — *"✓ All normal — nothing needs you"*.
- Covered: `#EDF3FF` / `1px solid #C7D9FB` / `#0F4CC0` — *"Covered — nothing for you to start"*.
- Under every state, a `11.5px #8D8D8D` line explaining **why this is the CTA**. This is a requirement,
  not decoration — it makes the resolver checkable.

**`⋯` overflow menu** — 8 entries. Report an issue here · Assign a task · Tell supervisor ·
Mark maintenance mode · Bring back online · Open group control · Export this level · Manuals & drawings.
Entries with no producer this quarter render **disabled with the reason inline**
(`"Phase 2 — needs alert suppression and the safety co-sign"`). Backed by a full-bleed click-catcher that
dismisses on outside click.

### 4 · Mobile (390×844)

Same card, same block order, same profiles. Differences:

- The level list **is** the screen; the card is a sheet over it at `top:150px`, radius `20px 20px 0 0`,
  `box-shadow:0 -8px 28px rgba(10,25,40,.32)`, with a 38×4 grab handle.
- Navy header (`#183650`) holds **one** back target (`Back to Kanpur STP`, 36px) plus level label,
  profile chip and current name. No breadcrumb trail — it can't be tapped accurately.
- Below the tile list, an `Open this card` tile when the sheet is closed.
- **Progressive disclosure is the rule**: only what changes minute to minute is always visible —
  name, place, status, health, offline banner, next PM, the queue, the CTA. Attributes, linked equipment
  and duty group are collapsed behind one-line summaries; sensors capped at 3 worst-first.
- Every disclosure header and control is **≥44px**. Disclosure state resets on navigation.

---

## Interactions & behaviour

| Trigger | Result |
|---|---|
| Tap child tile | Navigate down one level; tab resets to Issues, horizon to All, disclosures collapse |
| Tap breadcrumb (desktop) / back (mobile) | Navigate up to that node |
| Tap linked-equipment chip | Navigate to that asset's card |
| Tap duty cell | Navigate to that peer |
| Tap row | Expand detail in place (only if it has detail) |
| Tap `Open <owner>` in an expanded rolled-up row | Navigate to the owning node |
| Tap tab | Switch queue, clear expansion and the show-all cap |
| Tap horizon pill | Filter Tasks in place |
| Tap CTA | Navigate to the owning node of the worst item |
| `Report an issue here` | Appends a Minor issue **owned by the current node**, jumps to Issues, toasts |
| `Assign a task` | Appends a task due today, jumps to Tasks, toasts |
| Disabled menu entry | Toasts `Not available yet — <reason>` |
| `✕` / `Esc` | Close the card; `Esc` closes an open menu first |
| Outside click on menu | Dismiss |
| `Retry` in the offline banner or failure row | Restore live data |
| `EN / हिं` | Switch language for **both** artboards, including generated copy |

Toasts: `#183650`, white `12.5px/600`, radius 9, pinned above the CTA, auto-dismiss **3200 ms**.

No animation is specified beyond default disclosure reflow. Transitions are the implementer's choice;
keep them under 200 ms.

---

## The CTA resolver

One button, chosen by rule. Order of precedence:

1. **Unclaimed open issue**, lowest `pri` wins (1 = Major, 2 = Minor).
2. **Unclaimed open task**, lowest `pri` (3 = due, 4 = PM).
3. If every open item is **claimed** → the **Covered** state, no button, and a line naming who holds it.
   Taking it over requires a supervisor.
4. If nothing is open anywhere below → the **Calm** state, worded per level
   (*"Nothing open across 14 assets below this level."*).

**Offline override.** When live data is down, issues are skipped entirely — they need live readings to be
trustworthy — and the resolver falls back to planned work, saying so:
*"Issues need live readings, so the card falls back to planned work."*

Verb is derived: `Start working — ` for issues, `Start task — ` for tasks, `Plan PM — ` for preventive.
Label truncates at the first ` — ` in the item title.

> The prototype's claim model is a flag plus a name. The real mechanic is **free pickup with a
> stand-down broadcast**, never deployed and untested (U-05).

---

## States to implement

| State | How to reach it in the prototype | Behaviour |
|---|---|---|
| **Normal** | default | As described above |
| **Covered** | jump `Covered · FS-1` | Covered chip, dimmed row, no CTA, supervisor note |
| **New asset** | jump `New · BL-4` | Health `—` / `Not enough data` with fill-in date; per-tab empty states; one history line |
| **No live data** | state chip `Live data` → `No live data` | Amber banner `#FFF8EC`/`#F3DDB6`/`#6B4C00` with outage age + Retry; all sensors stale with `last 09:12`; health held at **Paused** with its own basis; illustration dimmed to `.62`; a dashed load-failure row in Issues with Retry; CTA falls back a rung |
| **Health gated** | state chip `Health: shown` → `gated` | Health block removed everywhere, pending the spec contradiction below |
| **Part tags off** | state chip | Illustration part labels hidden — the Phase-2 tap-rate baseline |
| **Empty unit** | navigate to Aeration basin 2 | Dashed "no equipment mapped" card + Map equipment |
| **Empty horizon** | Tasks → Tomorrow on most assets | *"Nothing in this window. Switch to All to see the whole queue."* |

The three **state chips** in the prototype header are review scaffolding. In the product these states come
from the data layer.

---

## State management

Per artboard (desktop and mobile hold independent copies in the prototype; in the product there is one):

```
path            string[]   ancestor chain to the current node
tab             0..2       issues | tasks | history
horizon         0..3       all | today | tomorrow | week
expandedRow     string     "<tabKey><index>" or ""
drawerOpen      bool
menuOpen        bool
showAllBelow    bool       roll-up cap override
specsOpen       bool
whyOpen         bool       health basis
sensorsOpen     bool|null  null = inherit profile default
sensorsShowAll  bool       mobile 3-row cap override
linksOpen       bool       mobile only
dutyOpen        bool       mobile only
```

Global: `lang` (`en` | `hi`), `offline`, `healthGated`, `showParts`, `toast`.

Navigation resets `tab`, `horizon`, `expandedRow`, all disclosures and both caps.

### Data the card needs per node

```
id, type: plant|process|unit|equipment
profile: rotating|vessel|instrument|fixed     (equipment only)
name, statusText, updatedAt
duty: lead|running|standby|new                (optional)
attributes: [{label, value}]                  (optional)
health: {score, basis, confidence}            (optional)
links: {upstream: [id], downstream: [id]}     (optional)
sensors: [{name, value, expectedRange, state: ok|warn|stale|off, meta, viaAssetId}]
items: {issues: [], tasks: [], maintenance: [], history: []}
children: [id]
```

Work item:

```
severity: major|minor|due|overdue|resolved|done
priority: 1..4        drives the resolver
preventive: bool      → PM chip + cadence
cadence: string       "every 500 run-hours"
claim: {name, at}     → Covered
sla: string           countdown
title, meta, detail
```

**Maintenance folds into Tasks** at read time: a preventive record is a task with `preventive: true`.
Keep the schedule as a separate object on the equipment (cadence, basis, owner, editable by Supervisor);
what the operator sees is the **instance** it emitted. This is the Part-5 recommendation, implemented.

---

## Design tokens

**Colour**

```
Brand / structure
  navy            #183650   headers, primary CTA, active tab text
  forest teal     #1A4639   links, secondary accents
  brand teal      #3DA385   active tab underline, accents
  teal wash       #E5F6F1   type chips
Ink
  primary         #161616
  secondary       #525252
  muted           #8D8D8D
  disabled        #A8A8A8
Surface
  white           #FFFFFF   card and blocks
  card bg         #F7F9FB   work area, insets
  page            #F2F7FB   canvas
  chip bg         #F4F7FA   icon buttons, chips
  action bg       #F1F5F9   secondary buttons
Line
  border          #E1E3E5
  divider         #F0F0F0
  strong          #D6DEE5
  dashed          #C9CDD1
  crumb sep       #B9C2C9
Severity  (text | bg | border)
  major/overdue   #DA1E28 | #FDECEC | #F7C8C8
  minor           #B75000 | #FFF0E4 | #FBD3B4
  due             #8A6100 | #FFF6DB | #F1DFA6
  resolved        #0E6027 | #E9F5ED | #CDE7D8
  covered         #0F62FE | #EDF3FF | #C7D9FB
  preventive      #5A3E9B | #F2EDFB | #DDD0F2
  done            #525252 | #F1F3F4 | #E1E3E5
Sensor / status dots
  ok              #198038
  warn            #FF832B
  stale / off     #C9CDD1
Health fill       linear-gradient(90deg, #3DA385, #F1C21B)
Banner            #6B4C00 | #FFF8EC | #F3DDB6
Illustration      #8FA2B0 stroke · #B8C4CD base · #CFDAE2 / #E5EDF2 / #C3CFD9 fills
```

**Type** — `IBM Plex Sans`, `IBM Plex Mono` (placeholder captions only),
`Noto Sans Devanagari` (Hindi). Weights 400 / 600 / 700.

```
10      badge count
10.5    section label (.08em, uppercase), type chip (.06em), spec label (.06em)
11      timestamp, owner tag, sensor meta
11.5    band chip, "Why?", footnote, horizon pill
12      row meta, crumb, composition, banner
12.5    crumb, link chip, summary, disclosure label, toast
13      tab, history title, spec value, menu item
13.5    spec value, status (mobile), self-card tile
14      status, row title, sensor value, calm CTA
14.5    tile name, calm CTA (mobile)
15.5    CTA label
17      mobile level title
19      level header (desktop), card title (mobile)
21      card title (desktop), roll-up number
22      health score
```

All numerics use `font-variant-numeric: tabular-nums`. Minimum body size **11px**; nothing smaller.

**Spacing** — 2 · 3 · 4 · 5 · 6 · 7 · 8 · 9 · 10 · 11 · 12 · 13 · 14 · 16 · 18 · 20 · 24 px.
Block padding `13px 20px` desktop / `12px 16px` mobile. Row gap 8px. Chip gap 6–7px.

**Radius** — 3 (bars, part tags) · 4 (band chip) · 5 (owner tag) · 6 (spec chip) · 7 (icon button,
pill, secondary button) · 8 (link chip, duty cell, inset) · 9 (roll-up cell, CTA, toast) ·
10 (row) · 11 (tile) · 12 (dashed panel) · 14 (reading tag) · 20 (status chip) · 50% (dots) ·
`20px 20px 0 0` (mobile sheet) · 44 (device bezel).

**Shadow**

```
drawer        -14px 0 34px rgba(15,35,55,.12)
mobile sheet   0 -8px 28px rgba(10,25,40,.32)
menu           0 14px 34px rgba(15,35,55,.22)
toast          0 10px 26px rgba(15,35,55,.30)
reading tag    0 2px 6px rgba(20,40,60,.10)
```

**Targets** — desktop controls 34–38px; **mobile everything ≥44px**.

---

## Assets

- **Equipment illustrations** are CSS-box placeholders assembled from absolutely positioned `div`s —
  four variants (blower cutaway, tank section, probe in liquid, bar screen) keyed off the profile's
  `art` field. They are **placeholders**. Replace with real OEM artwork or SVG; keep the
  reading-tag overlay and part-label positions as the contract.
- **Part labels** (air filter · belt · impeller) are non-tappable in Phase 1 by design — tap rate on
  them is the Phase-2 baseline measurement. Do not make them interactive yet, and do not remove them.
- **No icon set.** The prototype uses text glyphs: `⋯ ✕ → ↑ ▲ ▼ › ● ◌ ▲ ✓ /`. Swap for the codebase's
  icon library.
- **Fonts** load from Google Fonts in the prototype. Use the app's font pipeline.

---

## Known gaps — decide before or during build

**Blocking ADRs**

1. **Issue/trigger scope above equipment.** Config allows equipment only; the design needs plant,
   process, unit/group and equipment. Also: is a sensor a fifth scope, or is a sensor fault simply the
   diagnostic behind any alarm? And what does `Start working` open at a level with no equipment?
2. **Archetype set.** `instrument` and `fixed` are not in AR-1…AR-7. Archetypes 8 and 9, or traits?
3. **A probe is a sensor.** The hierarchy is Equipment → Part → Sensor → Reading, so giving DO-1 its own
   card, tasks and health puts one object at two levels. Calibration tasks are real and need a home:
   a sensor-health sheet off the parent's Sensors block, or promote instruments as a deliberate
   exception. **The prototype implements the exception** — treat it as a proposal, not a decision.
4. **Level count.** "Unit" currently means a control group, a vessel **or** a room. Multi-plant adds a
   level above. Nobody has asked an operator whether "Blower group IU-G3" is a place they believe exists.
5. **Health score.** The PRD puts health in Phase 2; the phased plan puts it at P5 with no producer this
   quarter. The two specs contradict. The `healthGated` flag exists so health can ship dark.
6. **Operator-name visibility per role** on history rows — bears on whether operators report honestly.
7. **Addendum A** (PPM config: work-type field, meter basis) — proposed, not approved.

**Not built, by design**

- **Supervisor / Lead lens** — approvals rung, assign, reassign, remind, maintenance mode with co-sign,
  audit log, two co-equal primaries on an unclaimed emergency. Waiting on the roles model.
- **Senior Lead and client lenses** — no fleet view, no compliance or outlet quality, no cost/energy
  framing, no photo evidence on resolved rows, no FM window to add context before an owner sees an item.
- **Alert → card entry.** The design assumes the app is already open. The operator's real journey starts
  in WhatsApp; the alert-to-card hop is unmeasured and belongs in the same instrumentation.
- **No help path, no "the alarm is wrong", no "I know the cause".** `Start working` assumes the operator
  accepts the framing. The relevance check is the one behaviour both field rounds agree on.
- **Roll-up scale.** Uncapped collection from all descendants; 4-item cap is a UI band-aid. Needs a
  server contract (severity floor, pagination) before a 200-asset plant.
- **Cross-reference between dependent items.** BL-2's pressure issue and Secondary's DO issue are
  causally linked and say so in prose, but the rows do not reference each other structurally.
- **Shift lens.** Deliberately day-based: Phase 1 has no shift schedule or roster data. If shift windows
  are added they must be display-only buckets — nothing routing or claim-pooling through them.
- **Audio / video affordance**, despite every persona preferring video.

**Instrument at pilot:** card opens, entry source, CTA state at open, taps per tab, seconds from
card-open to CTA tap, and seconds from alert receipt to CTA tap.

---

## Sample data in the prototype

Kanpur STP · 5 unit processes · 7 units · 16 equipment, all four profiles represented, in **both
languages**. Notable nodes for testing:

- `BL-2` blower — Major unclaimed issue, task due 16:00, PM in 14 days, one stale sensor. The busiest card.
- `FS-1` fine screen — **claimed** issue → Covered state.
- `BL-4` blower — commissioned 6 days ago → no-health state.
- `EQ-1` tank — vessel profile, PM schedule, level attributed `via LP-1`.
- `SHT-1` tank — vessel profile with **no** preventive schedule.
- `DO-1` probe — instrument profile, calibration **17 days overdue**, and it sits under the same process
  as the DO issue. The causal pairing the roll-up is meant to surface.
- `Aeration basin 2` — unit with no equipment mapped.
- `Secondary Treatment` — the process-level issue that the config model cannot currently express.
