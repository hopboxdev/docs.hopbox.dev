# API definitions

> The API definitions under proto/, the code generators and their versions, the generated outputs, the make targets, and the API checks in CI.

The API definitions of hopbox are protobuf files under `proto/`. buf checks them and runs the code generators that turn them into Go code, TypeScript code, MCP tools, and OpenAPI documents. This page lists the files, the generators, the outputs, the make targets, and the checks in CI. After a change to a proto file, run `make generate` and commit the generated code with the change (the build guide of the repo has an example).

## Packages and folders

The STANDARD lint rules require that the folder of a file matches its package.

| Package | Folder | Holds |
| --- | --- | --- |
| `hopbox.v1` | `proto/hopbox/v1` | The public API ([the API server](/reference/api/index.html.md)). |
| `hopbox.admin.v1` | `proto/hopbox/admin/v1` | The admin API ([the admin API](/reference/admin-api/index.html.md)). |
| `hopbox.cluster.v1` | `proto/hopbox/cluster/v1` | The cluster API between the servers of an install and the control role: the enrollment and the control channel. Only the internal listener serves it ([the internal listener](/reference/internal-listener/index.html.md)). It is not part of the public API. The name is not `internal`, because Go forbids an import of `gen/hopbox/internal` from `internal/control`. |
| `hopbox.guest.v1` | `proto/hopbox/guest/v1` | The guest link between a node and the guest agent of a machine. [Guest link](/reference/guest-link/index.html.md) describes it. It is not part of the public API. |

| File | Services |
| --- | --- |
| `proto/hopbox/v1/options.proto` | none. It defines the options `(hopbox.v1.idempotent)` and `(hopbox.v1.read_only)`. |
| `proto/hopbox/v1/install.proto` | `hopbox.v1.InstallService` |
| `proto/hopbox/admin/v1/install_config.proto` | `hopbox.admin.v1.InstallConfigService` |
| `proto/hopbox/admin/v1/keyring.proto` | `hopbox.admin.v1.KeyringService` |
| `proto/hopbox/admin/v1/server.proto` | `hopbox.admin.v1.ServerService` |
| `proto/hopbox/cluster/v1/enrollment.proto` | `hopbox.cluster.v1.EnrollmentService` ([Enrollment and the install PKI](/reference/enrollment/index.html.md)) |
| `proto/hopbox/cluster/v1/control.proto` | `hopbox.cluster.v1.ControlService` ([the control channel](/reference/control-channel/index.html.md)) |
| `proto/hopbox/guest/v1/link.proto` | `hopbox.guest.v1.LinkService` |

## Definitions

### `(hopbox.v1.idempotent)`

File: `proto/hopbox/v1/options.proto`.

| Item | Value |
| --- | --- |
| Kind | extension of `google.protobuf.MethodOptions` |
| Type | `bool` |
| Default | `false` |
| Field number | `51200` |

