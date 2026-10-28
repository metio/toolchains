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
| `@go` | go, staticcheck, gosec, gofumpt, arch-go, delve |

## Go

The `go` command finds its GOROOT by resolving the symlink breto links it through, so neither GOROOT nor GOPATH needs setting. `go` downloads a newer toolchain on its own when a `go.mod` asks for one, unless `GOTOOLCHAIN=local` is set. With `GOTOOLCHAIN=local`, the pinned version is the one that runs, and a `go.mod` that needs a newer one fails until the group moves.

## Adding a tool

Every member carries the comment Renovate reads, directly above it:

```clojure
;; renovate: datasource=github-releases depName=mvdan/gofumpt extractVersion=^v(?<version>.+)$
"gofumpt@0.12.0"
```

`datasource` and `depName` name where Renovate looks for releases, `extractVersion` turns a tag into the version breto expects, and `versioning` is available for versions that are not semantic. A member needs a breto definition of the same name. CI installs the pinned versions of every group, which checks that each version exists and verifies.

## License

0BSD, see [LICENSES/0BSD.txt](LICENSES/0BSD.txt).
