# Artifact format

> The artifact format version 1 for images: the files, artifact.json, the digest, the Go API, and tools/imageartifact.

An artifact is a folder that holds a disk and its metadata. Images, snapshots, and checkpoints share this format. Format version 1 has one kind, `image`. An image has no memory file. Snapshots and checkpoints come in phase 3, with a new format version.

The Go package `internal/artifact` reads, writes, and checks artifacts. It uses only the Go standard library, so the control role and the node role can both use it.

## Files

| File | Required | What it holds |
| --- | --- | --- |
| `artifact.json` | yes | The metadata. |
| the disk file | yes | The disk, as raw bytes. The default image calls it `disk.raw`. |
| the compressed copy | no | A zstd copy of the disk, for downloads. The default image calls it `disk.raw.zst`. |

The metadata names each file. Each file must be a regular file: the checks refuse a symlink. A file name is a plain name in the folder: it has no `/` or `\`, it does not start with a dot, and it is not `artifact.json`. So metadata can never point at a file outside the folder.

## The disk

| Item | Rule |
| --- | --- |
| Format | Raw: the bytes of the block device, in a sparse file. A hole reads as zeros. |
| Partitions | None. The file system fills the whole disk. Firecracker gives the disk to the guest as `/dev/vda`. |
| File system | ext4 |
| Size | A multiple of 512 bytes, the sector size of the virtio disk. |
| sha256 | The sha256 of all bytes of the disk, with each hole read as zeros. |

A node makes the disk of a machine from the image disk with a reflink copy, and makes the copy bigger. At boot, `systemd-growfs` in the guest grows the file system to fill the disk ([Default image](/reference/default-image/index.html.md#disk)).

## artifact.json

`artifact.json` is a JSON object with these fields, in this order. Every field is required, apart from `disk.compressed`.

| Field | Type | Description |
| --- | --- | --- |
| `format_version` | number | `1`. |
| `kind` | string | `image`. |
| `arch` | string | `x86_64`, the CPU architecture of the programs on the disk. A node refuses an artifact of another architecture. |
| `name` | string | The name of the artifact, for example `default`. It is 1 to 63 lowercase letters, digits, dots, and hyphens, and it starts and ends with a letter or digit. Each build of an image has the same name, so the name does not identify one build. The digest does. |
| `created_at` | string | The build time, in RFC 3339 format in UTC with the suffix `Z`, for example `2026-10-16T00:00:00Z`. A time with an offset, also `+00:00`, is not valid. A reproducible build sets it from its `SOURCE_DATE_EPOCH`. |
| `disk.file` | string | The name of the disk file. |
| `disk.format` | string | `raw`. |
| `disk.filesystem` | string | `ext4`. |
| `disk.size` | number | The size of the disk in bytes. |
| `disk.sha256` | string | The sha256 of the disk, as 64 lowercase hex digits. |
| `disk.compressed.file` | string | The name of the compressed copy. It differs from `disk.file`. |
| `disk.compressed.codec` | string | `zstd`. |
| `disk.compressed.size` | number | The size of the compressed file in bytes. |
| `disk.compressed.sha256` | string | The sha256 of the compressed file, as 64 lowercase hex digits. |
| `kernel.sha256` | string | The sha256 of the guest kernel `vmlinux` that the image was built for and tested with ([Guest kernel](/reference/guest-kernel/index.html.md#outputs)). A node refuses to boot the image with a kernel of another sha256. |
| `programs` | list | The program manifest: each program of the image with its version, sorted by name. An image lists at least one program. |
| `programs[].name` | string | The name of the program, for example `docker-ce`. Each name appears once. |
| `programs[].version` | string | The version that the build installed. |
| `programs[].source` | string | Where the build got the program, for example `go.dev/dl`. |

A reader refuses these cases:

| Case | Error |
| --- | --- |
| A `format_version` other than `1` | `ErrUnsupportedVersion` |
| A field that the reader does not know, for example a `memory` field in an image | `ErrInvalid` |
| A field name in another case, for example `Kind` | `ErrInvalid` |
| A field name that appears twice in one object | `ErrInvalid` |
| A value that breaks a rule above | `ErrInvalid` |
| Data after the JSON object | `ErrInvalid` |

A new field therefore needs a new format version. A new version can add the kinds `snapshot` and `checkpoint` with their memory files.

Example, shortened:

```json
{
  "format_version": 1,
  "kind": "image",
  "arch": "x86_64",
  "name": "default",
  "created_at": "2026-10-16T00:00:00Z",
  "disk": {
    "file": "disk.raw",
    "format": "raw",
    "filesystem": "ext4",
    "size": 3758096384,
    "sha256": "465579a3e137d8222d1bfde177cca61fb344791812203d628aa09777a1e26162",
    "compressed": {
      "file": "disk.raw.zst",
      "codec": "zstd",
      "size": 558547703,
      "sha256": "03ebe2e931f5521a445822065eb6b7a5ebcc5eb857542eec8d5288c609b07563"
    }
  },
  "kernel": {
    "sha256": "dfb03703bab6d4b590964f2d9473fdc82994ae94384cc5ed1c1b4a0430118a91"
  },
  "programs": [
    {
      "name": "containerd.io",
      "version": "2.4.2-2~ubuntu.26.04~resolute",
      "source": "download.docker.com/linux/ubuntu"
    }
  ]
}
```

## Digest

The digest of an artifact is `sha256:` and the sha256 of the bytes of its `artifact.json`, for example `sha256:354fb767...`. The metadata holds the sha256 of each file, so the digest covers the whole artifact. A node that has the digest from the control plane checks `artifact.json` against it, then checks each file against `artifact.json`. An image cache with the compressed copy checks it with `VerifyCompressed`, unpacks it to the disk file, and runs `VerifyDisk`. After that check, the cache can delete the compressed copy. `VerifyDisk` alone then proves the disk. A node that downloads the compressed disk from the URL of an image gets no `artifact.json`. It checks the unpacked disk against the sha256 and the size that the control plane sends with the image ([the Firecracker driver](/reference/firecracker-driver/index.html.md#the-image-cache)).

The writer encodes the metadata the same way each time: the field order above, two spaces of indent, `created_at` in UTC, and one newline at the end. So two builds with the same content have the same digest.

## Go API

| Function | Description |
| --- | --- |
| `Parse(data)` | Decodes and checks `artifact.json`. |
| `Metadata.Validate()` | Checks the rules above. |
| `Metadata.Marshal()` | Checks the metadata, then encodes it as `artifact.json`. |
| `Digest(data)` | Returns the digest of the bytes of an `artifact.json`. |
| `Read(dir)` | Reads and checks `dir/artifact.json`. Returns the metadata and the digest. It does not read the disk. |
| `Write(dir, metadata)` | Writes `dir/artifact.json` in one rename. Returns the digest. |
| `HashFile(ctx, path)` | Returns the size and the sha256 of a file. |
| `Metadata.VerifyDisk(ctx, dir)` | Checks the size and the sha256 of the disk file. |
| `Metadata.VerifyCompressed(ctx, dir)` | If the metadata names a compressed copy, checks its size and its sha256. |
| `Verify(ctx, dir)` | `Read`, then `VerifyDisk` and `VerifyCompressed`. |

If a file does not match its metadata, the error is a `*MismatchError`. It holds the file name, the field (`size` or `sha256`), and both values.

## imageartifact

The build tool `tools/imageartifact` writes and checks the `artifact.json` of an image build. `make image` and `make check-image` run it ([Default image](/reference/default-image/index.html.md#make-targets)).

```sh
go run ./tools/imageartifact write -dir DIR -name NAME -programs FILE -kernel-sha256 HEX -created UNIX
go run ./tools/imageartifact verify -dir DIR
```

| Command | What it does |
| --- | --- |
| `write` | Hashes `DIR/disk.raw` and `DIR/disk.raw.zst`, and writes `DIR/artifact.json` with the programs of `FILE`, a JSON list of `{name, version, source}`. Prints the digest. |
| `verify` | Runs `Verify` on `DIR`. Prints the digest. |

| Flag | Description |
| --- | --- |
| `-dir` | The artifact folder. |
| `-name` | The name of the artifact. |
| `-programs` | The program manifest. |
| `-kernel-sha256` | The sha256 of the guest kernel. |
| `-created` | The build time in Unix seconds. |

The exit status is 0 on success, 1 on an error, and 2 for a bad command line.