A method with this option creates a resource, such as a machine or a fork. A client must send an `Idempotency-Key` header with each call of such a method. A client that retries the call sends the same key again. The server keeps each key for 24 hours. For a key that it knows, the server returns the first result and does not act again. [The API server](/reference/api/index.html.md#idempotency-keys) says how the server handles the header. `hopbox.admin.v1.KeyringService.RotateKEK` has the option.

Example:

```proto
import "hopbox/v1/options.proto";

service MachineService {
  rpc CreateMachine(CreateMachineRequest) returns (CreateMachineResponse) {
    option (hopbox.v1.idempotent) = true;
  }
}
```

### `(hopbox.v1.read_only)`

File: `proto/hopbox/v1/options.proto`.

| Item | Value |
| --- | --- |
| Kind | extension of `google.protobuf.MethodOptions` |
| Type | `bool` |
| Default | `false` |
| Field number | `51201` |

A method with this option changes nothing on the server, such as a get, a list, or a watch call. A second run of the call does no harm. The request gate still records the last use of the credential of the call, such as an API key, at most once per minute, except in maintenance mode ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#lifetimes)). A second run only moves that time. So the Go client and the TypeScript SDK retry a unary call with it, also after a lost answer ([Go client](/reference/go-client/index.html.md#retries)). They never retry a stream. In maintenance mode, the server lets a call with this option in, and refuses it at its first write, such as its audit entry, before the write ([the API server](/reference/api/index.html.md#maintenance-mode)). So a read that writes nothing works in maintenance mode. The option itself does not say whether a call writes an audit entry: an audit entry of a read, such as the entry of each call of the admin API, does not count as a change for a retry. Give the option to each new call that only reads. Do not give it to a call that writes, also when its name starts with `Get`. For example, `GetDeviceAuthorization` counts each wrong code against a limit, so it has no option.

The standard option `idempotency_level` with the value `NO_SIDE_EFFECTS` says the same. But `buf breaking` with the FILE rules counts a change of `idempotency_level` as a break, so the read calls of v0.1.0 cannot get it. A new custom option passes `buf breaking`. The clients treat both options the same. `GetInstallInfo` has both.

These methods of `hopbox.v1` have the option:

| Service | Methods |
| --- | --- |
| `InstallService` | `GetInstallInfo` |
| `AccountService` | `GetAccount`, `ListSignInMethods`, `ListCLISignIns` |
| `APIKeyService` | `ListAPIKeys` |
| `AuditService` | `ListAuditEntries` |
| `MachineService` | `GetMachine`, `ListMachines`, `ListMachineSizes`, and the streams `WatchMachines`, `Download`, and `StreamMachineLogs` |
| `OrgService` | `GetOrg`, `ListOrgs`, `GetOrgUsage`, `ListOrgMembers`, `ListInvitations` |
| `SSHKeyService` | `ListSSHKeys` |
| `VarService` | `ListVars` |

These methods of `hopbox.admin.v1` have the option:

| Service | Methods |
| --- | --- |
| `InstallConfigService` | `GetInstallConfig`, `GetSMTPConfig`, `GetOAuthConfig` |
| `CertificateService` | `GetCertificate` |
| `KeyringService` | `ListKEKs` |
| `MailService` | `ListMailFiles`, `GetMailFile` |
| `InstanceAdminService` | `ListInstanceAdmins` |
| `AccountService` | `ListAccountSignInMethods` |
| `ServerService` | `ListServers`, `GetServer` |
| `OrgService` | `ListOrgs`, `GetDefaultOrgLimits` |
| `AdminKeyService` | `ListAdminKeys` |
| `MachineService` | `ListMachines` |
| `ImageService` | `ListImages` |
| `PoolService` | `GetPool`, `ListPools` |
| `SizeService` | `GetSize`, `ListSizes` |

No method of `hopbox.cluster.v1` has the option.

Example:

```proto
import "hopbox/v1/options.proto";

service OrgService {
  rpc ListOrgs(ListOrgsRequest) returns (ListOrgsResponse) {
    option (hopbox.v1.read_only) = true;
  }
}
```

## Configuration files

| File | What it sets |
| --- | --- |
| `buf.yaml` | buf configuration version `v2`. One module at `proto/`. Lint uses the `STANDARD` rules. Breaking-change checks use the `FILE` rules. The module has no dependencies, so buf never reads a registry. |
| `buf.gen.yaml` | The plugins, their options, and their output folders. Every plugin is local. `exclude_types` keeps `hopbox.admin.v1`, `hopbox.cluster.v1`, and `hopbox.guest.v1` out of the TypeScript SDK. The SDKs never include the admin API, only the servers of an install and the control role use the cluster API, and only a node and a guest agent use the guest link. `clean: true` deletes each output folder before buf writes to it. Managed mode sets `go_package` to the path of the file under `gen/` in the Go module of hopbox, so the proto files carry no Go options. |
| `proto/hopbox/v1/openapi.base.yaml` | The `info` block of the OpenAPI document of `hopbox.v1`: its title, description, and version. |
| `proto/hopbox/admin/v1/openapi.base.yaml` | The `info` block of the OpenAPI document of `hopbox.admin.v1`. |
| `tools/proto/go.mod` | The versions of buf and of the Go plugins, as `tool` directives. |
| `web/packages/sdk/package.json` | The versions of `@bufbuild/protoc-gen-es` and its runtime `@bufbuild/protobuf`, and the other packages of the TypeScript SDK ([TypeScript SDK](/reference/ts-sdk/index.html.md#package)). `pnpm-lock.yaml` locks them. |

## Generators and outputs

`make generate` runs `buf generate`, which runs these plugins.

| Plugin | Version | Runs on | Output |
| --- | --- | --- | --- |
| `protoc-gen-go` | `google.golang.org/protobuf` v1.36.12 | every package | `gen/<path>/*.pb.go`, Go package `<name><version>`, for example `hopboxv1` |
| `protoc-gen-connect-go` | `connectrpc.com/connect` v1.21.1 | every package | `gen/<path>/<name><version>connect/*.connect.go`, for each file with a service |
| `protoc-gen-go-mcp` | `github.com/redpanda-data/protoc-gen-go-mcp` v0.0.0-20260729122341-6334690a62b6 | `hopbox.v1` | `gen/hopbox/v1/hopboxv1mcp/*.pb.mcp.go`, one MCP tool for each unary method, for each file with a service. The plugin skips streaming methods. The MCP server in `internal/mcp` serves these tools ([the MCP server](/reference/mcp/index.html.md)). |
| `protoc-gen-connect-openapi` | `github.com/sudorandom/protoc-gen-connect-openapi` v0.28.0 | `hopbox.v1` and `hopbox.admin.v1`, one run each | `docs/reference/api/hopbox.v1.openapi.yaml` and `docs/reference/api/hopbox.admin.v1.openapi.yaml`, one OpenAPI 3.1 document for each package |
| `protoc-gen-es` | `@bufbuild/protoc-gen-es` 2.16.0 | every package except `hopbox.admin.v1`, `hopbox.cluster.v1`, and `hopbox.guest.v1` | `web/packages/sdk/src/gen/<path>/*_pb.ts`, TypeScript with `target=ts` and `import_extension=js`, so a relative import between generated files ends in `.js`, as Node.js ES modules require |
| `protoc-gen-es` | `@bufbuild/protoc-gen-es` 2.16.0 | `hopbox.admin.v1`, with the types of `hopbox.v1` that it uses | `web/apps/console/src/gen/<path>/*_pb.ts`, with the same options. The Admin area of the console calls the admin API with this code ([Console](/reference/console/index.html.md#admin-area)). The SDK never includes the admin API. |

buf is `github.com/bufbuild/buf` v1.74.0.

The output folders are in git. Do not edit a file in them by hand. The next `make generate` writes over it, and CI rejects it. `.gitattributes` marks the folders as generated, so GitHub folds them in a diff.

[Public API](/api/hopbox.v1.openapi.yaml) is the generated reference of the public API, and [Admin API](/api/hopbox.admin.v1.openapi.yaml) is the generated reference of the admin API. The cluster API `hopbox.cluster.v1` and the guest link get no OpenAPI document and no MCP tools. [Enrollment and the install PKI](/reference/enrollment/index.html.md) and [the control channel](/reference/control-channel/index.html.md) describe the cluster API. Their paths come from the services, and their descriptions come from the comments in the proto files.

### Version rules

- A generator and the runtime that builds its output must have the same version. The test `TestGoGeneratorsMatchRuntimes` in `proto/generators_test.go` compares `google.golang.org/protobuf`, `connectrpc.com/connect`, and `github.com/redpanda-data/protoc-gen-go-mcp` in `go.mod` with the same modules in `tools/proto/go.mod`. It checks only the modules that `go.mod` requires. The test `TestESGeneratorMatchesRuntime` compares `@bufbuild/protobuf` with `@bufbuild/protoc-gen-es` in `web/packages/sdk/package.json`. It also rejects a range such as `^2.16.0` or `~2.16.0`, so each must be an exact version. `make test` runs both tests.
- To change a version, change it in both files, then run `make generate` and commit the result.
- `protoc-gen-go-mcp` has no release tags, so its version is a pseudo-version of one commit.
- `buf.gen.yaml` runs `protoc-gen-go-mcp` with `GOEXPERIMENT=nojsonv2`. The plugin writes each tool schema with Go's `%#v` format. With the JSON v2 package of Go 1.27, a `json.RawMessage` prints as `jsontext.Value`, and the generated file does not import `jsontext`. With `nojsonv2`, the plugin prints `json.RawMessage`, and the output compiles.
- The generated MCP code imports `google.golang.org/grpc`. grpc uses `http2.TrailerPrefix` from `golang.org/x/net`. With Go 1.27, `golang.org/x/net` v0.54.0 does not define it, and the build fails with `undefined: http2.TrailerPrefix`. `golang.org/x/net` v0.55.0 defines it. So `go.mod` requires `google.golang.org/grpc` v1.83.1, which requires `golang.org/x/net` v0.55.0. The `go.mod` of `github.com/redpanda-data/protoc-gen-go-mcp` asks only for grpc v1.82.1. Do not move grpc below v1.83.1.

## Make targets

| Target | What it does |
| --- | --- |
| `generate` | Runs `buf generate`, then sqlc for the store queries ([the bundled Postgres](/reference/postgres/index.html.md#queries)). If `web/packages/sdk/node_modules/.bin/protoc-gen-es` is missing, it fails with a message. Run `corepack pnpm install` first. |
| `check-generate` | Runs `make generate`. It then looks for a changed, added, or deleted file in `gen/`, `docs/reference/api/`, `web/packages/sdk/src/gen/`, `web/apps/console/src/gen/`, and `internal/store/db/`. If git sees one, it prints the changed files and fails. |
| `lint` | Runs golangci-lint, then `buf lint`, then the docs check, which makes sure that the docs and the code agree. |
| `breaking` | Runs `buf breaking --against '.git#ref=<BREAKING_AGAINST>'`. If that ref has no `proto/` folder, it prints a message and passes. If the ref does not exist, it fails. |

| Variable | Default | Description |
| --- | --- | --- |
| `BUF` | `$(GO) tool -modfile=tools/proto/go.mod buf` | The buf command. |
| `BREAKING_AGAINST` | `origin/main` | The git ref that `make breaking` compares `proto/` with. Run `git fetch` first, so that the ref is current. In a checkout without the `origin` remote, set it to `main`. |

The `breaking` target passes for a ref without `proto/`, so the first change that adds `proto/` can merge. After that change is on `main`, every later change is checked against it.

## Checks in CI

| Check | Job | Command | Fails when |
| --- | --- | --- | --- |
| Lint | `lint` | `make lint` | a definition breaks a `STANDARD` rule, for example a message name that is not PascalCase |
| Breaking changes | `api` | `make breaking BREAKING_AGAINST=origin/main` | a change breaks a `FILE` rule against `main`, for example a deleted extension or a changed field type |
| Drift | `api` | `make check-generate` | the generated code in git differs from the output of `make generate` |
| Versions | `test` | `make test` | a generator and its runtime have different versions |
| Tidy | `lint` | `go mod tidy -diff` in `tools/proto` | `tools/proto/go.mod` or `tools/proto/go.sum` is not tidy |

These outputs illustrate each failure. The `make lint` sample comes from a message that was added to `options.proto` only for the illustration.

```text
$ make lint
proto/hopbox/v1/options.proto:18:9:Message name "install_info" should be PascalCase, such as "InstallInfo".
proto/hopbox/v1/options.proto:19:10:Field name "Domain" should be lower_snake_case, such as "domain".

$ make breaking
proto/hopbox/v1/options.proto:15:3:Extension "51200" with name "hopbox.v1.idempotent" on message "google.protobuf.MethodOptions" changed type from "bool" to "string".

$ make check-generate
 M gen/hopbox/v1/options.pb.go
make: the generated code does not match the API definitions or the SQL queries. Run make generate and commit the result.
```
