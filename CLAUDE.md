# CLAUDE.md
## What this is
A household pay planner for a two-income couple in Dunedin, NZ. Andrew is on a fixed
salary; Hollie is a nurse on shift work at Te Whatu Ora (Southern) whose pay varies
enormously with which shifts she picks up. The tool exists to answer one
question: **which shifts are worth adding, after tax?**
The whole app is `index.html`. That is not a starting point to be improved on — it is
the design.
## Hard constraints
Do not do any of the following without the owner explicitly asking:
- **Do not split the file.** CSS, JS, icon and manifest are inlined deliberately so the
  thing is one artefact that can be emailed, dropped on any host, or opened from disk.
- **Do not add a build step, bundler, framework, or package.json.** No React, no Vite,
  no TypeScript, no Tailwind. It is vanilla ES5-flavoured JS and hand-written CSS.
- **Do not add dependencies.** The only external request is the Archivo webfont from
  Google Fonts, and the page degrades cleanly to system fonts if it fails.
- **Do not add analytics, telemetry, or any network call.** All state is `localStorage`
  on the user's own device. Nothing leaves the phone. Keep it that way.
- **Do not change the numbers in the pay model** to make output "look right". They are
  calibrated against real payslips — see below.
- **The names are deliberate.** Person labels are `Andrew` (salary) and `Hollie` (shifts).
  The owner chose to use first names knowing the repo is public. Don't anonymise them
  back to `Salary`/`Shifts`; don't add surnames or anything more identifying.
- The repo must stay **public** — GitHub Pages from a private repo needs a paid plan.
  Don't "helpfully" flip it private and break the deploy.
## Architecture
Single file, four layers, in this order inside `<script>`:
1. **Pay engine** — `shiftGross()`, `paye()`, `packet()`. Pure functions, no DOM.
2. **State** — `S` object, loaded from / saved to `localStorage` key `fortnight`. The
   roster is a bare fortnight: day index 0..13, Monday of week 1 = 0. `shifts`, `half`
   and `stat` are all keyed by that index. There are no calendar dates anywhere.
3. **Render** — `render()` rebuilds the summary, roster, marginal list, chart and
   ledger from `S` on every change. Cheap enough; don't optimise it into diffing.
4. **Wiring** — event handlers at the bottom.
`render()` recreates the roster DOM each call. Anything transient (the tap animation)
must be tracked in a variable outside the DOM — see `lastPop`.
## The pay model — calibrated, not guessed
Reverse-engineered from three real payslips (period ending 14 Jun, 28 Jun, 12 Jul 2026).
Every rule below reproduces those payslips to the cent. If you change one, re-run the
regression figures at the bottom of this file.
| Rule | Value | Evidence |
|---|---|---|
| Base rate | $55.04/hr | stated on all three slips |
| Paid hours per shift | 8.00 | ordinary hours of 24.00 and 32.00 are clean multiples; the 1 Jun shift splits 1.25 + 6.75 = 8.00 |
| Night loading | **+25% on hours worked 20:00–06:00, weekdays only** | a late shift touches 20:00–23:00 = 3 hrs × 25% × $55.04 = **$41.28**, the exact figure on all five "NIGHT RATE" lines |
| Weekend | time and a half (+50% on hours falling on Sat/Sun) | "W/E SHIFT" line = 1.25 hrs × 50% × $55.04 = **$34.40** |
| Stacking | weekend and public-holiday rates **replace** night loading, never stack | the Sun→Mon 1 Jun shift shows weekend and stat lines but no night line |
| Public holiday | double time (+100%) plus a day in lieu | "Penal 1 STAT" 6.75 hrs × 1.0 on top of ordinary |
| Short late | 14:40–19:00, 4.33 paid hrs, **no night loading** (ends before 20:00) | inferred, not payslip-confirmed |
The counter-intuitive result that matters: **the "NIGHT RATE" lines are not night
shifts.** They are the evening tail of afternoon shifts. Don't "fix" this.
Second result worth preserving: on a weekend, time of day makes no difference to pay —
early, late and night all gross $660.48 — because the weekend rate replaces the night
loading.
### Shift definitions
```
early  07:00–15:00   8.00 paid hrs
late   14:40–23:00   8.00 paid hrs   (the 20 min gap is an unpaid break)
night  22:45–06:45   8.00 paid hrs   (spans midnight — split per calendar day)
short  14:40–19:00   4.33 paid hrs   (long-press the Late box)
```
`shiftGross()` splits a shift into per-calendar-day segments and applies loadings per
segment. This is why a Friday night shift ($643.28) beats a plain weekday night
($540.08) — 6.75 of its hours land on Saturday. Keep the segment logic.
Loadings are computed on **clock** hours in the window; ordinary pay uses **paid**
hours (8.00). That asymmetry is deliberate and is what matches the payslips.
## Tax constants — IRD 2026–27
```
brackets  10.5 / 17.5 / 30 / 33 / 39 %  at  15,600 / 53,500 / 78,100 / 180,000
ACC earners' levy   1.75%, capped at $156,641 of earnings
IETC                $520, full to $66,000, abating 13c/$ to zero at $70,000 (ME code)
student loan        12% above $928.00 per fortnight ($24,128/yr ÷ 26)
KiwiSaver default   3.5% employee (rose from 3% on 1 Apr 2026)
```
PAYE annualises each pay period independently (`gross × 26`), which is what IRD's tables
do. A lumpy fortnight is therefore over-deducted and IRD squares it up at year end; the
page used to warn about this and the owner removed the note. Don't "correct" the
calculation to a smoothed annual one — it would stop matching payroll.
These rates change on 1 April. When the 2027–28 year starts, check IRD and update
`BRACKETS`, `ACC_RATE`, `ACC_CAP`, `SL_ANNUAL`.
## No calendar — deliberately
The page models **one generic fortnight**: Week 1 and Week 2, Mon–Sun, no dates. The
owner removed the date picker, the fortnight-stepping arrows and the hardcoded public
holiday table on 6 Sep 2026 because the only question is "what will *this* fortnight
pay", never "plan several periods". Don't bring dates back.
Consequences:
- Weekend/weekday is `idx % 7 >= 5`. A Sunday-night shift spills onto index 14, which
  is treated as a plain weekday (no stat, no weekend) — same as the Monday it would be.
