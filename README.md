# DPM Git Dependencies Demo App 

A tiny Daml project aimed to test git-like links resolution in DPM.

The committed `daml.yaml` declares two git dependencies:

```yaml
dependencies:
  - git:github.com/Moonsong-Labs/test-daml-hello#master?path=dist/test-daml-hello-sdk-3.5.2-lf-2.2.dar

data-dependencies:
  - git:github.com/canton-network/splice#release-line-0.6.8?path=daml/dars/splice-amulet-0.1.19.dar
```

Each line points at a `.dar` file sitting in two different Git repositories. One of them exposes a `Hello.greeting` value and the other is a known library used in production, that exposes, among other things, the template `Splice.Amulet.Amulet`. Both are imported in the app, so if either link stops resolving, the build will fail. 

## What you need

- A locally built `dpm-dev`. Git links are not in a released DPM yet, so you
  need to use the [Moonsong Labs fork](https://github.com/Moonsong-Labs/dpm/) at the `proposal/git-dependencies-support` branch:

  ```bash
  git clone --branch proposal/git-dependencies-support git@github.com:Moonsong-Labs/dpm.git
  cd dpm
  go build -o bin/dpm-dev ./cmd/dpm/
  export PATH="$PWD/bin:$PATH"
  ```

  Building needs Go 1.25.0. The `export` puts `dpm-dev` on your `PATH` for the
  current shell; add the same line to your `~/.zshrc` or `~/.bashrc` with
  the real path in place of `$PWD` to make it durable. You can also call
  the binary by the full path, as `/path/to/dpm/bin/dpm-dev`.
- Network access to `github.com` the first time, so DPM can fetch the DARs.
  After that they are cached under `~/.dpm/cache`.

## Quick start

```bash
dpm-dev install package
dpm-dev build
dpm-dev test
```

`dpm-dev install package` resolves both git dependencies and caches the DARs, you should see something like:

```
SDK version 3.5.2 is already installed
installing git dar "git:github.com/Moonsong-Labs/test-daml-hello#master?path=dist/test-daml-hello-sdk-3.5.2-lf-2.2.dar"...
Resolving git dependency: fetching https://github.com/Moonsong-Labs/test-daml-hello @ "master" path "dist/test-daml-hello-sdk-3.5.2-lf-2.2.dar"
installing git dar "git:github.com/canton-network/splice#release-line-0.6.8?path=daml/dars/splice-amulet-0.1.19.dar"...
Resolving git dependency: fetching https://github.com/canton-network/splice @ "release-line-0.6.8" path "daml/dars/splice-amulet-0.1.19.dar"
No opt-in components to install
```

The two `Resolving` lines only appear when the DAR is not cached yet. The
committed `daml.yaml` uses movable refs (`master` and `release-line-0.6.8`) on
purpose, so `dpm-dev install package` will pin them to commit SHAs and dirty
your working tree. That rewrite is expected; do not commit it — restore the
unpinned file before you push.

`dpm-dev build` compiles, including the module that imports from both git-hosted
DARs:

```
Created .daml/dist/dpm-git-links-demo-0.0.1.dar
```

Templates and `daml-script` live in the same package here, which would normally
warn that uploading the DAR also uploads `daml-script`. That is expected for a
single-package demo and unrelated to git links, so `daml.yaml` silences it with
`-Wno-template-interface-depends-on-daml-script`.

`dpm-dev test` runs the scripts in `daml/Tests.daml`:

```
Test Summary

daml/Main.daml:setup: ok, 1 active contracts, 1 transactions.
daml/Tests.daml:testReword: ok, 1 active contracts, 2 transactions.
daml/Tests.daml:testSetup: ok, 1 active contracts, 1 transactions.
```

To prove the links are doing real work, delete either one from `daml.yaml` and
run `dpm-dev build` again. Without the splice line:

```
Could not find module ‘Splice.Amulet’
ERROR: Creation of DAR file failed.
```

and without the Hello line:

```
Could not find module ‘Hello’
ERROR: Creation of DAR file failed.
```

## How to declare a git dependency

The canonical form is one string:

```
git:<host>/<repo>#<ref>?path=<path/inside/the/repo.dar>
```

- `<ref>` is a branch, a tag, or a commit SHA.
- `?path=` is required, is relative to the repo root, and must end in `.dar`.
- A trailing `.git` on the repo is fine; DPM drops it.
- HTTPS only. SSH is not supported: `git:ssh://…` and `git@github.com:org/repo`
  are both rejected. DPM fetches anonymously over HTTPS and never touches your
  SSH agent, keys or tokens, so the repo you depend on has to be public.
- Any public git host reachable over HTTPS, including GitLab, Bitbucket, and
  self-hosted servers — for example
  `git:gitlab.com/example-org/example-repo#main?path=packages/foo.dar`.
  `?release=` is `github.com` only, since it uses the GitHub releases API.
  On other hosts, depend on a committed `.dar` with
  `#<ref>?path=` instead.

Put it either under `dependencies` or `data-dependencies`. If your dependency goes the the former, the package will need to match the SDK version of the demo app, which is set to be 3.5.2.

For a GitHub release single asset, swap `#<ref>?path=` for a release query:

```
git:<host>/<repo>?release=<tag>&asset=<file>.dar
```

For a GitHub release with all its assets, omit `&asset=`. DPM will expand the entry into one line per `.dar` attached to
that release.

### Git Raw links support 

We support this feature via `dpm add dar`, so you don't have to assemble the git line by hand. You can copy any Git URL pointing to a DAR, and hand it over:

```bash
dpm-dev add dar \
  'https://github.com/canton-network/splice/raw/refs/tags/0.6.10/daml/dars/splice-amulet-0.1.19.dar' \
  --data-dependencies
```

DPM will normalize the browser URL, and resolve the dependency as explained earlier.

## Samples

In the samples folder you'll find some handy ways of defining git dependencies, with a header explaining what you should expect.
Quoted refs are the input (`master`, `0.6.10`, `main`); resolved commit SHAs are
not frozen in the samples.

| Sample | Form |
| --- | --- |
| [`browser-url-tag.yaml`](samples/browser-url-tag.yaml) | GitHub `raw` URL at a release tag, via `dpm-dev add dar` |
| [`browser-url-branch.yaml`](samples/browser-url-branch.yaml) | GitHub `raw` URL on a branch, via `dpm-dev add dar` |
| [`browser-url-commit.yaml`](samples/browser-url-commit.yaml) | GitHub `blob` URL at a commit SHA, via `dpm-dev add dar` |
| [`artifact-location-alias.yaml`](samples/artifact-location-alias.yaml) | Name the repo once under `artifact-locations`, use `@alias` |
| [`release-single-asset.yaml`](samples/release-single-asset.yaml) | GitHub release, one named asset |
| [`release-all-assets.yaml`](samples/release-all-assets.yaml) | GitHub release, no asset named: all 35 of them |

The samples use three real DARs: `splice-amulet-0.1.19.dar` from
`canton-network/splice`, the small `Hello` fixture this project also builds
against, and the `Moonsong-Labs/daml-finance` `test-release-0.0.6` release for
the release forms. `canton-network/splice` publishes its DARs as files in the
tree rather than as GitHub release assets, which is why the release samples use
a different repo.

## Gotchas

### **A browser URL works with `dpm-dev add dar` but not in `daml.yaml`.** 

Pasting one raw Git link into a dependency list fails:

```yaml
data-dependencies:
  - "https://github.com/canton-network/splice/raw/refs/tags/0.6.10/daml/dars/splice-amulet-0.1.19.dar"
```

```
Error: failed to parse provided data-dependencies: couldn't parse dependency "https://github.com/canton-network/splice/raw/refs/tags/0.6.10/daml/dars/splice-amulet-0.1.19.dar": http dependencies not yet supported
```

URL normalization only happens inside `dpm-dev add dar`. When DPM reads
`daml.yaml` it expects a `git:` line and nothing else. So do not paste browser
URLs into the file: pass them to `dpm-dev add dar` and let it write the
canonical line.

### **DPM rewrites your `daml.yaml`** 

Write a branch or a tag and DPM replaces it with the commit SHA that ref pointed at:

```
- git:github.com/Moonsong-Labs/test-daml-hello#master?path=dist/test-daml-hello-sdk-3.5.2-lf-2.2.dar
+ git:github.com/Moonsong-Labs/test-daml-hello#<sha>?path=dist/test-daml-hello-sdk-3.5.2-lf-2.2.dar
```

Both `dpm-dev install` and `dpm-dev add dar` do this. It is pinning, not
corruption: a branch moves, so a build that says "whatever `master` is today"
is not reproducible. In your own project, commit the pinned line, and re-run
`dpm-dev add dar` (or `dpm-dev update`) when you actually want to move. Adding
the *same* DAR from the *same* repo at a different ref updates the existing
line rather than appending a second one, and DPM says so:

```
dependency "git:github.com/canton-network/splice#main?path=daml/dars/splice-amulet-0.1.19.dar" already exists in daml.yaml, will be updated...
```

You may also see DPM flip the `/` separators inside `?path=` between plain and
percent-encoded (`dist/foo.dar` and `dist%2Ffoo.dar`) on the first run after a
pin. Both forms are accepted and it settles after one run, but it will show up
as a diff in your working tree.

### `dependencies` is stricter than `data-dependencies`

At list these two rules apply to every put under `dependencies`:

1. Daml-LF version must equal your build target. Point `dependencies` at
   `test-daml-hello-sdk-3.5.2-lf-2.1.dar` and SDK 3.5.2 (which targets LF 2.2)
   fails with `Targeted LF version 2.2 but dependencies have different LF
   versions`. You can add `build-options: [--target=2.1]` if you want to force your app to use this LF target for all packages. This project uses the LF 2.2 that comes with SDK 3.5.2, so no need to add a specific target.
2. Every DAR in the closure must have been built by the *same SDK version*,
   yours included, or you get `Package dependencies from different SDK
   versions`.

`splice-amulet` fails rules 2 and 3 in a way you cannot fix from your side. It
was built by SDK 3.4.11 and ships `daml-stdlib-3.4.11` inside its own closure,
while this project is on 3.5.2, so rule 2 stops the build with `Package
dependencies from different SDK versions: 3.4.11, 3.5.2`.

## Troubleshooting

| Message | Cause |
| --- | --- |
| `http dependencies not yet supported` | A browser URL in `daml.yaml`. Use `dpm-dev add dar` instead. |
| `?path= is required (e.g. git:github.com/org/repo.git#main?path=loyalty.dar)` | A `git:` line with a ref but no `?path=`. |
| `repo-relative path "README.md" must end with .dar` | `?path=` points at something that is not a DAR. |
| `only https:// clone URLs are supported` | An SSH clone URL. Use the HTTPS form. |
| `parse "git@github.com:org/repo.git": first path segment in URL cannot contain colon` | The scp-style SSH spelling. Write `git:github.com/org/repo` instead. |
| `<host> does not expose the GitHub releases API, so ?release= dependencies are only supported for github.com` | A `?release=` link aimed at a host other than GitHub. |
| `Targeted LF version 2.2 but dependencies have different LF versions` | An LF mismatch under `dependencies`. See [Gotchas](#gotchas). |
| `Package dependencies from different SDK versions` | DARs under `dependencies` built by more than one SDK. Move them to `data-dependencies`. |
| `cannot satisfy --package <name>: unusable due to missing dependencies` | A `dependencies` entry whose transitive closure is not declared. Move it to `data-dependencies`. |
| `release "<tag>" not found for <owner>/<repo>` | A `?release=` tag that does not exist. Check it against the repo's releases page. |
| `asset "<name>" not found in <owner>/<repo> release "<tag>": check that the release tag and the asset name both exist` | An `&asset=` name that is not attached to that release. GitHub answers 404 for an unknown tag and an unknown asset alike, so check both. |

To re-resolve from scratch, delete `.daml/` for a clean build, and
`~/.dpm/cache/git` as well if you want to force a fresh clone.

## Technical specification and testing guide

You can get more details about the git-links specification in the internal docs of the DPM
repository you checked out earlier. From there, you can generate the docs with:

```bash
cd /path/to/dpm
make run-internal-docs
```
