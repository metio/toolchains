<!--
SPDX-FileCopyrightText: The toolchains Authors
SPDX-License-Identifier: 0BSD
-->

# toolchains

Shared toolchains for [breto](https://github.com/metio/breto): each `@NAME.edn` file is a breto group that pins the versions of the tools one kind of project needs. Renovate keeps the versions current, and every project naming the group follows.

## Usage

Add the published registry to the `:registries` of the breto config:

```clojure
{:registries [{:github "metio/toolchains"}]}
```

breto installs the latest signed release of this repository before the other packages on `breto update` and `breto sync`. In GitHub Actions, the breto action takes the same registry as a line of its `registries` input:

```yaml
- uses: metio/breto@VERSION
  with:
    registries: github:metio/toolchains
```

A clone works too, as a directory under `:registries`, which is how a change to a group is tried before it is released.

The groups name packages that breto defines; a breto release carries those definitions, so no clone of breto is needed. A group member whose definition only a newer breto release has fails as an unknown package until breto is updated; `breto update` keeps it current when `breto` is in `:packages`.

A project then names the group in its `.breto.edn`:

```clojure
{:packages ["@go"]}
```

`breto env`, or the prompt hook in breto's `shell/env-hook.bash`, links the pinned versions into the project's bin directory. After a release brings a new version, `breto update` or `breto sync` installs it and moves every project using the group to it.

| Group | Tools |
| --- | --- |
| `@go` | go, staticcheck, gosec, gofumpt, arch-go, delve, govulncheck, gopls, modernize |
| `@lint` | uv, reuse, yamllint, typos, actionlint, shellcheck, harper-cli, zizmor, node, markdownlint-cli2 |
| `@helm` | helm, helm-schema, helm-unittest |
| `@babashka` | babashka |
| `@containers` | hadolint |

## Releases

Every change to a group or to `breto.lock` that reaches `main` is released under a calendar version, once `breto lock . --check` passes. A release carries `toolchains-VERSION.tar.gz` with the group files and the lock file, `toolchains_VERSION_SHA256SUMS`, and its cosign bundle `toolchains_VERSION_SHA256SUMS.bundle`, signed keyless by the release workflow.

## Lock file

`breto.lock` records the hash of every version the groups pin: the SHA-256 of the release asset, or for a tool built with `go install`, the module's `h1:` sum from the Go checksum database. breto reads the lock files of all registry directories, and an install of a pinned version whose download does not match its entry fails. A pinned version without an entry installs with a warning, or fails when the breto config sets `:require-locked true`.

CI writes the file, so a Renovate bump needs nothing run locally:

1. Renovate opens a pull request that changes a version in a group.
2. The `Lock` workflow installs the pinned versions, writes `breto.lock` and pushes it to the pull request's branch with a GitHub App token. A push made with the workflow's own token would start no further workflows, so the checks would never rerun.
3. The `Verify` workflow reruns on that push. `breto lock . --check` fails while the file lags behind the groups, and an existing entry whose upstream asset has changed fails the install instead of being locked anew.
4. Renovate merges once the checks pass, and machines pick the change up with their next pull and `breto sync`.

The App needs the `Contents: write` permission on this repository. Its client ID goes into the repository variable `LOCK_APP_CLIENT_ID` and a private key into the secret `LOCK_APP_PRIVATE_KEY`. The workflow only runs for branches named `renovate/*` in this repository, and on manual dispatch, which locks the branch it runs on; a dispatch on `main` creates the first lock file.

## Lint

`@lint` holds the checks most repositories run in CI. reuse and yamllint come from PyPI, so breto installs them with uv, which the group pins as well. markdownlint-cli2 comes from npm, so breto installs it with the group's node, which also runs it. Their lock entries pin every dependency by hash together with the cut-off time of the resolution.

The group also defines the commands that run these checks the way CI runs them. In a project naming `@lint`, `breto run ci-lint` runs every one of them, and `ci-reuse`, `ci-typos`, `ci-yaml`, `ci-actionlint`, `ci-markdown` and `ci-harper` run one each, with arguments passed on. `ci-markdown` lints the Markdown git tracks, so ignored directories such as `node_modules` stay out of it. `ci-harper` lints the prose under `docs` and `kb`, or the whole tree when neither exists, with spell checking and title case off; `HARPER_IGNORE` adds further rules to skip. The `Verify` workflow runs `ci-lint` on this repository through its own `.breto.edn`, which names `@lint` and skips the harper rules that lowercase tool names trip.

## Helm

helm-unittest is a helm plugin and links no command. In a project using `@helm`, `breto env` sets `HELM_PLUGINS` to a directory holding the project's plugins, where `helm unittest` finds it. Outside a project, helm does not see the plugin.

## Go

The `go` command finds its GOROOT by resolving the symlink breto links it through, so neither GOROOT nor GOPATH needs setting. `go` downloads a newer toolchain on its own when a `go.mod` asks for one, unless `GOTOOLCHAIN=local` is set. With `GOTOOLCHAIN=local`, the pinned version is the one that runs, and a `go.mod` that needs a newer one fails until the group moves. govulncheck, gopls and modernize are built by the group's own `go` with `GOTOOLCHAIN=local` and `CGO_ENABLED=0`, so they need no C compiler.

## Adding a tool

Every member carries the comment Renovate reads, directly above it:

```clojure
;; renovate: datasource=github-releases depName=mvdan/gofumpt extractVersion=^v(?<version>.+)$
"gofumpt@0.12.0"
```

`datasource` and `depName` name where Renovate looks for releases, `extractVersion` turns a tag into the version breto expects, and `versioning` is available for versions that are not semantic. A member needs a breto definition of the same name. Tools that breto builds with `go install`, such as govulncheck, use `datasource=go` with the module path as `depName`; the Go proxy lists versions with a leading `v`, which `extractVersion` removes.

## License

0BSD, see [LICENSES/0BSD.txt](LICENSES/0BSD.txt).
