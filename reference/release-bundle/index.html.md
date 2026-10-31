# Release bundle

> The release bundle: its file, contents, manifest, the checks of a release, make bundle, tools/bundle, and the Firecracker pins.

A release bundle is one file with every file of one server release of hopbox. The same bundle plays every role. `hopboxd install` puts its release on a server, and `hopboxd upgrade` moves a server to it ([hopboxd](/reference/hopboxd/index.html.md#install)). The Go package `internal/bundle` reads, writes, and checks bundles. `tools/bundle` builds them, and `make bundle` runs it.

## File

| Item | Value |
| --- | --- |
| Name | `hopbox-<version>-linux-amd64.tar.gz`, for example `hopbox-v0.1.0-linux-amd64.tar.gz` |
| Format | A tar file in the PAX format, compressed with gzip |
| Top dir | One dir, `hopbox-<version>/`. Every other entry is inside it. |
| Checksum | `make bundle` writes `<name>.sha256` beside the bundle, in the format of `sha256sum`: the hex sha256, two spaces, and the name. |

`<version>` is the semver version of `hopboxd` in the bundle, from `tools/gitversion` ([the version command](/reference/version/index.html.md)). A server keeps each release in `/opt/hopbox/releases/<version>` ([Server layout](/reference/server-layout/index.html.md)).

A bundle is about 600 MB. The compressed disk of the default image is about 560 MB of it.

## Contents

| Path in the bundle | Mode | What it is |
| --- | --- | --- |
| `manifest.json` | 0644 | The [manifest](#manifest). It is the first entry after the top dir. |
| `hopboxd` | 0755 | `hopboxd` for `linux/amd64`, built with `CGO_ENABLED=0` by `make build`. |
| `firecracker/firecracker` | 0755 | Firecracker v1.17.0. |
| `firecracker/jailer` | 0755 | The jailer v1.17.0. |
| `firecracker/LICENSE`, `firecracker/NOTICE`, `firecracker/THIRD-PARTY` | 0644 | The license files of the Firecracker release. The Apache License 2.0 asks for them in each copy. |
| `postgres/` | 0755 and 0644 | The bundled Postgres 18.6 for `linux/amd64`: `bin/`, `lib/`, and `share/` ([the bundled Postgres](/reference/postgres/index.html.md#the-install)). `lib/` also holds the OpenSSL libraries `libssl.so.3` and `libcrypto.so.3`, which `pg_dump` and `pg_restore` need for TLS. It holds symlinks, such as `lib/libpq.so`. |
| `kernel/vmlinux` | 0644 | The guest kernel ([Guest kernel](/reference/guest-kernel/index.html.md)). Its sha256 is the one in `kernel/vmlinux.sha256` of the checkout. |
| `image/artifact.json` | 0644 | The metadata of the default image ([Artifact format](/reference/artifact-format/index.html.md#artifactjson)). |
| `image/disk.raw.zst` | 0644 | The compressed disk of the default image, under the name that `artifact.json` gives it. |
| `systemd/hopboxd.service` | 0644 | The unit of `hopboxd` ([Server layout](/reference/server-layout/index.html.md#systemd-units)). |
| `systemd/hopbox-machines.slice` | 0644 | The slice of the machines. |
| `monitoring/` | 0644 | The files of the monitoring bundle, from `monitoring/` of the checkout. The release does not run them. Today it holds `monitoring/node-exporter/prometheus-node-exporter`, the arguments of node_exporter ([Metrics](/reference/metrics/index.html.md#node_exporter)). |

Every dir has mode 0755. A bundle holds only dirs, regular files, and symlinks.

### The default image in the bundle

The bundle carries the default image, not only a reference to it. These reasons decide it:

- The image is built and tested with one guest kernel, and `artifact.json` names that kernel by its sha256. The bundle checks that the kernel and the image belong together, so they ship together.
- An install needs no download at run time, just as for Postgres. No hosting for releases and images exists yet.
- The image holds `hopbox-guest` of the same checkout, so its digest changes with each commit anyway.

The cost is size: an upgrade uploads the image again, about 560 MB. A server keeps one copy of each image. The install moves the image from the release into the image cache `/var/lib/hopbox/images/<digest>` and keeps no copy in the release dir ([Server layout](/reference/server-layout/index.html.md#the-image-cache)).

## Manifest

`manifest.json` is a JSON object with these fields, in this order. Every field is required.

| Field | Type | Description |
| --- | --- | --- |
| `format_version` | number | `1`. |
| `version` | string | The semver version of the release, such as `v0.1.0`. |
| `os` | string | `linux`. |
| `arch` | string | `amd64`. |
| `postgres_version` | string | The version of the bundled Postgres, such as `18.6`, from `build/postgres/source.env`. |
| `firecracker_version` | string | The Firecracker release, such as `v1.17.0`, from `build/bundle/firecracker.env`. |
| `schema_version` | number | The number of the last schema migration that `hopboxd` of the release knows, at least 1 ([the bundled Postgres](/reference/postgres/index.html.md#schema-migrations)). An upgrade or a rollback refuses a release whose `schema_version` is lower than the oldest schema version of the newest migration of the control store ([hopboxd](/reference/hopboxd/index.html.md#upgrade), [the bundled Postgres](/reference/postgres/index.html.md#two-releases-on-one-store)). |
| `kernel_sha256` | string | The sha256 of `kernel/vmlinux`, as 64 lowercase hex digits. |
| `image_digest` | string | The digest of the default image: `sha256:` and the sha256 of the bytes of `image/artifact.json` ([Artifact format](/reference/artifact-format/index.html.md#digest)). |
| `files` | list | Every file and symlink of the release, sorted by path. |
| `files[].path` | string | The path in the release, with `/`. |
| `files[].link` | string | Only for a symlink: its target, relative to the dir of the path. |
| `files[].mode` | string | Only for a file: `0755` or `0644`. |
| `files[].size` | number | Only for a file: the size in bytes. An empty file leaves it out. |
| `files[].sha256` | string | Only for a file: the sha256, as 64 lowercase hex digits. |

The list leaves out `manifest.json` and the compressed disk of the image. `image_digest` covers the disk, because `artifact.json` holds its sha256.

Example, shortened:

```json
{
  "format_version": 1,
  "version": "v0.1.0",
  "os": "linux",
  "arch": "amd64",
  "postgres_version": "18.6",
  "firecracker_version": "v1.17.0",
  "schema_version": 14,
  "kernel_sha256": "dfb03703bab6d4b590964f2d9473fdc82994ae94384cc5ed1c1b4a0430118a91",
  "image_digest": "sha256:354fb76713c333b4d22c0341cab5f9b04b288eb44bd97110f81ca13bfc260e7c",
  "files": [
    {
      "path": "firecracker/firecracker",
      "mode": "0755",
      "size": 3676680,
      "sha256": "99ad0f5cd0514a88aad0e9ae8cfdb3cc3b4ab9d190e1194602406c786b5de7a5"
    },
    {
      "path": "postgres/lib/libpq.so",
      "link": "libpq.so.5"
    }
  ]
}
```

A reader refuses a manifest in these cases:

- A `format_version` other than `1`.
- A field that it does not know, or a field name in another case.
- A field name that appears twice, or data after the object.
- A `version` that is not semver, a platform other than `linux/amd64`, or a `schema_version` below 1.
- A path that is not clean and relative, that leaves the release, or that has a part that starts with a dot. A path that appears twice, or a list that is not sorted.
- A symlink whose target is not another path of the list.
- A file in `image/` other than `artifact.json`.
- A missing `hopboxd`, `firecracker/firecracker`, `firecracker/jailer`, `postgres/bin/postgres`, `kernel/vmlinux`, `image/artifact.json`, or unit. A mode other than `0755` on `hopboxd`, Firecracker, the jailer, or `postgres`.
- A mode other than `0644` on `kernel/vmlinux`, `image/artifact.json`, or a unit. A kernel build leaves `vmlinux` executable, so the bundle sets `0644` on these files, whatever the mode of their source.
- A `kernel_sha256` that is not the sha256 of `kernel/vmlinux`.

## Checks of a release

An install and an upgrade check a release dir before they use it. The check has these rules:

- Each file of the manifest exists as a regular file, with its mode, its size, and its sha256.
- Each symlink exists with its target.
- The dir holds no other file, dir, or symlink. Only the compressed disk of the image can be there or not.
- `image/artifact.json` has the digest `image_digest`, names the kernel `kernel_sha256`, and names a compressed disk.

As root, the install also checks the owner of the release. Root must own every file and dir, and group and others must not be able to write them. The reason is that `hopboxd` runs as root and runs the programs of its release. The install of a node checks the compressed disk of the image against its sha256 before it puts the image into the image cache.

### Errors

| Error | Cause |
| --- | --- |
| `invalid bundle manifest: ...` | The manifest breaks a rule of the format. The rest of the message names the rule. |
| `<path>: the sha256 is <got>, but the manifest says <want>` | A file differs from the manifest. The field can also be `size`, `mode`, `type`, or `symlink target`. |
| `the release misses a file of its manifest: ...` | A file of the manifest is gone. |
| `the release <dir> holds the file <path>, which its manifest does not name` | An extra file. A dir gives the same error with `the dir`. |
| `image/artifact.json: the digest is <got>, but the manifest says <want>` | The image metadata is not the one of the build. |
| `the default image was built for the kernel <a>, but the release holds the kernel <b>` | The image and the kernel do not belong together. |
| `bundle: the bundle file has the path "<path>", which is not a clean path in the release` | The bundle file holds a path with `..`, an absolute path, or a dot file. Nothing is written outside the target dir. |
| `bundle: <entry>: the entry has the type "<t>", but a bundle holds only dirs, files, and symlinks` | A hard link, a device, or a FIFO. |
| `bundle: <entry>: the symlink points out of the release, to "<target>"` | A symlink with an absolute target or a target outside the release. |
| `bundle: the bundle file has a second top entry ...` | The bundle file has more than one top dir. |
| `bundle: the bundle file unpacks to more than 100000 entries or 4294967296 bytes, more than a release holds` | The bundle file is broken or hostile. A real release has about 750 entries and 700 MB. The extract stops at the limit, so it cannot fill the disk. |

## Build a bundle

```sh
make kernel
make image
make bundle
```

`make bundle` needs the guest kernel in `KERNEL_OUT` and the default image in `IMAGE_OUT`. Each takes many minutes, so `make bundle` does not build them. It builds `hopboxd` for `linux/amd64` with `make build PLATFORMS=linux/amd64`, finds the Linux build of Postgres, and runs `tools/bundle`. It prints the path of the bundle file.

| Variable | Default | Description |
| --- | --- | --- |
| `VERSION` | the output of `tools/gitversion` | The version of the release. |
| `BUNDLE_OUT` | `bin/bundle` | The dir of the bundle file. |
| `PG_LINUX` | the output of `go run ./tools/pgdist -target linux/amd64` | A Linux build of the bundled Postgres. Without it, pgdist builds one with Docker. |
| `KERNEL_OUT` | `bin/kernel` | The dir of `vmlinux`. |
| `IMAGE_OUT` | `bin/image` | The artifact dir of the default image, with `artifact.json` and the compressed disk. |

The time of every tar entry is `DATE`, the commit time of the checkout. Two builds of the same inputs give the same bundle, byte for byte.

## tools/bundle

```sh
go run ./tools/bundle -version V -hopboxd FILE -postgres DIR -kernel FILE -image DIR -out DIR [flags]
```

`tools/bundle` checks each part, then writes the bundle to a temp file in the out dir, renames it into place, and writes the `.sha256` file. It prints the path of the bundle on standard output. Logs go to standard error.

| Flag | Default | Description |
| --- | --- | --- |
| `-version` | none, required | The semver version of the release. |
| `-hopboxd` | none, required | The `hopboxd` binary. |
| `-postgres` | none, required | The Postgres install for `linux/amd64`. |
| `-kernel` | none, required | The guest kernel `vmlinux`. |
| `-image` | none, required | The artifact dir of the default image. |
| `-out` | none, required | The dir of the bundle file. |
| `-mtime` | `1970-01-01T00:00:00Z` | The time of every tar entry, in RFC 3339 or Unix seconds. |
| `-root` | the first parent dir of the current dir with `build/bundle/firecracker.env` | The root of the checkout. |
| `-cache` | `$HOPBOX_BUNDLE_CACHE`, else `hopbox/bundle` in the user cache dir | The download cache. |
| `-firecracker` | none | A local copy of the Firecracker release tarball, in place of a download. |

It checks these parts before it writes anything:

| Part | Check |
| --- | --- |
| `hopboxd` | The build info says `GOOS=linux`, `GOARCH=amd64`, and `CGO_ENABLED=0`. The variable `version` of the package `internal/version` of the Go module of hopbox, which `make build` sets with `-ldflags -X`, holds exactly `-version`. The tool reads it from the ELF symbol table, because a `-trimpath` build records no `-ldflags` in its build info. |
| Console | `hopboxd` holds the console build of `make console`: its bytes contain the line `<meta name="application-name" content="hopbox console" />` of the `index.html` of the console ([Console](/reference/console/index.html.md#build)). No Go code of `hopboxd` has that line, so a `hopboxd` of `go build` without `make console` fails. |
| Kernel | Its sha256 is the `vmlinux` line of `kernel/vmlinux.sha256`, the hash of the reproducible build. |
| Image | `artifact.json` is valid, names a compressed disk with the right size and sha256, and names the kernel of the bundle. Its arch is `x86_64`, and its program `hopbox-guest` has the version `-version`: the guest agent and `hopboxd` of a release ship together. |
| Postgres | The dir has `bin/postgres`, and its name is the one that `tools/pgdist -target linux/amd64` gives the build of the current recipe: `postgres-<version>-linux-amd64-<hash>` ([the bundled Postgres](/reference/postgres/index.html.md#pgdist)). So a build for another platform or of another recipe fails. The version comes from `build/postgres/source.env`. |
| Schema | `make build` counts the files in `internal/store/migrations` and sets the variable `schemaStamp` of the package `internal/store` in `hopboxd` with `-ldflags -X`. The tool reads this variable from the ELF symbol table and uses it as `schema_version`. It refuses a `hopboxd` without the stamp, and a stamp that differs from the migrations of the checkout. At its start, `hopboxd` refuses a stamp that differs from the migrations that it embeds. |
| Firecracker | The release tarball has the pinned sha256, and `firecracker` and the jailer in it have their pinned sha256. |

### Firecracker pins

`build/bundle/firecracker.env` pins the Firecracker release, one `key=value` per line. Lines that start with `#` are comments.

| Key | Value | Description |
| --- | --- | --- |
| `version` | `v1.17.0` | The release. |
| `url` | `https://github.com/firecracker-microvm/firecracker/releases/download/v1.17.0/firecracker-v1.17.0-x86_64.tgz` | The release tarball. |
| `sha256` | `06094a1108ae9e82aa4c23a775aa92758f53f1175d422270d9d6162cb9ade558` | The sha256 of the tarball. It matches the `.sha256.txt` file of the release. |
| `firecracker_sha256` | `99ad0f5cd0514a88aad0e9ae8cfdb3cc3b4ab9d190e1194602406c786b5de7a5` | The sha256 of `firecracker-v1.17.0-x86_64` in the tarball. |
| `jailer_sha256` | `65ef226e96f0ceda55ba643f445801ef2cc0ea667ef67cad8ac4f406c9c8434f` | The sha256 of `jailer-v1.17.0-x86_64` in the tarball. |

The download goes to `<cache>/firecracker/`. A cached tarball with the pinned sha256 is not downloaded again.

### Errors of tools/bundle

| Error | Cause |
| --- | --- |
| `-kernel: open ...: no such file or directory. Run make kernel first` | No kernel in `KERNEL_OUT`. |
| `-image: read artifact metadata: .... Run make image first` | No image in `IMAGE_OUT`. |
| `-kernel <path> has the sha256 <a>, but kernel/vmlinux.sha256 pins <b>. Run make kernel on this checkout` | The kernel is from another recipe. |
| `-hopboxd <path> carries the version "<got>", not <v>. Build it with make build VERSION=<v>` | The binary is from another build. |
| `-postgres <dir> is not the output of tools/pgdist -target linux/amd64 for the recipe in <recipe>, which is <name>. Run make pg-linux` | The Postgres install is from another platform or recipe. |
| `-image <dir>: the image holds hopbox-guest <a>, not <v>. Build it with make image VERSION=<v>` | The image is from another build. |
| `-hopboxd <path> knows <n> schema migrations, and this checkout has <m>. Build it from this checkout with make build` | The binary is from another checkout. |
| `-hopboxd <path>: read its schema version: the binary has no symbol ...` | The binary was not built with `make build`. |
| `-hopboxd <path> is built for <os>/<arch>, not linux/amd64` | The binary is for another platform. |
| `-hopboxd <path> holds no console. Build it with make build, which runs make console first` | The binary was built without the console. |
| `-hopboxd <path> is built with cgo. ...` | The binary is not static. |
| `bundle: the image in <dir> was built for the kernel <a>, but the kernel <path> has the sha256 <b>. Build the image with this kernel` | The image and the kernel do not belong together. |
| `-firecracker <path> has the sha256 <a>, and it does not have the pinned sha256 <b>` | A local tarball that is not the pinned release. |
| `download <url>: the sha256 is <a>, but build/bundle/firecracker.env pins <b>` | The download is not the pinned release. |
| `<entry> in <tarball> has the sha256 <a>, but build/bundle/firecracker.env pins <b>` | A binary in the tarball differs from its pin. |

The exit status is 0 on success, 1 on an error, and 2 for a bad command line.

## Tests

| Test | Where it runs | What it checks |
| --- | --- | --- |
| `internal/bundle` | `make test` | A build, an extract, and the checks of a release. Two builds give the same bytes. A changed file, mode, symlink, or image, and an extra file or dir, fail the check. Unsafe tar entries fail the extract. Bad manifests fail the parse. |
| `tools/bundle` | `make test` | A bundle from stand-in parts with pins of a test checkout, and the refusal of a wrong version, kernel, Firecracker tarball, and platform. |
| `TestSystemdServer` | `make test-install`, and the CI job `install` | An install, an upgrade, and a rollback in a systemd container ([hopboxd](/reference/hopboxd/index.html.md#tests-of-install-and-upgrade)). |

The package `internal/bundle/bundletest` makes bundles for tests from small stand-ins of Firecracker, the jailer, the kernel, Postgres, and the image.
