# Changelog from Conventional Commits - Github Action

This GitHub Action automatically generates a changelog based on all the [Conventional Commits](https://www.conventionalcommits.org) between the latest tag and the previous tag, or beween 2 specific tags.

- [Features](#features)
- [Example Workflows](#example-workflows)
  - [Using the latest tag](#using-the-latest-tag)
  - [Using a specific tag range](#using-a-specific-tag-range)
- [Inputs](#inputs)
- [Outputs](#outputs)
- [Important Info](#warning-important-warning)

## Features

- Generates the CHANGELOG changes in Markdown format
- Turns PR ids into links and add the PR author.
- Turns Jira ids into links.
- Prepends a shortened commit SHA ID to the commit for quick access.
- `BREAKING CHANGE` notes are added to the top of the changelog version along with the related commit.
- Exports changelog to a variable that can used in a subsequent step to create a release changelog.
- Automatically injects the changes into the CHANGELOG.md file or creates it if it doesn't exist yet. *(optional)*
- Will not mess up with any header or instructions you already have at the top of your CHANGELOG.md.
- Will not add duplicate version changes if it already exists in the CHANGELOG.md file.
- Optionally exclude types from the CHANGELOG. (default: `build,docs,other,style`)

## Example Workflows

### Using the latest tag

``` yaml
name: Deploy

on:
  push:
    tags:
      - v[0-9]+.[0-9]+.[0-9]+

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Update CHANGELOG
        id: changelog
        uses: 58ielaay0/changelog-action@v1
        with:
          jiraBrowseUrl: # Url to jira, e.g. "https://jira.example.com/browse"
          jiraPrefixes: 'EPIC, TEST'
          token: ${{ github.token }}
          tag: ${{ github.ref_name }}

      - name: Create Release
        uses: ncipollo/release-action@v1.12.0
        with:
          allowUpdates: true
          draft: false
          makeLatest: true
          name: ${{ github.ref_name }}
          body: ${{ steps.changelog.outputs.changes }}
          token: ${{ github.token }}

      - name: Commit CHANGELOG.md
        uses: stefanzweifel/git-auto-commit-action@v4
        with:
          branch: main
          commit_message: 'docs: update CHANGELOG.md for ${{ github.ref_name }} [skip ci]'
          file_pattern: CHANGELOG.md
```

### Using a specific tag range

``` yaml
name: Deploy

on:
  push:
    tags:
      - v[0-9]+.[0-9]+.[0-9]+

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Get previous tag
        id: previousTag
        run: |
          name=$(git --no-pager tag --sort=creatordate --merged ${{ github.ref_name }} | tail -2 | head -1)
          echo "previousTag: $name"
          echo "previousTag=$name" >> $GITHUB_ENV

      - name: Update CHANGELOG
        id: changelog
        uses: requarks/changelog-action@v1
        with:
          token: ${{ github.token }}
          fromTag: ${{ github.ref_name }}
          toTag: ${{ env.previousTag }}
          writeToFile: false

      - name: Create Release
        uses: ncipollo/release-action@v1.12.0
        with:
          allowUpdates: true
          draft: true
          makeLatest: true
          name: ${{ github.ref_name }}
          body: ${{ steps.changelog.outputs.changes }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Field | Description | Required | Default |
|-------|-------------|:--------:|---------|
| `token` | Your GitHub token (e.g. `${{ github.token }}`) | :white_check_mark: | |
| `tag` | The latest tag which triggered the job. (e.g. `${{ github.ref_name }}`) | :white_check_mark: <br> *(unless using `fromTag` and `toTag`)* | |
| `fromTag` | The tag from which the changelog is to be determined (latest) | :white_check_mark: <br> *(unless using `tag`)* | |
| `toTag` | The tag up to which the changelog is to be determined (oldest) | :white_check_mark: <br> *(unless using `tag`)* | |
| `excludeTypes` | A comma-separated list of commit types you want to exclude from the changelog (e.g. `doc,chore,perf`) | :x: | `build,docs,other,style` |
| `writeToFile` | Should CHANGELOG.md be updated with latest changelog | :x: | `true` |
| `includeRefIssues` | Should the changelog include the issues referenced for each PR. | :x: | `true` |
| `useGitmojis` | Should type headers be prepended with their related gitmoji | :x: | `true` |
| `includeInvalidCommits` | Whether to include commits that don't respect the Conventional Commits format | :x: | `false` |
| `reverseOrder` | List commits in reverse order (from newer to older) instead of the default (older to newer). | :x: | `false` |
| `jiraBrowseUrl` | Jira browse url you want the issues to be linked to  (e.g https://jira.example.com/browse ) | :x: | |
| `jiraPrefixes` | 'SL, SBKRESP, LPS : If you want to specify which issues that should be shown | :white_check_mark: <br> *(unless using `jiraBrowseUrl`)* | |

## Outputs

| Field | Description |
|-------|-------------|
| `changes` | Generated CHANGELOG changes for the latest tag, without the version / date header *(for use in GitHub Releases)*. |

## :warning: Important :warning:

You must already have 2 tags in your repository (1 previous tag + the current latest tag triggering the job). The job will exit with an error if it can't find the previous tag!
