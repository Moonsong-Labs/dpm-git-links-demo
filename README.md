# dpm-git-links-demo

A tiny Daml project aimed to test git-like links resolution in DPM.

The `daml.yaml` has two unusual lines:

```yaml
data-dependencies:
  - "git:github.com/canton-network/splice#63cfb340ce0f8f254386d2d5df58905d695de902?path=daml/dars/splice-amulet-0.1.19.dar"
  - "git:github.com/gonzamontiel/test-daml-hello#1c7c5d4776cdb6f57139c7f7fe674e137e79c1e2?path=dist/test-daml-hello-sdk-3.5.2-lf-2.1.dar"
```

Each line points at a `.dar` file sitting in someone else's git repository. DPM
clones the repo, pulls out the DAR, and hands it to the compiler. There is no
vendored binary in this repo, no registry to publish to, and no download step
for you to run. `daml/Main.daml` imports `Hello.greeting` from one DAR and the
`Splice.Amulet.Amulet` template type from the other, so if either link stops
resolving, the build stops working. That is deliberate: it keeps this demo
honest.

Clone it, run three commands, then look at [`samples/`](samples) for every other
way you can write the same kind of link.

## What you need

- A locally built `dpm-dev`. Git links are not in a released DPM yet, so you
  need to use the [Moonsong Labs fork](https://github.com/Moonsong-Labs/dpm/) at the `with-data-deps-v1` tag:

  ```bash
  git clone --branch with-data-deps-v1 git@github.com:Moonsong-Labs/dpm.git
  cd dpm
  go build -o bin/dpm-dev ./cmd/dpm/
  export PATH="$PWD/bin:$PATH"
  ```

  Building needs Go 1.25.0. The `export` puts `dpm-dev` on your `PATH` for the
  current shell only; add the same line to your `~/.zshrc` or `~/.bashrc` with
  the real path in place of `$PWD` to make it stick. Otherwise skip it and call
  the binary by path, as `/path/to/dpm/bin/dpm-dev`.
- Network access to `github.com` the first time, so DPM can fetch the DARs.
  After that they are cached under `~/.dpm/cache`.
- There's no need to recompile damlc. So no Bazel or Nix are required.
- This project pins `sdk-version: 3.5.2`, an ordinary released SDK, and DPM installs it for you.

## Quick start

```bash
dpm-dev install package
dpm-dev build
dpm-dev test
```

`dpm-dev install package` resolves both git dependencies and caches the DARs:

```
SDK version 3.5.2 is already installed
installing git dar "git:github.com/canton-network/splice#63cfb340ce0f8f254386d2d5df58905d695de902?path=daml%2Fdars%2Fsplice-amulet-0.1.19.dar"...
Resolving git dependency: fetching https://github.com/canton-network/splice @ "63cfb340ce0f8f254386d2d5df58905d695de902" path "daml/dars/splice-amulet-0.1.19.dar"
installing git dar "git:github.com/gonzamontiel/test-daml-hello#1c7c5d4776cdb6f57139c7f7fe674e137e79c1e2?path=dist%2Ftest-daml-hello-sdk-3.5.2-lf-2.1.dar"...
Resolving git dependency: fetching https://github.com/gonzamontiel/test-daml-hello @ "1c7c5d4776cdb6f57139c7f7fe674e137e79c1e2" path "dist/test-daml-hello-sdk-3.5.2-lf-2.1.dar"
No opt-in components to install
```

The two `Resolving` lines only appear when the DAR is not cached yet. Both refs
are already commit SHAs in the committed `daml.yaml`, so nothing is rewritten
and your working tree stays clean.

`dpm-dev build` compiles, including the module that imports from both git-hosted
DARs:

```
Created .daml/dist/dpm-git-links-demo-0.0.1.dar
```

It also prints a warning that this package defines templates and depends on
`daml-script`. That is normal for a single-package demo and unrelated to git
links.

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
- HTTPS only. `git:ssh://…` is rejected.

Put it under `data-dependencies`. It is also accepted under `dependencies`, but
that field carries constraints most published DARs cannot meet, so treat it as
the exception rather than the default (see [Gotchas](#gotchas)).

For a GitHub release asset, swap `#<ref>?path=` for a release query:

```
git:<host>/<repo>?release=<tag>&asset=<file>.dar
```

Omit `&asset=` and DPM expands the entry into one line per `.dar` attached to
that release.

### Or let DPM write the line

You do not have to assemble that string by hand. Find the DAR on GitHub, copy
the URL out of your address bar, and hand it over:

```bash
dpm-dev add dar \
  'https://github.com/canton-network/splice/raw/refs/tags/0.6.10/daml/dars/splice-amulet-0.1.19.dar' \
  --data-dependencies
```

DPM normalizes the browser URL, resolves the ref, downloads the DAR, and writes
the canonical line into `daml.yaml` for you. This is the shortest path from "I
found a DAR on GitHub" to "my project builds against it".

## Samples

One file per form, each a `daml.yaml` fragment with a header explaining what it
shows and when you would reach for it. Every command output, normalized `git:`
line and error message quoted in them was captured by running DPM, not written
from the docs.

| Sample | Form |
| --- | --- |
| [`browser-url-tag.yaml`](samples/browser-url-tag.yaml) | GitHub `raw` URL at a release tag, via `dpm-dev add dar` |
| [`browser-url-branch.yaml`](samples/browser-url-branch.yaml) | GitHub `raw` URL on a branch, via `dpm-dev add dar` |
| [`browser-url-commit.yaml`](samples/browser-url-commit.yaml) | GitHub `blob` URL at a commit SHA, via `dpm-dev add dar` |
| [`git-line-data-dependencies.yaml`](samples/git-line-data-dependencies.yaml) | The canonical `git:` one-liner under `data-dependencies` |
| [`git-line-dependencies.yaml`](samples/git-line-dependencies.yaml) | The same line under `dependencies`, plus the Daml-LF catch |
| [`git-line-dependencies-splice.yaml`](samples/git-line-dependencies-splice.yaml) | A DAR that `dependencies` rejects outright, and why no SDK pin saves it |
| [`structured-yaml.yaml`](samples/structured-yaml.yaml) | Structured `url` / `ref` / `path` mapping: installs, then fails to build |
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

Four things that look like bugs the first time. One of them is.

**A browser URL works with `dpm-dev add dar` but not in `daml.yaml`.** Pasting
one straight into a dependency list fails:

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

**DPM rewrites your `daml.yaml`, on purpose.** Write a branch or a tag and DPM
replaces it with the commit SHA that ref pointed at:

```
- "git:github.com/gonzamontiel/test-daml-hello.git#master?path=dist/test-daml-hello-sdk-3.5.2-lf-2.1.dar"
+ "git:github.com/gonzamontiel/test-daml-hello#1c7c5d4776cdb6f57139c7f7fe674e137e79c1e2?path=dist%2Ftest-daml-hello-sdk-3.5.2-lf-2.1.dar"
```

Both `dpm-dev install` and `dpm-dev add dar` do this. It is pinning, not
corruption: a branch moves, so a build that says "whatever `master` is today"
is not reproducible. Commit the pinned line, and re-run `dpm-dev add dar` (or
`dpm-dev update`) when you actually want to move. Adding the *same* DAR from
the *same* repo at a different ref updates the existing line rather than
appending a second one, and DPM says so:

```
dependency "git:github.com/canton-network/splice#main?path=daml/dars/splice-amulet-0.1.19.dar" already exists in daml.yaml, will be updated...
```

You may also see DPM flip the `/` separators inside `?path=` between plain and
percent-encoded (`dist/foo.dar` and `dist%2Ffoo.dar`) on the first run after a
pin. Both forms are accepted and it settles after one run, but it will show up
as a diff in your working tree.

**`dependencies` is far stricter than `data-dependencies`, and the Daml-LF
version is the least of it.** Three separate rules apply to every DAR you put
there, and a published library usually fails at least one:

1. Its Daml-LF version must equal your build target. The `Hello` DAR is LF 2.1
   and SDK 3.5.2 targets LF 2.2, so it fails with `Targeted LF version 2.2 but
   dependencies have different LF versions` until you add
   `build-options: [--target=2.1]`.
2. Every DAR in the closure must have been built by the *same SDK version*,
   yours included, or you get `Package dependencies from different SDK
   versions`.
3. The full transitive closure must be declared explicitly, even though the DAR
   already contains every dalf it needs.

`splice-amulet` fails rules 2 and 3 in a way you cannot fix from your side. It
was built by SDK 3.4.11 and ships `daml-stdlib-3.4.11` inside its own closure,
while this project is on 3.5.2, so rule 2 stops the build with `Package
dependencies from different SDK versions: 3.4.11, 3.5.2`. Pinning your project
to 3.4.11 to match does not rescue it: on that SDK, git links are not resolved
at all, and `damlc` is handed the raw link as if it were a filename. So 3.5.2
resolves the link and then rejects the DAR, and 3.4.11 would accept the DAR and
never resolves the link. `data-dependencies` applies none of these rules — it
reconstructs the interface from the DAR — which is why this project puts both of
its links there and needs no `build-options` at all. See
[`git-line-dependencies-splice.yaml`](samples/git-line-dependencies-splice.yaml).

**The structured YAML form installs but will not build.** `dpm-dev install
package` accepts a `- git:` mapping with `url` / `ref` / `path` keys, resolves
it, pins the ref and writes the file back. `dpm-dev build` then rejects the
file DPM itself just wrote:

```
damlc: ConfigFieldInvalid "package" ["data-dependencies"] "Error in $[0]: expected String, but encountered Object"
```

Use the one-liner. See
[`structured-yaml.yaml`](samples/structured-yaml.yaml).

## Troubleshooting

| Message | Cause |
| --- | --- |
| `http dependencies not yet supported` | A browser URL in `daml.yaml`. Use `dpm-dev add dar` instead. |
| `?path= is required (e.g. git:github.com/org/repo.git#main?path=loyalty.dar)` | A `git:` line with a ref but no `?path=`. |
| `repo-relative path "README.md" must end with .dar` | `?path=` points at something that is not a DAR. |
| `only https:// clone URLs are supported` | An SSH clone URL. Use the HTTPS form. |
| `Targeted LF version 2.2 but dependencies have different LF versions` | An LF mismatch under `dependencies`. See [Gotchas](#gotchas). |
| `Package dependencies from different SDK versions` | DARs under `dependencies` built by more than one SDK. Move them to `data-dependencies`. |
| `cannot satisfy --package <name>: unusable due to missing dependencies` | A `dependencies` entry whose transitive closure is not declared. Move it to `data-dependencies`. |
| `ConfigFieldInvalid "package" ["data-dependencies"] … expected String, but encountered Object` | The structured `url` / `ref` / `path` mapping form. Use the one-liner. |
| A wall of HTML ending in `HTTP 404` | A `?release=` tag or `&asset=` name that does not exist. DPM currently echoes GitHub's 404 page; check the tag and asset names on the release page. |

To re-resolve from scratch, delete `.daml/` for a clean build, and
`~/.dpm/cache/git` as well if you want to force a fresh clone.

## Notes

The `Hello` fixture lives in
[`gonzamontiel/test-daml-hello`](https://github.com/gonzamontiel/test-daml-hello),
a personal repository. The splice link carries the demo on its own, so the
walkthrough no longer depends solely on one person's account, but before this
is published the fixture should still be mirrored under a neutral organization
or dropped.

If you need a development-head SDK (`sdk-version: 0.0.0`), that is a
DPM-contributor workflow and out of scope here.
