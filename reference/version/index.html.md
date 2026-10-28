# The version command

> The version command of every hopbox binary, its output, and where its values come from.

Every hopbox binary has a `version` command: `hopboxd`, `hopbox`, `terraform-provider-hopbox`, and `hopbox-guest`. The command prints the build identity of the binary and exits.

## Usage

```sh
hopboxd version [--json]
hopbox version [--json]
terraform-provider-hopbox version [--json]
hopbox-guest version [--json]
```

## Flags

| Flag | Type | Default | Description |
| --- | --- | --- | --- |
| `--json` | bool | `false` | Prints one JSON object instead of one line of text. |
| `-h`, `--help` | bool | `false` | Prints the help of the command and exits. |

## Text output

The text output is one line:

```text
<binary> <version> (commit <commit>, built <date>, <go> <os>/<arch>)
```

Example:

```sh
$ hopboxd version
hopboxd v0.1.0 (commit 1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b, built 2026-10-16T09:00:00Z, go1.27.1 linux/amd64)
```

If the binary has no commit or no date, the line leaves out that part.

## JSON output

The JSON output is one object on one line, with these fields:

| Field | Type | Always present | Description |
| --- | --- | --- | --- |
| `binary` | string | yes | The name of the binary, for example `hopboxd`. |
| `version` | string | yes | The version of the build. See [Where the values come from](#where-the-values-come-from). |
| `commit` | string | no | The full hash of the git commit of the build. |
| `date` | string | no | The time of that commit, in RFC 3339 format. |
| `go` | string | yes | The Go version that built the binary, for example `go1.27.1`. |
| `os` | string | yes | The operating system of the build, for example `linux`. |
| `arch` | string | yes | The CPU architecture of the build, for example `amd64`. |
| `features` | array of strings | no | The capabilities that a tool reads from the version. Only `hopboxd` has the field: `host-firewall` from the release with the host firewall on ([the host firewall](/reference/host-firewall/index.html.md)), `egress` from the release with the egress tunnels on ([the egress of the machines](/reference/egress/index.html.md)), and `peer-address` from the release that runs the mesh and the control channel on the peer addresses ([the mesh](/reference/mesh/index.html.md)). The Terraform provider reads it after each move of a server to another release, and from the `hopboxd` of a bundle before its install. |

Example:

```sh
$ hopbox version --json
{"binary":"hopbox","version":"v0.1.0","commit":"1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b","date":"2026-10-16T09:00:00Z","go":"go1.27.1","os":"darwin","arch":"arm64"}
```

## Where the values come from

`make build` sets the version, the commit, and the date at link time:

- `version` comes from `go run ./tools/gitversion`. It is always a semver version, so versions sort in commit order. The rules are in the next section.
- `commit` is the full hash of `HEAD`.
- `date` is the commit time of `HEAD`, not the time of the build. So two builds of the same commit print the same date.

A binary from `go build` or `go install` gets the values from the build information that the Go toolchain writes into the binary. The version is then the module version, for example `v0.0.0-20261016090000-1a2b3c4d5e6f+dirty`. When the toolchain gives no version, as with `go run`, the version is `v0.0.0-0.unknown`, and the commit and the date are absent. This version sorts below every other version.

## Version format

The version follows the rules of Go module pseudo-versions. Only tags that match `v[0-9]*` count. The time is the commit time of `HEAD` in UTC, as `yyyymmddhhmmss`. The hash is the first 12 characters of the commit hash.

| State of the checkout | Version | Example |
| --- | --- | --- |
| `HEAD` has the tag `vX.Y.Z` | `vX.Y.Z` | `v0.1.0` |
| `HEAD` is after the release tag `vX.Y.Z` | `vX.Y.(Z+1)-0.<time>-<hash>` | `v0.1.1-0.20261017101112-1a2b3c4d5e6f` |
| `HEAD` is after the pre-release tag `vX.Y.Z-<pre>` | `vX.Y.Z-<pre>.0.<time>-<hash>` | `v0.2.0-rc.1.0.20261017101112-1a2b3c4d5e6f` |
| The history has no `v*` tag | `v0.0.0-<time>-<hash>` | `v0.0.0-20261016090807-1a2b3c4d5e6f` |

If a tracked file holds changes that are not committed, the version ends in `+dirty`, for example `v0.1.0+dirty`. Files that git does not track do not count.

A version after a tag sorts above that tag and below the next patch release. For example, `v0.1.0` < `v0.1.1-0.20261017101112-1a2b3c4d5e6f` < `v0.1.1`.

If the nearest tag is not a full semver version, such as `v1.2` or `v1.2.3+build`, `make build` fails. In a shallow clone, `make build` also fails, because the nearest tag is unknown. Run `git fetch --unshallow --tags` first. Outside a git checkout, `make build` also fails. In these cases, you can set the version yourself:

```sh
make build VERSION=v0.1.0
```
