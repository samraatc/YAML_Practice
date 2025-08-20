## What are artifacts?

Artifacts are files you save from a workflow run so that:

other jobs in the same run can use them (e.g., build → test → deploy),

humans can download them from the Actions UI,

other workflows (even in other repos) can fetch them later.

Think: build zips, test reports, coverage, binaries, screenshots, logs.

Tip: Artifacts ≠ cache. Use cache for reusing dependencies to speed builds; use artifacts to pass/keep results


## The two actions you’ll use

#### Upload  :
```bash 
 uses: actions/upload-artifact@v4
```
Fast, immutable, supports exclusions, compression level, retention, etc. 


#### Download:
```bash 
uses: actions/download-artifact@v5

```
Can grab by name or ID, filter by pattern, merge multiple into one directory, and even pull from other runs/repos with a token.


### Basic: upload and download within one workflow


```sh
name: Build & Test (artifacts demo)

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: |
          mkdir -p dist
          echo "hello $(date)" > dist/hello.txt
      - name: Upload build output
        uses: actions/upload-artifact@v4
        with:
          name: web-dist
          path: dist/
          retention-days: 7          # 1..90 (or repo default)
          if-no-files-found: error   # warn | error | ignore
  
  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download build
        uses: actions/download-artifact@v5
        with:
          name: web-dist
          path: ./dist
      - run: ls -R dist && echo "…run tests using ./dist…"

```


- retention-days lets you keep artifacts for a set time (default comes from repo settings).
- if-no-files-found controls failure/warnings if your glob matched nothing


### Multiple paths, wildcards, and excludes

```sh
- name: Upload reports
  uses: actions/upload-artifact@v4
  with:
    name: reports
    path: |
      reports/**/*.xml
      coverage/**/*
      !**/*.tmp
```
- Wildcards and ! excludes are supported; dir structure is preserved sensibly.


### Matrix builds: one artifact per job, then merge when downloading
With v4, each artifact name must be unique (artifacts are immutable). Use matrix vars in the name, then download with a pattern and merge.
```sh
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - run: echo "${{ matrix.os }}" > out.txt
      - uses: actions/upload-artifact@v4
        with:
          name: bin-${{ matrix.os }}
          path: out.txt

  collect:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download all bins into one folder
        uses: actions/download-artifact@v5
        with:
          pattern: bin-*
          merge-multiple: true
          path: artifacts
```
- This pulls bin-* artifacts into a single artifacts/ directory.

- Optional: there’s also a helper sub-action to merge already-created artifacts into a new single artifact (actions/upload-artifact/merge@v4), best for cases where humans will download one file from the UI.

### Cross-workflow / cross-repo downloads
'download-artifact@v5' can fetch from another run or another repo if you pass a token with actions:read. 
GitHub

Example A: a “Release” workflow that runs after “Build”

The “Build” workflow uploads an artifact. This “Release” workflow triggers on workflow_run, reads that run’s ID, and downloads the artifact.

```sh
name: Release (consume artifact from Build)
on:
  workflow_run:
    workflows: ["Build"]
    types: [completed]

jobs:
  gather:
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
    steps:
      - name: Download artifact from the triggering run
        uses: actions/download-artifact@v5
        with:
          name: web-dist
          run-id: ${{ github.event.workflow_run.id }}
          # same repo by default; add repository: owner/repo for cross-repo
          # and github-token: ${{ secrets.GH_PAT }} for cross-repo
```
- Inputs repository, run-id, and github-token enable cross-run/cross-repo pulls


### Advanced knobs you should know

- Compression: compression-level: 0..9 (default 6). Use 0 for already-compressed or random binaries to speed uploads; 9 to save storage on text.
- Hidden files: excluded by default in v4; enable with include-hidden-files: true (be careful with secrets).
- Overwrite: set overwrite: true to replace an artifact of the same name within another job/run. Otherwise upload will fail if the name exists.
- Outputs: upload-artifact@v4 returns artifact-id, artifact-url, and artifact-digest (handy for linking or API use).
- imits: up to 500 artifacts per job; artifacts are immutable ZIPs; file permissions are not preserved (use tar if you must keep modes).


```sh 
- run: tar -cvf app.tar ./dist
- uses: actions/upload-artifact@v4
  with:
    name: dist-tar
    path: app.tar
```


### note :- 

- name → Logical identifier of the artifact.

- Used later when downloading.

- Can be anything (build-output, logs, reports).

- path → File(s)/folder(s) you want to upload.

- Supports single files (build/message.txt) or entire folders (build/).

- Can also use globs (build/**/*.txt).