# Reusable workflows

[日本語版 (Japanese 🇯🇵)](README.ja.md)

The workflows in this directory are reusable: other repositories call them with
`uses:` rather than copying them.

The repository is named `.github`, so the reference has the directory in it twice —
`tamada/.github` is the repository, `.github/workflows/...` is the path inside it:

```
uses: tamada/.github/.github/workflows/<file>.yaml@<ref>
```

The extension is `.yaml`, not `.yml`, and the `@<ref>` is required. Getting either
wrong fails the run before any job starts, with `invalid value workflow reference`.

| Workflow | Kind | Purpose |
|----------|------|---------|
| [`build-hugo-and-publish.yaml`](#build-hugo-and-publishyaml) | reusable | Build a Hugo site and deploy it to GitHub Pages. |
| [`release-start.yaml`](#release-startyaml) | reusable | Tag a release and open it as a draft. |
| [`release-finish.yaml`](#release-finishyaml) | reusable | Publish that draft created by `release-start.yaml`, so `release: published` actually fires. |
| [`release.yaml`](#this-repositorys-own-workflows) | this repository | Releases this repository, by calling the two above. |
| [`lint.yaml`](#this-repositorys-own-workflows) | this repository | Runs actionlint on every pull request. |

`release-start` and `release-finish` are two halves of one flow and are meant to
be used together, with the caller's own build jobs in between.

The two kinds sit in one directory because they have to: subdirectories of
`.github/workflows` are not supported, so a file placed in one is not a workflow
at all. Telling them apart needs no convention, though — the reusable ones are
`on: workflow_call` and nothing else, which also means they never start on their
own, and the only runs in this repository's Actions tab are its own two.

## Choosing a ref

This repository is tagged twice per release: an immutable `v1.2.3` recording
exactly what was released, and `v1`, which is moved to point at it. So `v1` is
always the newest v1.x.y, and `v1.2.3` is always that one release, forever.

`@v1` is a major-version tag that moves as fixes land within v1. It is the right
choice for almost everyone: you get fixes without editing every caller, and a
breaking change arrives as `v2`, which cannot reach you until you ask for it.

`@main` tracks whatever was merged most recently, including changes that have not
been released. Useful for trying something out, not for a workflow you rely on.

`@<full-sha>` pins exactly. Nothing moves under you, and nothing is fixed for you
either — you upgrade by editing the caller.

## What the caller has to provide

**Permissions.** A called workflow cannot give itself more than the caller allows;
permissions are maintained or reduced through the chain, never elevated. This means
the `permissions:` block belongs on the calling job, and leaving it out fails the
run with a message naming what was requested and what was allowed. Each workflow
below lists what it needs.

**Secrets.** Secrets are not passed implicitly. Either map them one by one under
`secrets:`, or write `secrets: inherit` to pass everything the caller has. Only
`release-finish` takes any.

**Environment variables do not cross over.** `env` set at the workflow level in the
caller is not propagated into the called workflow. Anything the workflow needs has
to arrive as an input.

**Access.** These workflows live in a public repository, so any repository may call
them, subject to the calling organization's policy on using public reusable
workflows.

Two limits are worth knowing, though neither is close for normal use: a chain may
be up to ten levels deep, and one workflow file may call up to 50 distinct
reusable workflows.

## `build-hugo-and-publish.yaml`

Builds a Hugo site with the extended edition and deploys it to GitHub Pages.

Before the first run, set *Settings → Pages → Build and deployment → Source* to
**GitHub Actions**. Without it the deploy step fails; the workflow does not enable
Pages on your behalf.

```yaml
name: publish

on:
  push:
    branches: [main]

jobs:
  publish:
    uses: tamada/.github/.github/workflows/build-hugo-and-publish.yaml@v1
    with:
      workdir: docs
      # branch: main
      # hugo-version: latest
    permissions:
      contents: read
      pages: write
      id-token: write
```

### Inputs

| Input | Default | Meaning |
|-------|---------|---------|
| `workdir` | `.` | Directory holding the Hugo site. The artefact is taken from `<workdir>/public`. |
| `branch` | *(empty)* | Git ref to build. Empty means the ref that triggered the caller's run, which is what you want for the usual push trigger. |
| `hugo-version` | `latest` | Hugo version. Always the extended edition. |

### Outputs

None. The deployed URL is visible on the run's `github-pages` deployment rather
than being handed back to the caller.

### Notes

**`baseURL` is your responsibility.** The build runs `hugo --minify` and nothing
else, so the `baseURL` in your Hugo configuration is what ends up in the generated
links. If it does not match where Pages actually serves the site, the pages load
but the stylesheets and internal links point somewhere wrong — a failure that
looks like a broken theme rather than a misconfiguration.

**Submodules are checked out recursively**, because Hugo themes are so often
vendored that way. A missing submodule otherwise surfaces as a missing layout,
which sends you looking in the wrong place entirely.

**Deployments are serialised** on a `pages` concurrency group and are never
cancelled mid-flight: GitHub rejects a second concurrent deployment rather than
queueing it, and cancelling one already running would leave the site on whatever
had been deployed so far.

## `release-start.yaml`

Tags a release and opens it as a *draft*.

It is deliberately language-agnostic — it tags and drafts, nothing more. Building
binaries, publishing to a registry and anything else project-specific belongs in
the caller's own jobs, which run between this workflow and `release-finish`.

The draft is also deliberate. A release created as published by `GITHUB_TOKEN`
fires no `release: published` event, so nothing downstream would notice it;
`release-finish` takes the draft off with a token that does.

### Cutting a release

1. Branch from the tag branch as `releases/v0.0.5` — the version, with a `v`,
   after the prefix.
2. Make whatever the release needs on it: version bumps, changelog, and so on.
3. Open a pull request onto `main` and merge it.

Merging is the trigger. Two branches are involved and they are not the same one:

```
releases/v0.0.5  ──── pull request ────▶  main
      │                                    │
carries the version                  gets the tag
```

The version is read from the head branch of the merged pull request; the tag is
placed on the branch it merged into.

```yaml
name: release

on:
  pull_request:
    types: [closed]

jobs:
  start:
    if: startsWith(github.head_ref, 'release') && github.event.pull_request.merged == true
    uses: tamada/.github/.github/workflows/release-start.yaml@v1
    permissions:
      contents: write
```

The `if` guard matters on both counts. `types: [closed]` fires whether the pull
request was merged or just closed, so without `merged == true` an abandoned
release branch would cut a tag. The `startsWith` half keeps unrelated pull
requests from reaching a workflow that would only fail on them.

### Inputs

| Input | Default | Meaning |
|-------|---------|---------|
| `version` | *(empty)* | Version to release, with or without the leading `v` — either is accepted. Empty means read it from the head branch. Pass it explicitly when the run was not triggered by a release branch. |
| `version-branch-prefixes` | `releases/v release/v` | Space-separated prefixes; what follows the first matching one is the version. |
| `tag-branch` | `main` | Branch checked out and tagged. |
| `appname` | *(repository name)* | Application name, for later jobs that name artefacts. Correct whenever the binary is named after its repository. |

### Outputs

| Output | Example | Meaning |
|--------|---------|---------|
| `version` | `0.0.5` | The version, without the leading `v`. |
| `tag` | `v0.0.5` | The git tag that was created. |
| `appname` | `myapp` | The resolved application name. |

### Notes

**`version` is `0.0.5`, `tag` is `v0.0.5`, and neither name ever means the other.**
Take `version` when the value goes into a binary, a package manifest or a
filename; take `tag` when it goes to git or to `release-finish`. Nothing needs
to add or strip the `v` on the way. The `version` *input* is more relaxed than
the outputs are — it takes `0.0.5` or `v0.0.5` — because there is no reason to
reject a version someone wrote out in the obvious way.

**A branch matching no prefix fails the run** rather than guessing. Trimming to
the last `v` instead would read `releases/v1.0-preview` as `iew`, and would hand
back the whole branch name when nothing matched — both of which tag the wrong
thing without complaining. Naming the prefixes means a branch that follows no
convention stops, loudly.

**Matching a prefix is not on its own enough**, because a plausible-looking
branch can still carry something that is not a version. `releases/vNEXT` clears
the prefix bar and would otherwise be tagged `vNEXT`; `releases/v0.0.5/hotfix`
would become a tag with a slash in it. So the version is also checked: it must
begin with a digit and contain only digits, letters, dots, `+` and `-`.
`1.0-preview` and `1.2.3+build5` pass; those two do not.

**`github.head_ref` is only set on pull request events.** Triggered any other way,
the branch cannot be read and the `tag` input becomes mandatory.

**Release notes are generated** by GitHub from the commits since the previous tag.

## `release-finish.yaml`

Takes the draft off the release opened by `release-start`.

This exists as its own workflow for one reason: which token publishes the release
decides whether anything downstream ever hears about it.

> With the exception of `workflow_dispatch` and `repository_dispatch`, other
> `GITHUB_TOKEN`-triggered events do not create workflow runs at all.

So a release published with `GITHUB_TOKEN` fires no `release: published`, and a
workflow waiting on that event stays silent with nothing to show for it — no
failed run, no log, just nothing. Publishing with a GitHub App token instead makes
the event behave like any other.

Without App credentials the release is still published, using `GITHUB_TOKEN`, and
the job logs a warning saying plainly that downstream workflows will not run. If
you have nothing listening for `release: published`, that is a perfectly good way
to use it.

```yaml
jobs:
  finish:
    needs: [start, build]
    uses: tamada/.github/.github/workflows/release-finish.yaml@v1
    with:
      tag: ${{ needs.start.outputs.tag }}
    permissions:
      contents: write
    secrets: inherit
```

### Inputs

| Input | Required | Meaning |
|-------|----------|---------|
| `tag` | yes | Git tag of the release to publish, with the leading `v` (`v0.0.5`) — exactly what `release-start` puts in its `tag` output, so it can be handed straight over. |

### Secrets

| Secret | Required | Meaning |
|--------|----------|---------|
| `APP_CLIENT_ID` | no | Client ID of a GitHub App allowed to edit releases. |
| `APP_PRIVATE_KEY` | no | Private key of that App. |

The names use underscores because repository secrets may only contain letters,
digits and underscores. Hyphenated names would read better here and would never
once arrive, because `secrets: inherit` matches by name and no repository secret
could be named to match.

### Setting up the GitHub App

Only needed if something of yours listens for `release: published`.

1. Create a GitHub App under your account: *Settings → Developer settings →
   GitHub Apps → New GitHub App*. It needs no webhook and no callback URL.
2. Give it one repository permission: **Contents: Read and write**. Releases live
   under Contents.
3. Generate a private key and note the Client ID.
4. Install the App on the account, granting it the repositories that will publish
   releases. `owner` is set to the calling repository's owner, so the token covers
   the repositories the App is installed on there.
5. Store the two values as repository or organization secrets named
   `APP_CLIENT_ID` and `APP_PRIVATE_KEY`.

The token is minted per run and revoked when the job ends.

## Putting the release workflows together

```yaml
name: release

on:
  pull_request:
    types: [closed]

jobs:
  start:
    if: startsWith(github.head_ref, 'release') && github.event.pull_request.merged == true
    uses: tamada/.github/.github/workflows/release-start.yaml@v1
    permissions:
      contents: write

  build:
    needs: start
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      # ... build and upload artefacts against
      # needs.start.outputs.version (0.0.5), needs.start.outputs.tag (v0.0.5)
      # and needs.start.outputs.appname

  finish:
    needs: [start, build]
    uses: tamada/.github/.github/workflows/release-finish.yaml@v1
    with:
      tag: ${{ needs.start.outputs.tag }}
    permissions:
      contents: write
    secrets: inherit
```

`build` runs while the release is still a draft, which is the point of the split:
artefacts are attached before anyone — or anything — is told the release exists.

## This repository's own workflows

`release.yaml` and `lint.yaml` are not for calling. They are how this repository
is released and tested, and they are worth reading mainly as a worked example of
everything above.

### `release.yaml`

Releases this repository, by the same flow the README describes: branch
`releases/vX.Y.Z`, pull request onto `main`, merge.

It calls the reusable workflows as `uses: ./.github/workflows/release-start.yaml`.
A local reference carries no `@ref` and takes the called workflow **from the same
commit as the caller**, so a release runs the workflows as they are on `main`
rather than as they were tagged. Anything that breaks `release-start` breaks this
repository's own release first — before the tag exists, and before it reaches
anyone pinning `@v1`.

Between `start` and `finish`, where a normal project would build artefacts, sits
the one job a repository publishing reusable workflows needs and almost nobody
else does: moving `v1` to the release just cut. It is skipped for pre-releases,
since `@v1` should not hand a release candidate to everyone who never asked for
one. Build metadata (`1.2.3+build`) is still a release; only a hyphen marks a
pre-release.

### `lint.yaml`

Runs actionlint on every pull request. Every file here is a workflow, so this is
the whole test suite — and a mistake in a reusable workflow is a mistake in
somebody else's repository.

Two patterns are passed to `-ignore`. They are actionlint's bundled copy of
`actions/create-github-app-token`'s inputs being out of date rather than
findings: the action has taken `client-id` since v2 and now marks `app-id`
deprecated in favour of it. Both patterns name that action, so a real problem
with any other action still fails the job.

It runs a pinned `rhysd/actionlint` image rather than the upstream `curl | bash`
installer. What this repository publishes ends up running in other people's
repositories, which is a poor place to start executing a script fetched from a
moving branch.

## Troubleshooting

| Symptom | Cause |
|---------|-------|
| `invalid value workflow reference` before any job starts | `.yml` instead of `.yaml`, a missing `@<ref>`, or the repository path written once instead of twice. |
| `The workflow is requesting 'pages: write', but is only allowed 'pages: none'` | No `permissions:` block on the calling job. A called workflow cannot elevate its own permissions. |
| `head branch '…' starts with none of: …` | The release branch does not begin with `releases/v` or `release/v`. Rename it, extend `version-branch-prefixes`, or pass `version` explicitly. |
| `version '…' does not start with a digit` | The branch is named like a release branch but carries something else — `releases/vNEXT` and the like. |
| `version '…' contains characters that do not belong in a version` | Usually a slash: `releases/v0.0.5/hotfix` would otherwise become a tag with a `/` in it. |
| The release is published, but a workflow waiting on `release: published` never runs | No App credentials, so `GITHUB_TOKEN` published it. Check the run for the warning; see [Setting up the GitHub App](#setting-up-the-github-app). |
| Tag creation fails on an existing tag | That version was already released. The tag is the record; bump the version rather than moving it. |
| Pages deploy fails on a repository that has never published | Pages source is not set to GitHub Actions. |
| The site deploys but styles and links are broken | `baseURL` in the Hugo configuration does not match where Pages serves the site. |
| Hugo fails on a missing layout | The theme is a submodule that was not fetched — check that it is committed as a submodule and not an empty directory. |
