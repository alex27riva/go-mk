# go-mk

One `common.mk`, shared by all my Go projects. Each repo keeps a committed copy
and a three-line `Makefile`; `make update-mk` pulls in changes made here.

## Adding it to a project

Create a `Makefile` in the project root:

```make
common.mk:
	@curl -fsSL https://raw.githubusercontent.com/CHANGE-ME/go-mk/main/common.mk -o $@

include common.mk
```

Then run `make`. GNU make auto-remakes included files, so the first invocation
downloads `common.mk` and restarts itself — no bootstrap script. Commit both
files.

That is the entire setup for a project laid out as `cmd/<binary>/main.go` with
an `internal/version` package: `MODULE`, `BINARY` and `CMD_PATH` are derived
from `go.mod`.

## Targets

| Target      | Does                                       |
|-------------|--------------------------------------------|
| `help`      | List targets (default goal)                |
| `build`     | Build into `bin/` with version ldflags     |
| `run`       | `go run` the app; args via `ARGS="..."`    |
| `test`      | `go test -race -count=1 ./...`             |
| `cover`     | Coverage profile plus a total line         |
| `lint`      | `golangci-lint run ./...`                  |
| `tidy`      | `go mod tidy` and `go mod verify`          |
| `ci`        | `tidy lint test`                           |
| `clean`     | Remove `bin/` and `coverage.out`           |
| `update-mk` | Re-download `common.mk` from this repo     |

## Version injection

`build` and `run` inject build metadata into `internal/version`. The package
needs exported `var` strings — with `const`, the linker silently does nothing:

```go
package version

var (
	Version   = "dev"
	GitCommit = "none"
	BuildDate = "unknown"
)
```

Check it worked:

```sh
make build && go version -m ./bin/<binary>
```

## Customising a project

Set variables **before** `include common.mk`; everything in the file uses `?=`,
so your value wins. Command-line assignments (`make build VERSION=1.2.3`) beat
both.

| Variable          | Default                        | Use                                  |
|-------------------|--------------------------------|--------------------------------------|
| `BINARY`          | module name                    | Output binary name                   |
| `CMD_PATH`        | `./cmd/$(BINARY)`              | Non-standard main package location   |
| `BUILD_DIR`       | `bin`                          | Output directory                     |
| `VERSION_PKG`     | `$(MODULE)/internal/version`   | Version package lives elsewhere      |
| `LDFLAGS_VERSION` | the three `-X` flags           | Set empty to disable injection       |
| `EXTRA_LDFLAGS`   | empty                          | Add flags without restating the base |
| `LDFLAGS`         | `-s -w` + the above            | Replace everything (use `=`, not `:=`) |
| `TEST_FLAGS`      | `-race -count=1`               | e.g. add `-timeout 5m`               |
| `VERSION`         | `git describe`                 | Pin a version                        |

Project-specific targets go in the project `Makefile`, after the include:

```make
.PHONY: docker
docker: ## Build the image
	docker build --build-arg VERSION=$(VERSION) -t $(BINARY):$(VERSION) .
```

If you override `LDFLAGS` wholesale, use `=` rather than `:=` — `VERSION` and
friends are defined inside `common.mk`, so `:=` would expand them to empty
strings before they exist.

Inspect what a variable resolves to:

```sh
make -n build
```

## Changing common.mk

Edit it here, push, then in each project:

```sh
make update-mk
git diff common.mk
```

Projects track `main`, so an update lands only when you ask for it. Nothing is
pinned or versioned — if that ever becomes a problem, tag releases and point
`MK_SOURCE` at a tag instead.

## Requirements

GNU make, Go, git. `lint` also needs `golangci-lint` on `PATH`.
