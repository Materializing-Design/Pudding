# Architecture

Pudding is a static site with a build step measured in milliseconds. There is no
framework, no package manifest, and no dependency tree — one Node script using only the
standard library turns a folder of build snapshots into a linked index.

## The shape of it

```
config.json          ──┐
                       ├──►  scripts/generate-index.mjs  ──►  index.html
builds/<date>/       ──┘
builds/<date>/index.html   ← linked from the generated index, never read by it
```

Two inputs, one output. `config.json` supplies the words; the folder names under
`builds/` supply the list. Everything else on the page is a constant inside the script.

| Path | Role |
|---|---|
| `config.json` | Project name, author, description |
| `builds/<name>/` | One self-contained build snapshot, each with its own `index.html` |
| `scripts/generate-index.mjs` | The generator — template and logic in one file |
| `index.html` | Generated output. Never edit by hand |
| `assets/` | Logos used by the generated page (not by builds) |
| `.github/workflows/pages.yml` | Regenerates and publishes on push |
| `.nojekyll` | Stops Pages running Jekyll over the build folders |

## The generator

`scripts/generate-index.mjs` is an ES module run as `node scripts/generate-index.mjs`.
It resolves paths relative to its own location, so the working directory doesn't matter.

**It overwrites `index.html` from scratch.** It never reads the existing file: it builds
the whole page as one string and writes it with `writeFileSync`, truncating whatever was
there. So the output is deterministic — the same inputs produce a byte-identical file,
and re-running when nothing has changed leaves no diff. It also means any hand-edit to
`index.html` is destroyed on the next run, including the run the deploy workflow
performs on every push.

The page's CSS lives inside the template literal and is emitted inline. One output file,
no stylesheet to serve, no asset pipeline. The palette and fonts mirror the main
Materializing Design site — coral `#E95F58`, ink `#2C2C2A`, stone `#8A8A8A`, background
`#EEE`, JetBrains Mono for headings and Archivo for body — and like that site it is
light-only.

Values from `config.json` are HTML-escaped on the way in, so an `&` or `<` in a
description can't break the page.

## Build folder naming

A directory under `builds/` is listed only if its name parses as a date, optionally
carrying a time or a sequence number:

```
YYYY-MM-DD              2017-07-02          → July 2, 2017
YYYY-MM-DD-HHMM         2017-07-02-1430     → July 2, 2017 · 14:30
YYYY-MM-DD-HHMMSS       2017-07-02-143012   → July 2, 2017 · 14:30:12
YYYY-MM-DD-N            2017-07-02-2        → July 2, 2017 · #2
YYYY-MM-DDTHHMM         2017-07-02T1430     → July 2, 2017 · 14:30
```

The digit count disambiguates the suffix: four or six digits is a clock time, one or two
is a sequence number. Both forms exist across the repositories generated from this
template, so both are supported — see the [implementation log](implementation-log.md)
for how that came about.

Parsing is deliberately stricter than the pattern. `new Date` is asked to round-trip the
value, which rejects names the regex alone would admit — `2017-02-30` rolls over into
March, `2017-07-02-2570` is not a time — so a typo is skipped rather than listed under a
silently wrong date.

Times are treated as UTC end to end and displayed exactly as written. No timezone
conversion happens, because the folder name records when the build was made, not an
instant to be re-interpreted in the reader's locale.

## Ordering and labelling

Builds sort newest-first by parsed timestamp, not by string comparison, so mixed naming
stays in true chronological order. Ties break on sequence number, then on name. A folder
with neither time nor sequence counts as that day's earliest build, which is why it
sorts below numbered and timed siblings from the same day.

The summary line above the list collapses to fit what's there: `1 build · 2026-08-20`
for a fresh repo, `8 builds across 4 days · 2016-12-28 to 2017-07-04` when several
builds share days.

## Failure behaviour

The generator is loud about what it skips and refuses to produce a misleading page:

| Condition | Behaviour |
|---|---|
| Folder name doesn't parse | Skipped, name reported on stderr |
| Folder has no `index.html` | Skipped, name reported on stderr |
| `config.json` missing `name` or `author` | Exit 1, nothing written |
| No usable build folders at all | Exit 1, nothing written |

Skips warn and continue, so one malformed folder can't take down a whole site. But
because a skip is easy to miss in CI logs, treat any `Skipping` line as something to
fix — a build that is present on disk and absent from the index looks identical to a
build that was never added.

## Contracts a build must honour

The generator links to `builds/<name>/index.html` and otherwise leaves builds alone. For
that link to work:

- **The folder must contain `index.html`.** That is the entry point.
- **Every path inside must be relative.** `css/style.css`, never `/css/style.css` — the
  site is served from a repository subpath, so a leading slash resolves against the
  domain root and the build loads blank.
- **Each build must be self-contained.** Nothing referenced outside its own folder, so a
  snapshot keeps working years later however the rest of the repository changes.
- **Builds are immutable.** The point of the archive is what the project actually was on
  that day; editing a snapshot destroys the evidence.

## Template inheritance

Repositories generated from this template share no git history with it, so updates are
copied rather than merged. The split that makes that safe:

- **Template-owned:** `scripts/generate-index.mjs`, `.github/workflows/pages.yml`. Safe
  to overwrite downstream.
- **Repository-owned:** `config.json`, `builds/`, and in practice `README.md`. Never
  overwrite these when propagating a change.

`index.html` is generated, so it is regenerated rather than copied.

## Why `index.html` is committed

It is build output, and committing build output is usually a smell. It is committed here
deliberately: if someone sets Pages to "Deploy from a branch" instead of "GitHub
Actions", a committed index still serves a working site — stale, but working — where an
ignored one would serve a 404 with no obvious cause. It also lets someone clone the
repository and open the page without installing anything. The diff noise is small,
because the file only changes when `builds/` or `config.json` changes, which is already
a commit.
