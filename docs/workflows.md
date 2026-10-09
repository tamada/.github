# workflows

This repository contains a collection of GitHub Actions workflows by calling other workflows.

## `release-start.yaml`

The workflow prepares the release.


## `build-hugo-and-publish.yaml`

This workflow builds the site by Hugo and publishes it to GitHub Pages.
To call this workflow, create a new workflow dispatch event or use the GitHub UI to trigger it.
This workflow requires the following permissions:

- `contents`: `read`
- `pages`: `write`
- `id-token`: `write`

```yaml
jobs:
  call-hugo-build:
    uses: tamada/workflows/.github/workflows/build-hugo-and-publish.yaml@main
    with:
      workdir: docs
      # branch: main
      # hugo-version: latest
    # Hugoのビルド＆Pagesデプロイに必要な標準的な権限
    permissions:
      contents: read
      pages: write
      id-token: write  
```

### Variables

- `hugo-version`: The version of Hugo to use.
  - Default: `latest`
- `branch`: The branch to publish to.
  - Default: `main`
- `workdir`: The working directory to use.
  - Default: `.`
