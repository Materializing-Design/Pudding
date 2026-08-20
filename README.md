# Pudding
It's where the design proof is

A GitHub Pages site that hosts every dated static build of a project and serves them one at a time, with a generated index linking to each.

## Adding a build

1. Drop the build's folder into `builds/`, named `YYYY-MM-DD`. It must contain an `index.html`, and its asset paths must be relative (no leading `/`), since the site is served from a subpath.
2. Commit and push.

### More than one build in a day

Add a time to the folder name — `YYYY-MM-DD-HHMM`, 24-hour, e.g. `2017-07-02-1430`.
Use `YYYY-MM-DD-HHMMSS` if two land in the same minute; `T` also works as the
separator (`2017-07-02T1430`).

Times are optional and mix freely with plain dates: a folder with no time counts as
that day's earliest build. Listed times are shown as written, with no timezone
conversion.

The index regenerates on push — the list is read from the folder names in `builds/`,
never hand-written. To preview locally first:

```bash
node scripts/generate-index.mjs && python3 -m http.server 4321
```

## Editing the copy

Project name, author, and description live in `config.json`:

```json
{
  "name": "Your game's title",
  "author": "Your name",
  "description": "A sentence or two about the project."
}
```

`name` and `author` are required; `description` is optional, and its paragraph is
omitted when absent or empty.

## How it deploys

`.github/workflows/pages.yml` runs `scripts/generate-index.mjs` and publishes the
repository to GitHub Pages on every push to `main`. This requires
**Settings → Pages → Source: "GitHub Actions"**.

`index.html` is generated output — edit `scripts/generate-index.mjs`, not the HTML.
