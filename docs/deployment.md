# Deployment

The site is published by GitHub Pages from a GitHub Actions workflow. Every push to
`main` regenerates the index and redeploys — there is no manual build or upload step.

## First-time setup

1. **Generate a repository from the template.** On GitHub: *Use this template → Create a
   new repository*.
2. **Set the Pages source.** *Settings → Pages → Source* → **GitHub Actions**. This is
   required; leaving it on "Deploy from a branch" changes the behaviour (see below).
3. **Edit `config.json`** with the project's name, author, and description.
4. **Add a build** under `builds/`, then commit and push.

The site appears at `https://<org>.github.io/<repo>/`. The first deploy can take a
couple of minutes; later ones are usually under one.

## What the workflow does

`.github/workflows/pages.yml`:

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
```

On each run it checks out the repository, sets up Node 22, runs
`node scripts/generate-index.mjs`, then uploads the whole repository (`path: .`) as the
Pages artifact and deploys it.

Two details worth knowing:

- **`permissions: pages: write` and `id-token: write`** are what let the job deploy to
  Pages at all. Removing them breaks the deploy with an opaque permissions error.
- **`concurrency: { group: pages, cancel-in-progress: false }`** lets a running deploy
  finish instead of being cancelled by a newer push. Two quick pushes therefore produce
  two deploys in sequence rather than one aborted mid-publish.

`workflow_dispatch` means you can also deploy by hand from the Actions tab without
pushing — useful for the very first deploy after enabling Pages.

Because the workflow regenerates `index.html` before uploading, the deployed index
always reflects what is actually in `builds/`, even if the committed `index.html` is out
of date.

## `.nojekyll`

The repository root contains an empty `.nojekyll` file. Without it, Pages runs the
uploaded files through Jekyll, which ignores paths beginning with `_` and can rewrite or
drop files inside build snapshots. Deleting this file will break builds in ways that are
tedious to diagnose. Leave it there.

## If Pages is set to "Deploy from a branch"

The site still works, but it serves the **committed** `index.html` rather than a freshly
generated one. New build folders will not appear until someone runs the generator
locally and commits the result:

```bash
node scripts/generate-index.mjs
git commit -am "Regenerate index"
```

This fallback is why `index.html` is committed rather than ignored. The intended setup
is still GitHub Actions.

## Working locally

```bash
node scripts/generate-index.mjs
python3 -m http.server 4321
```

Then open `http://localhost:4321`. Opening `index.html` straight from disk mostly works,
but a local server matches how Pages actually serves the site — particularly for paths
inside builds.

To check a specific build in isolation:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:4321/builds/<name>/index.html
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Site 404s entirely | Pages source not set, or no successful deploy yet | *Settings → Pages → Source → GitHub Actions*, then re-run the workflow |
| A build is missing from the list | Folder name didn't parse, or it has no `index.html` | Check the Actions log for a `Skipping` line; rename the folder or add the entry point |
| A build opens blank or unstyled | Absolute paths inside the build (`/css/style.css`) | Make every path relative — the site is served from a subpath |
| New builds never appear | Pages is on branch deploy | Switch to GitHub Actions, or regenerate and commit locally |
| Workflow doesn't run at all | Pushed to a branch other than `main` | Merge to `main`, or add the branch to the trigger |
| Deploy fails on permissions | `permissions:` block edited | Restore `pages: write` and `id-token: write` |
| Whole run fails, nothing deploys | `config.json` missing `name`/`author`, or `builds/` empty | The generator exits 1 on both; the log names which |

Fonts load from Google Fonts and degrade to system fallbacks, so a blocked font request
changes the typography but never breaks the page.

## Updating repositories generated from this template

Generated repositories share no history with the template, so changes are copied, not
merged. Only `scripts/generate-index.mjs` and `.github/workflows/pages.yml` are
template-owned; `config.json`, `builds/`, and `README.md` belong to each repository and
must not be overwritten.

For each repository: copy those two files in, run the generator, commit, push. Pushing
triggers that repository's own workflow, which redeploys it.

Two cautions from the last rollout:

- **Check for `Skipping` lines after regenerating.** A generator change can stop
  recognising a naming convention some repository already uses, which removes builds
  from that site with nothing but a log line to show for it.
- **Verify afterwards, not just before.** Comparing each site's listed entries against
  its actual folder count in `builds/` catches silent drops that a green workflow badge
  will not.

If this becomes frequent, the alternative is to stop vendoring the generator: publish it
as a reusable workflow from the template and have generated repositories call it as
`uses: Materializing-Design/Pudding/.github/workflows/build.yml@v1`. Changes then
propagate on each repository's next push, at the cost of making them depend on the
template repository — pin a tag rather than `@main` so the timing stays under your
control.