- Public holidays can't be auto-detected. The user taps a day label to mark a stat day
  (`S.stat[idx] = 1`); tapping again clears it.
- Pay periods still run Monday–Sunday in reality, which is why the grid starts on Mon.
`load()` still understands saves from the earlier dated version (`start` + ISO-keyed
maps) and folds that fortnight's 14 days onto indexes 0..13 via `migrate()`. That path
can be deleted once the owner's phones have loaded the page once after the change.
## Verifying changes
There is no test runner. Verify two ways, both cheap:
**1. Regression the pay engine headlessly.** Stub the DOM, load the `<script>` body in
Node, and assert:
```
paye(1885.12, true) → 300.89   (payslip actual 300.88)
paye(1843.84, true) → 292.94   (payslip actual 292.92)
paye(3446.88, true) → 808.46   (payslip actual 808.44)
packet(1885.12,true,true,.035).net → 1403.40  (payslip actual 1403.43)
weekday:  early 440.32   late 481.60   night 540.08   short late 238.32
Saturday: early 660.48   late 660.48   night 660.48   short late 357.48
Friday night 643.28      public-holiday late 880.64
```
Cents-level drift is expected (payroll rounds per-table); dollars-level drift means
something broke.
**2. Render it in a real browser before claiming a layout works.** Chrome is available
via the puppeteer bundled with `@mermaid-js/mermaid-cli`. Screenshot at 390px and 760px,
and assert no element's `getBoundingClientRect().bottom` exceeds its parent card's.
Reasoning about CSS without rendering has already produced two shipped bugs here.
## Bugs already found — don't reintroduce
- **CSS class collision.** The totals block was given `class="tot-block key"`, but `.key`
  was already the chart legend's swatch class with `height:3px`. The block collapsed to
  15px and its children spilled over the next card. Legend styles are now scoped to
  `.legend .key`. **Namespace new classes.**
- **`var` hoisting.** The fortnight anchor was a `var` assigned after `load()` ran, so it
  was `undefined` at startup and the page failed to load cold. It's a function
  (`anchor()`) now — and `anchor()` itself is gone with the calendar, but the lesson
  stands: `TEMPLATE` is declared *above* `DEF` because `load()` reads it at startup.
  Watch initialisation order — `S = load()` runs early.
- **`:first-of-type` on a mixed-element parent.** `.tot-row:first-of-type` matched nothing
  because `.tot-lab` is the first `div` sibling. Use an explicit class (`.big`).
- **Mixing periods in one line.** A line once read "$7,697 gross · $137,616 a year",
  putting a fortnightly figure beside an annual one. Always label the period.
## Soft assumptions — confirm against future payslips
- No plain **early shift** and no ordinary **Saturday** appears in the three payslips, so
  the weekend rule rests on a single $34.40 line.
- The night window is modelled as **20:00–06:00**. A 20:00–08:00 window fits the data
  equally well *if* early shifts are exempt. **If an early shift ever shows a $13.76
  night line, the window is the wider one** and early shifts are worth more.
- The **short late** is paid its full 4.33 clock hours with no unpaid break. Unconfirmed.
  If a payslip shows 4.00 or 4.25, change `phHalf`.
- Ignores annual/sick/study leave, overtime past 80 hrs/fortnight, on-call, PDRP
  allowances.
Everything above is also surfaced to the user in the Assumptions panel. Keep that panel
honest — it is the thing that makes the tool trustworthy.
## Deploy
GitHub Pages, `main` branch, `/` root. `.nojekyll` is present. No Actions workflow.
```
git add -A && git commit -m "..." && git push
```
Pages rebuilds in ~1 minute. Don't rename the repo: `localStorage` is keyed to the
origin, so a URL change silently wipes the user's saved roster and rates.
