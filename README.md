<!--
SPDX-FileCopyrightText: The toolchains Authors
SPDX-License-Identifier: 0BSD
-->

# toolchains

Shared toolchains for [breto](https://github.com/metio/breto): each `@NAME.edn` file is a breto group that pins the versions of the tools one kind of project needs. Renovate keeps the versions current, and every project naming the group follows.

## Usage

Clone the repository and add the clone to the `:registries` of the breto config, after the directory holding the package definitions:

```clojure
{:registries ["/home/me/git/github.com/metio/breto/packages"
              "/home/me/git/github.com/metio/toolchains"]}
```

A project then names the group in its `.breto.edn`:

```clojure
{:packages ["@go"]}
```

`breto env`, or the prompt hook in breto's `shell/env-hook.bash`, links the pinned versions into the project's bin directory. After a pull brings a new version, `breto update` or `breto sync` installs it and moves every project using the group to it.

| Group | Tools |
| --- | --- |
| `@go` | go, staticcheck, gosec, gofumpt, arch-go, delve, govulncheck, gopls, modernize |

## Lock file

`breto.lock` records the hash of every version the groups pin: the SHA-256 of the release asset, or for a tool built with `go install`, the module's `h1:` sum from the Go checksum database. breto reads the lock files of all registry directories, and an install of a pinned version whose download does not match its entry fails. A pinned version without an entry installs with a warning, or fails when the breto config sets `:require-locked true`.

CI writes the file, so a Renovate bump needs nothing run locally:

1. Renovate opens a pull request that changes a version in a group.
2. The `Lock` workflow installs the pinned versions, writes `breto.lock` and pushes it to the pull request's branch with a GitHub App token. A push made with the workflow's own token would start no further workflows, so the checks would never rerun.
3. The `Verify` workflow reruns on that push. `breto lock . --check` fails while the file lags behind the groups, and an existing entry whose upstream asset has changed fails the install instead of being locked anew.
4. Renovate merges once the checks pass, and machines pick the change up with their next pull and `breto sync`.

The App needs the `Contents: write` permission on this repository. Its client ID goes into the repository variable `LOCK_APP_CLIENT_ID` and a private key into the secret `LOCK_APP_PRIVATE_KEY`. The workflow only runs for branches named `renovate/*` in this repository, and on manual dispatch, which locks the branch it runs on; a dispatch on `main` creates the first lock file.

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
