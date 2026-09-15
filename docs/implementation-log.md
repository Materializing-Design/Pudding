# Implementation log

A record of how Pudding was built and why it works the way it does — decisions, the
reasoning behind them, and the ones that were reversed. Written in the spirit of MDM:
the reasoning is the part that usually goes missing, so it is written down here rather
than reconstructed later from diffs.

Built in a single session on **2026-08-20**, against a repository that already held a
folder of dated build snapshots and little else.

---

## 1. Serving many builds from one Pages site

**Problem.** `builds/` held thirteen dated folders, each a complete static build of a
game, ~78MB in total. GitHub Pages serves one site per repository; the need was to serve
every build from that one site and link them from a front page.

**Decision.** A generated root `index.html` linking to `builds/<date>/`, with Pages
serving the repository root.

Checking the builds first settled the main risk: every asset reference in them was
already relative, so they would work unchanged under the `/<repo>/` subpath. Had they
used absolute paths, every build would have needed rewriting.

Added `.nojekyll` at the same time — Pages otherwise runs Jekyll over the upload, which
ignores `_`-prefixed paths and can mangle files inside build folders.

## 2. Reading the design tokens instead of guessing them

**Reversed.** The first version was styled from an impression of the Materializing
Design site: Spectral and Atkinson Hyperlegible, a warmer coral, an invented dark mode.
It was rejected — *"the aesthetics do not match the materializing design page."*

**Correction.** Reading `globals.css` and `layout.tsx` in the MatDes-Website repository
took one tool call and produced the real system: coral `#E95F58`, ink `#2C2C2A`,
parchment `#F1EFE8`, stone `#8A8A8A`, background `#EEE`, JetBrains Mono for headings,
Archivo for body, and **no dark mode at all**. The logos were copied from that
repository's `public/` rather than approximated.

**Why it matters beyond the colours.** Guessing produced something that looked plausible
in isolation and wrong next to the real thing. The tokens were a file away the whole
time.

## 3. From cards to a plain list

The first list was a card grid modelled on the MatDes Design Archive, complete with a
per-card description. With thirteen builds differing only by date, every card carried
the same sentence — filler dressed as content.

Cut to a plain rule-separated list of hyperlinks: date on the left, folder name on the
right. The nav bar went too, since a single-page site has nothing to navigate to. Links
were pointed at `builds/<name>/index.html` explicitly rather than the directory, so they
don't depend on directory-index behaviour.

## 4. Copy moved into `config.json`

Project name, author, and description were lifted out of the template into
`config.json`, so a designer can change the words without touching JavaScript.

`name` and `author` are required — missing either exits 1 rather than rendering blanks.
`description` is optional; omitted or empty, its paragraph is left out entirely rather
than leaving a gap. Values are HTML-escaped, so an `&` in a title cannot break the page.

## 5. Confirming the index was already generative

A request to "make it generative" turned out to describe what the script already did —
the confusion was that `index.html` is committed, which makes the entries *look*
hardcoded. Demonstrated by adding `builds/2099-01-01/`, regenerating (14 builds),
removing it, regenerating again (13).

**What was actually missing** was automation: someone had to remember to run the script.
Added `.github/workflows/pages.yml`, which regenerates and deploys on every push. Adding
a build became: drop the folder in, commit, push.

**Kept `index.html` committed** rather than ignoring it, despite it being build output.
If Pages is set to "Deploy from a branch" — an easy mistake — a committed index still
serves a working site, where an ignored one serves a 404 with no obvious cause. It also
lets someone clone and open the page with nothing installed.

## 6. More than one build per day

**Problem.** The folder name was the identity, and `YYYY-MM-DD` can only express one
build per day.

**Options.** A sidecar metadata file; nested `YYYY-MM-DD/HHMM/` folders; or a longer
folder name.

**Chose the folder name:** `YYYY-MM-DD-HHMM`, with `-HHMMSS` for collisions within a
minute and `T` accepted as a separator. Nothing to migrate, plain-date folders keep
working, and the name stays the identity.

Sorting moved from string comparison to parsed timestamps, so mixed naming orders
correctly. Validation was tightened to round-trip through `new Date`, rejecting
`2017-02-30` and `2017-07-05-2570` rather than listing them under a silently wrong date.

## 7. The sequence-number discovery

**The near miss.** Before rolling the new generator out to the repositories generated
from this template, a survey of what they actually contained turned up three —
`Pudding-AGP-Chess`, `Pudding-AGP-Competition`, `Pudding-AGP-Bitsy-Demake` — already
using a different convention for same-day builds: `2019-04-04-1`, `2019-04-04-2`.

The new parser required four digits for `HHMM`. Those six folders would have been
**dropped from their sites**, announced only by a `Skipping` line in an Actions log
nobody reads.

**Fix.** The parser now accepts a one-or-two-digit sequence suffix alongside times,
rendered as `#1`, `#2`. The digit count disambiguates: four or six digits is a time, one
or two a sequence. Checked against all **74 distinct folder names across the 19
repositories** — every one parses, versus six rejections before.

**The lesson worth keeping:** the naming rule was written from the template's own
contents, but the template's contents were not the whole population. Surveying the real
data before shipping caught what testing against local folders never would have.

## 8. Details that only surfaced by measuring

- **A 768px footer overflow.** The SSHRC and Concordia logos sit side by side at
  ~565px; with the link columns beside them that exceeds a 768px viewport. Found by
  querying `scrollWidth` against `innerWidth` across breakpoints through Chrome's
  DevTools protocol, not by looking — headless screenshots at a given `--window-size`
  gave misleading mobile renders, suggesting breakage at 390px that did not exist and
  hiding the real problem at 768px.
- **"1 builds · 2026-08-20 to 2026-08-20".** Invisible while thirteen builds existed,
  obvious the moment the template was reduced to one. Both the count and the range now
  collapse to singular forms — a template's default state is one build, so that is the
  state to get right.

## 9. The placeholder build

The template ships `builds/2026-08-20/`, a single self-contained HTML file that is
simultaneously the example build and the tutorial: it opens by pointing out that the
reader arrived via a link generated from its own folder name, then covers setup, naming,
the rules a build must follow, and how to delete itself.

It deliberately obeys its own advice — no external files, and its one link back to the
index is relative. Documentation that breaks the rule it is teaching is worse than none.

---

## Open questions

- **Vendored generator vs reusable workflow.** Each generator change currently has to be
  copied into every generated repository. Publishing it as a reusable workflow
  (`uses: Materializing-Design/Pudding/.github/workflows/build.yml@v1`) would make
  changes propagate on each repository's next push, at the cost of a dependency on this
  repository. Pinning a tag keeps the timing controlled. Not done yet.
- **Copy drift downstream.** Some generated repositories may still carry the template's
  placeholder `config.json` text. Not audited.
