# The API server

> The API server: its listener, the request gate, callers and the policy matrix, the interceptors, idempotency keys, error codes, reflection, health, and InstallService.

The control role serves the API of hopbox with Connect. This page covers the listener, the protocols, the request gate, and the callers and the policy check. It also covers the chain of interceptors, idempotency keys, the error codes, reflection, health, and `InstallService`. [The sign-in, account, and audit API](/reference/auth-api/index.html.md) covers the sign-in, account, and audit calls. [The org API](/reference/org-api/index.html.md) covers the org, member, and invitation calls. [The machine API](/reference/machine-api/index.html.md) covers the machine calls. [The admin API](/reference/admin-api/index.html.md) covers the admin API and the admin socket. [The internal listener](/reference/internal-listener/index.html.md) covers the internal listener, which serves the cluster API to the servers of the install over TLS. The chain of a call, the gate, the policy, and the errors of this page apply to it too. The code is in `internal/api` and `internal/service`.

## Listener

| Item | Value |
| --- | --- |
| Address | `control.api_listen` of the bootstrap config, by default `127.0.0.1:7080` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)) |
| Transport | TCP without TLS. The address must be a loopback address. |
| HTTP versions | HTTP/1.1, and HTTP/2 with prior knowledge (h2c) |
| Protocols | Connect, gRPC, and gRPC-Web, with the binary protobuf and JSON codecs, and gzip |
| Services | Every service of `hopbox.v1`: `InstallService`, `AuthService`, `AccountService`, `AuditService`, `OrgService`, `APIKeyService`, `SSHKeyService`, `MachineService`, and `VarService` ([the env var API](/reference/vars-api/index.html.md)). The services of the admin API, with `hopbox.admin.v1.MachineService` and `hopbox.admin.v1.ImageService` ([the admin API](/reference/admin-api/index.html.md#where-the-admin-api-is-served)). gRPC reflection and gRPC health. |
| Browser endpoints | `GET /auth/github/start`, `GET /auth/github/callback`, `GET /auth/google/start`, and `GET /auth/google/callback`: plain HTTP for the sign-in with GitHub and Google, outside of the chain of a call ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#endpoints)). `GET /auth/machine`: plain HTTP for the sign-in of a private machine URL ([the edge role](/reference/edge/index.html.md#private-machine-urls)). |
| Console | every other `GET` or `HEAD` request whose first path segment has no dot, outside of the chain of a call ([Console](/reference/console/index.html.md#where-hopboxd-serves-it)). Any other request with an unknown path gets `404`. |

gRPC needs HTTP/2. Without TLS, a client must start HTTP/2 with prior knowledge, for example `curl --http2-prior-knowledge` or `grpcurl -plaintext`. The Connect protocol also works over HTTP/1.1.

The clients of the install do not reach this listener. They reach the edge servers, which forward the requests of the domain of the install to the internal listener of the control role ([the edge role](/reference/edge/index.html.md#forwarded-requests)). The internal listener serves these requests with the same handler as this listener, with the address of each client as the peer ([the internal listener](/reference/internal-listener/index.html.md#forwarded-requests)). This listener stays on loopback for the processes and the SSH tunnels of the server. Its peers are loopback addresses, so the rate limits of a client skip them ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)).

The listener serves the admin API too, for admin API keys and for the Admin area of the console. The edge servers forward it too, so the admin API is public behind the edge. On this listener, the policy refuses every admin call of every other caller ([Callers and policy](#callers-and-policy)). The Admin area calls a part of the admin API with an admin session: the session of an instance admin with a sign-in within the last 10 minutes ([the admin API](/reference/admin-api/index.html.md#admin-sessions)). Root on the server calls the admin API through the admin socket ([the admin API](/reference/admin-api/index.html.md#the-admin-socket)).

| HTTP server setting | Value |
| --- | --- |
| Header read timeout | 10 s |
| Body read timeout of a unary call | 30 s from the start of the request, until the whole body arrives. A request without a body, such as a Connect `GET`, has no such bound. A stream has none, and neither has a connection that a handler takes over, such as a WebSocket. A read that runs out of time fails with the code `deadline_exceeded` and the message `the request body did not arrive in time`. |
| Idle connection timeout | 2 min |
| Largest request message | 4 MiB. A larger message gets `resource_exhausted` before any handler reads it. |
| HTTP/2 streams of one connection | at most 100 at once (`api.MaxClientStreams`) |
| HTTP/2 receive window | 256 KiB for each stream, and 25 MiB for the connection. The window of the connection holds every stream at once. So a stream whose reads stop, such as an upload to a slow node, never stalls the other calls of the connection. |

A path that names no service gets HTTP status 404.

The listener also serves the WebSocket endpoint `/api/ws/exec` of exec over HTTP/1.1 ([the machine API](/reference/machine-api/index.html.md#the-websocket-endpoint)). It also serves the MCP server of hopbox at `/mcp`, for AI agents with a CLI token or an API key ([the MCP server](/reference/mcp/index.html.md)). Each tool of the MCP server calls the API in the process, through the chain of a call below.

## The chain of a call

Every call passes these steps, in this order:

1. **Request ID.** The server makes a new UUIDv7 for each HTTP request. It puts the ID in the response header `Hopbox-Request-Id`, in the logs of the call, and in the audit entries of the call. The server never takes an ID from the client, so an ID always names one request.
2. **Metrics and access log.** After the call ends, the server counts it in the metrics ([Metrics](/reference/metrics/index.html.md#metrics-of-the-api)) and writes one log record ([Logs](#logs)).
3. **Error mapping.** The server turns each error into a Connect error with a code ([Errors](#errors)).
4. **Default deadline.** A unary call whose client sent no deadline gets a deadline of 30 s. A lifecycle call of a machine gets 61 s instead (`api.LifecycleTimeout`), so its wait of up to 60 s fits ([the machine API](/reference/machine-api/index.html.md#the-wait)). These calls are `StartMachine`, `StopMachine`, `RebootMachine`, `RenameMachine`, and `DeleteMachine` of `hopbox.v1.MachineService`, and `StopMachine` and `DeleteMachine` of `hopbox.admin.v1.MachineService`. A deadline from the client stays as it is, longer or shorter. The client sends a deadline with the header `Connect-Timeout-Ms` or `grpc-timeout`. A stream gets no default deadline.
5. **Maintenance mode.** In maintenance mode, the server refuses each call that can change the install ([Maintenance mode](#maintenance-mode)).
6. **Request gate.** The gate names the caller of the call ([Request gate](#request-gate)).
7. **Idempotency.** A method with the option `(hopbox.v1.idempotent)` runs once for each idempotency key ([Idempotency keys](#idempotency-keys)).
8. **Handler.** The handler calls one operation of the service layer. The operation checks the policy first, then its input, then acts.

Reflection and health pass the same steps, except the idempotency step. In place of a service operation, they run the policy check with their own actions ([Callers and policy](#callers-and-policy)).

## Request gate

The request gate is the one place where every call proves who calls. It holds a chain of credential resolvers. Each resolver knows one kind of credential. The gate asks each resolver in turn:

1. A resolver that does not find its kind of credential passes the call to the next resolver.
2. A resolver that finds its kind of credential, and that credential proves a caller, decides. The call goes on with that caller.
3. A resolver that finds its kind of credential, but the credential proves no caller, ends the call with `unauthenticated` and the message `the credential is not valid`. Examples are an unknown, expired, revoked, or malformed key.
4. Sometimes a resolver cannot check the credential, for example because the control store does not answer. It then ends the call with `unavailable` and the message `the server failed to check the credential. Retry later`. The SDKs retry this call, because its answer carries `Hopbox-Not-Run: 1` ([Answers of calls that changed nothing](#answers-of-calls-that-changed-nothing)).
5. If no resolver decides and the call has an `Authorization` header, the gate ends the call with `unauthenticated` and the message `the credential is not valid`. A credential that the server cannot check never passes as anonymous.
6. If no resolver decides and the call has no `Authorization` header, the caller is anonymous.

The client always gets the fixed messages above. The cause goes only to the log, with the request ID ([Logs](#logs)).

The gate only names the caller. The policy check in the service layer decides what the caller can do.

| Listener | Resolvers, in order |
| --- | --- |
| API listener | the session cookie `__Host-hopbox_session` ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)), then CLI access tokens, API keys, and admin API keys in the header `Authorization: Bearer <credential>` ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#how-the-request-gate-reads-them)) |
| Admin socket | the peer of the socket, which proves the caller `local_root` ([the admin API](/reference/admin-api/index.html.md#the-admin-socket)) |
| Internal listener | the server certificate of the TLS connection, which proves the caller `server` ([the internal listener](/reference/internal-listener/index.html.md)) |

On the internal listener, a step after the gate refuses an anonymous caller on every procedure but `/hopbox.cluster.v1.EnrollmentService/Enroll`, before the policy check. The call gets `unauthenticated` with the message `this call needs a server certificate`. This covers gRPC health and reflection there too. On the other listeners, an anonymous caller reaches every procedure, and the policy decides.

### The contract of a resolver

The API listener has a resolver for each credential that hopbox issues: sessions, CLI access tokens, API keys, and admin API keys. Each of the last three reads only the bearer credentials of its own prefix ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#forms-and-prefixes)). The resolver of server certificates exists on the internal listener. It reads the certificate of the TLS connection from the context, and ties it to a server of the registry. It refuses an unknown, expired, foreign, or decommissioned certificate with `the credential is not valid` ([the internal listener](/reference/internal-listener/index.html.md)). A resolver is a Go value with the method `Resolve(ctx, credentials) (caller, ok, err)` in `internal/api`. It gets the request headers and the peer, and must keep these rules:

| Case | It returns |
| --- | --- |
| The request does not carry its kind of credential. | `ok` false, no error |
| The credential proves a caller. | the caller, `ok` true, no error |
| The credential is of its kind but proves no caller: unknown, expired, revoked, or malformed. | an error that wraps `api.ErrCredentialRefused`. The resolver of the session cookie is the one exception, below. |
| It cannot check the credential, for example because the control store does not answer. | any other error |

An error of a resolver goes to the log, so it must never hold a secret, such as the credential itself.

A browser sends a cookie on its own, also long after its session ended. So the resolver of the session cookie treats a cookie that names no live session as no credential. The gate then goes on to the next resolver. The resolver also adds the clear of the cookie to `ResponseHeader` of the credentials. The response carries the clear, unless the handler sets the cookie itself ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)). A stale cookie thus never blocks the next sign-in. A live session from a call that fails the check against cross-site requests is refused, as the contract says.

## Callers and policy

Each call has one typed caller. Its kind is also the actor kind of its audit entries ([Control store tables](/reference/control-store/index.html.md#audit_log)).

| Kind | Caller | Proved by |
| --- | --- | --- |
| `anonymous` | A caller without a credential. | no credential |
| `session` | A user signed in to the console. The caller also says whether the account is an instance admin, and whether it is pending. | the session cookie ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md)) |
| `cli_token` | A user through the hopbox CLI. It acts as its account in every org of the account. The caller also says whether the account is an instance admin. | a CLI access token ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#cli-tokens)) |
| `member_key` | A member API key, which acts as its account, only in the org of the key. | an API key ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)) |
| `org_key` | An org API key: a service credential of one org, with the rights of a member without personal resources. | an API key ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#api-keys)) |
| `admin_key` | An admin API key, with install scope. | an admin API key ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#admin-api-keys)) |
| `local_root` | Root, or the user of `hopboxd`, on a control server. | the peer of the admin socket |
| `server` | A server of the install, for itself. The caller carries the roles of its certificate, such as `node` or `dns`. | a server certificate on the internal listener ([the internal listener](/reference/internal-listener/index.html.md)) |
| `control` | Another control process of the install, for forwarding a data stream. It has no account and no credential ID. It may do only `control.forward_stream`, besides the actions that every caller may do. | the control certificate of its internal listener, which names `hopbox://control` ([the internal listener](/reference/internal-listener/index.html.md#callers)) |
| `ssh_key` | An account through the SSH front door. No request of the API carries it. The service layer makes it only to check an SSH connection, for `machine.ssh` ([the SSH front door](/reference/ssh-front-door/index.html.md#who-opens-a-machine)). It is the actor of the audit entry `machine.ssh` ([the SSH front door](/reference/ssh-front-door/index.html.md#audit)). | a signature with an SSH key of the account, which the edge server checked |

Every operation of the service layer calls one policy check, `policy.Authorize(caller, action, resource)`, before it reads or changes anything. An action without a rule is refused for every caller. This matrix lists every action and the callers that it allows. Every other caller is refused.

| Action | Operation | Resource | Allowed callers |
| --- | --- | --- | --- |
| `install.get_info` | `InstallService.GetInstallInfo` | the install | every caller, also `anonymous` |
| `install_config.get` | `InstallConfigService.GetInstallConfig` | the install | `local_root`, `admin_key`, an admin `session` |
| `install_config.update` | `InstallConfigService.UpdateInstallConfig` | the install | `local_root`, `admin_key`, an admin `session` |
| `certificate.get` | `CertificateService.GetCertificate` | the install | `local_root`, `admin_key` |
| `smtp_config.get` | `InstallConfigService.GetSMTPConfig` | the install | `local_root`, `admin_key`, an admin `session` |
| `smtp_config.set` | `InstallConfigService.SetSMTPConfig` | the install | `local_root`, `admin_key`, an admin `session` |
| `smtp_config.clear` | `InstallConfigService.ClearSMTPConfig` | the install | `local_root`, `admin_key`, an admin `session` |
| `oauth_config.get` | `InstallConfigService.GetOAuthConfig` | the install | `local_root`, `admin_key`, an admin `session` |
| `oauth_config.set` | `InstallConfigService.SetOAuthClient` | the install | `local_root`, `admin_key`, an admin `session` |
| `oauth_config.clear` | `InstallConfigService.ClearOAuthClient` | the install | `local_root`, `admin_key`, an admin `session` |
| `keyring.rotate` | `KeyringService.RotateKEK` | the install | `local_root` only |
| `keyring.list` | `KeyringService.ListKEKs` | the install | `local_root` only |
| `keyring.rewrap` | `KeyringService.RewrapSealedValues` | the install | `local_root` only |
| `keyring.stage` | `KeyringService.StageKEK` | the install | `local_root` only |
| `keyring.promote` | `KeyringService.PromoteKEK` | the install | `local_root` only |
| `mail_file.list` | `MailService.ListMailFiles` | the install | `local_root` only |
| `mail_file.get` | `MailService.GetMailFile` | the install | `local_root` only |
| `login_link.create` | `LoginLinkService.CreateLoginLink` | the install | `local_root` only |
| `instance_admin.list` | `InstanceAdminService.ListInstanceAdmins` | the install | `local_root`, `admin_key`, an admin `session` |
| `instance_admin.grant` | `InstanceAdminService.GrantInstanceAdmin` | the install | `local_root`, `admin_key`, an admin `session` |
| `instance_admin.revoke` | `InstanceAdminService.RevokeInstanceAdmin` | the install | `local_root`, `admin_key`, an admin `session` |
| `admin_account.list_sign_in_methods` | admin `AccountService.ListAccountSignInMethods` | the install | `local_root`, `admin_key` |
| `admin_account.remove_sign_in_method` | admin `AccountService.RemoveAccountSignInMethod` | the install | `local_root`, `admin_key` |
| `admin_account.delete` | admin `AccountService.DeleteAccount` | the install | `local_root`, `admin_key` |
| `admin_account.list` | admin `AccountService.ListAccounts` | the install | `local_root`, `admin_key`, an admin `session` |
| `admin_account.suspend` | admin `AccountService.SuspendAccount` | the install | `local_root`, `admin_key`, an admin `session` |
| `admin_account.unsuspend` | admin `AccountService.UnsuspendAccount` | the install | `local_root`, `admin_key`, an admin `session` |
| `admin_account.approve` | admin `AccountService.ApproveAccount` | the install | `local_root`, `admin_key`, an admin `session` |
| `admin_account.reject` | admin `AccountService.RejectAccount` | the install | `local_root`, `admin_key`, an admin `session` |
| `sign_up_block.list` | admin `AccountService.ListSignUpBlocks` | the install | `local_root`, `admin_key`, an admin `session` |
| `sign_up_block.add` | admin `AccountService.BlockSignUp` | the install | `local_root`, `admin_key`, an admin `session` |
| `sign_up_block.remove` | admin `AccountService.UnblockSignUp` | the install | `local_root`, `admin_key`, an admin `session` |
| `admin_key.create` | `AdminKeyService.CreateAdminKey` | the install | `local_root`, an admin `session` |
| `admin_key.register` | `AdminKeyService.RegisterAdminKey` | the install | `local_root` only |
| `admin_key.list` | `AdminKeyService.ListAdminKeys` | the install | `local_root`, an admin `session` |
| `admin_key.revoke` | `AdminKeyService.RevokeAdminKey` | the install | `local_root`, an admin `session` |
| `auth.request_magic_link` | `AuthService.RequestMagicLink` | the install | `anonymous`, `session` |
| `auth.verify_magic_link` | `AuthService.VerifyMagicLink` | the install | `anonymous`, `session` |
| `auth.verify_login_link` | `AuthService.VerifyLoginLink` | the install | `anonymous`, `session` |
| `auth.start_oauth_sign_in` | `GET /auth/{provider}/start` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#start)) | the install | `anonymous`, `session` |
| `auth.complete_oauth_sign_in` | `GET /auth/{provider}/callback` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#callback)) | the install | `anonymous`, `session` |
| `auth.sign_out` | `AuthService.SignOut` | the install | `session`, `cli_token` |
| `auth.start_device_authorization` | `AuthService.StartDeviceAuthorization` | the install | every caller, also `anonymous`. The device code is the credential of the flow. |
| `auth.get_device_authorization` | `AuthService.GetDeviceAuthorization` | the install | `session` |
| `auth.approve_device` | `AuthService.ApproveDevice` | the install | `session`, with a sign-in within the last 10 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)) |
| `auth.poll_device_authorization` | `AuthService.PollDeviceAuthorization` | the install | every caller, also `anonymous`. The device code is the credential. |
| `auth.refresh_cli_token` | `AuthService.RefreshCLIToken` | the install | every caller, also `anonymous`. The refresh token is the credential. |
| `account.get` | `AccountService.GetAccount` | the account of the caller | the `session`, `cli_token`, or `member_key` of the account |
| `account.update` | `AccountService.UpdateAccount` | the account of the caller | the `session` or `cli_token` of the account |
| `account.delete` | `AccountService.DeleteAccount` | the account of the caller | the `session` or `cli_token` of the account |
| `account.set_sign_up_note` | `AccountService.SetSignUpNote` | the account of the caller | the `session` or `cli_token` of the account. Only a pending account has a note, and a pending account has no CLI token. |
| `account.list_sign_in_methods` | `AccountService.ListSignInMethods` | the account of the caller | the `session` or `cli_token` of the account |
| `account.remove_sign_in_method` | `AccountService.RemoveSignInMethod` | the account of the caller | the `session` or `cli_token` of the account |
| `account.link_sign_in_method` | `AuthService.RequestMagicLink` with `link`, and `GET /auth/{provider}/start?link=1` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#link-to-a-signed-in-account)) | the account of the caller | the `session` of the account, with a sign-in within the last 10 minutes ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)) |
| `auth.list_cli_sign_ins` | `AccountService.ListCLISignIns` | the account of the caller | the `session` or `cli_token` of the account |
| `auth.end_cli_sign_in` | `AccountService.EndCLISignIn` | the account of the caller | the `session` of the account. A stolen CLI token thus cannot end the other sign-ins of its owner. |
| `ssh_key.add` | `SSHKeyService.AddSSHKey` | the account of the caller | the `session` or `cli_token` of the account |
| `ssh_key.list` | `SSHKeyService.ListSSHKeys` | the account of the caller | the `session` or `cli_token` of the account |
| `ssh_key.remove` | `SSHKeyService.RemoveSSHKey` | the account of the caller | the `session` or `cli_token` of the account |
| `api_key.create` | `APIKeyService.CreateAPIKey` | the new key: its org with the role of the caller, its kind, and its account | a `session` or `cli_token`. A member key for itself needs the role `member` or more. An org key needs `admin` or `owner`. |
| `api_key.list` | `APIKeyService.ListAPIKeys` | the org, with the role of the caller in it | a `session` or `cli_token` with a role in the org. The call returns only the keys that the caller can revoke. |
| `api_key.revoke` | `APIKeyService.RevokeAPIKey` | the key: its org with the role of the caller, its kind, and its account | a `session` or `cli_token`. Its own member key needs the role `member` or more. Any other key needs `admin` or `owner`. |
| `audit.list_org` | `AuditService.ListAuditEntries` for one org | the org, with the role of the caller in it | a `session` or `cli_token` with the role `owner` or `admin` in the org. Also a `member_key` of such an account in the org of the key, a `cli_token` of an instance admin, and an admin `session`. |
| `audit.list_install` | `AuditService.ListAuditEntries` for the whole install | the install | a `cli_token` of an instance admin, or an admin `session` |
| `org.create` | `OrgService.CreateOrg` | the install | `session`, `cli_token` |
| `org.list` | `OrgService.ListOrgs` | the account of the caller | the `session` or `cli_token` of the account |
| `org.get` | `OrgService.GetOrg` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with a role in the org, or an `org_key` of the org |
| `org.get_usage` | `OrgService.GetOrgUsage` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with a role in the org, or an `org_key` of the org |
| `org.member_list` | `OrgService.ListOrgMembers` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with a role in the org, or an `org_key` of the org |
| `org.leave` | `OrgService.LeaveOrg` | the org, with the role of the caller in it | a `session` or `cli_token` with a role in the org |
| `org.invitation_list` | `OrgService.ListInvitations` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with the role `admin` or `owner` |
| `org.invitation_revoke` | `OrgService.RevokeInvitation` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with the role `admin` or `owner` |
| `org.invitation_create` | `OrgService.CreateInvitation` | the org and the role of the invitation | a `session`, `cli_token`, or `member_key` with the role `owner`, or with the role `admin` for the role `member` |
| `org.member_remove` | `OrgService.RemoveOrgMember` | the org and the role of the member | a `session`, `cli_token`, or `member_key` with the role `owner`, or with the role `admin` for a `member` |
| `org.member_set_role` | `OrgService.SetOrgMemberRole` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with the role `owner` |
| `org.update` | `OrgService.UpdateOrg` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with the role `owner` |
| `org.delete` | `OrgService.DeleteOrg` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with the role `owner` |
| `org.invitation_accept` | `OrgService.AcceptInvitation` | the install | `session`, `cli_token`. The service layer then checks the email of the invitation. |
| `var.list` | `VarService.ListVars` | the org, with the role of the caller in it | a `session`, `cli_token`, or `member_key` with a role in the org, or an `org_key` of the org. The call returns the org-level values and the private values of the caller ([the env var API](/reference/vars-api/index.html.md)). |
| `var.set` | `VarService.SetVar` | the value: its org with the role of the caller, its scope, and its owner | a `session`, `cli_token`, or `member_key`. A shared or an all value needs the role `admin` or `owner`. A private value needs the role `member` or more, and the account of the caller as its owner. |
| `var.delete` | `VarService.DeleteVar` | the value: its org with the role of the caller, its scope, and its owner | as `var.set` |
| `org.list_all` | admin `OrgService.ListOrgs` | the install | `local_root`, `admin_key`, an admin `session` |
| `org.suspend` | admin `OrgService.SuspendOrg` | the install | `local_root`, `admin_key`, an admin `session` |
| `org.unsuspend` | admin `OrgService.UnsuspendOrg` | the install | `local_root`, `admin_key`, an admin `session` |
| `org.set_limits` | admin `OrgService.SetOrgLimits` | the install | `local_root`, `admin_key`, an admin `session` |
| `org_limits.get_defaults` | admin `OrgService.GetDefaultOrgLimits` | the install | `local_root`, `admin_key`, an admin `session` |
| `org_limits.set_defaults` | admin `OrgService.SetDefaultOrgLimits` | the install | `local_root`, `admin_key`, an admin `session` |
| `machine.create` | `MachineService.CreateMachine` | the org, with the role of the caller in it | a caller that acts in the org: a `session` or a `cli_token` with a role in the org. Also a `member_key` of the org whose account has a role in it, and an `org_key` of the org. |
| `machine.list` | `MachineService.ListMachines` | the org, with the role of the caller in it | a caller that acts in the org, as for `machine.create`. The list holds only the machines that `machine.get` allows. |
| `machine.watch` | `MachineService.WatchMachines` | the org, with the role of the caller in it | a caller that acts in the org, as for `machine.create`. The stream holds only the machines that `machine.get` allows. |
| `machine.get` | `MachineService.GetMachine` | the machine, with its org, its creator, and whether it is shared | the callers of `machine.start`, and a `session`, a `cli_token`, or a `member_key` with the role `admin` or `owner` in the org |
| `machine.list_sizes` | `MachineService.ListMachineSizes` | the install | `session`, `cli_token`, `member_key`, `org_key` |
| `machine.start` | `MachineService.StartMachine` | the machine | the creator of the machine, as a caller that acts in its org. For a shared machine, every caller that acts in the org. |
| `machine.reboot` | `MachineService.RebootMachine` | the machine | as for `machine.start` |
| `machine.rename` | `MachineService.RenameMachine` | the machine | as for `machine.start` |
| `machine.update` | `MachineService.UpdateMachine` | the machine | as for `machine.start` |
| `machine.stop` | `MachineService.StopMachine` | the machine | as for `machine.get` |
| `machine.delete` | `MachineService.DeleteMachine` | the machine | as for `machine.get` |
| `machine.share` | `MachineService.ShareMachine` | the machine | the creator of the machine, in its org: the account of a `session`, a `cli_token`, or a `member_key`, or the `org_key` that made it |
| `machine.unshare` | `MachineService.UnshareMachine` | the machine | as for `machine.share` |
| `machine.set_url_port` | `MachineService.SetMachineURLPort` | the machine | as for `machine.share` |
| `machine.set_url_visibility` | `MachineService.SetMachineURLVisibility` | the machine | as for `machine.share` |
| `machine.exec` | `MachineService.Exec`, `Upload`, `Download`, `StreamMachineLogs`, and the WebSocket `/api/ws/exec` | the machine | as for `machine.start` |
| `machine.ssh` | an SSH connection through the front door, in `EdgeService.AuthorizeSSH` ([the SSH front door](/reference/ssh-front-door/index.html.md#who-opens-a-machine)) | the machine | an `ssh_key` of an account that uses the machine, as for `machine.exec`. No other caller kind. |
| `machine.open_url` | `GET /auth/machine`, and `EdgeService.AuthorizeMachineRequest` for the account of a token ([the edge role](/reference/edge/index.html.md#private-machine-urls)) | the machine | a `session` of an account that uses the machine, as for `machine.start`. No other caller kind. |
| `machine.list_all` | admin `MachineService.ListMachines` | the install | `local_root`, `admin_key`, an admin `session` |
| `machine.admin_stop` | admin `MachineService.StopMachine` | the install | `local_root`, `admin_key`, an admin `session` |
| `machine.admin_delete` | admin `MachineService.DeleteMachine` | the install | `local_root`, `admin_key`, an admin `session` |
| `machine.move` | admin `MachineService.MoveMachine` ([Cold move](/reference/cold-move/index.html.md)) | the install | `local_root`, `admin_key` |
| `machine.list_moves` | admin `MachineService.ListMoves` | the install | `local_root`, `admin_key` |
| `machine.cancel_move` | admin `MachineService.CancelMove` | the install | `local_root`, `admin_key` |
| `image.register` | admin `ImageService.RegisterImage`, and the start of the control role for the default image of its release ([hopboxd](/reference/hopboxd/index.html.md#the-default-image-of-the-release)) | the install | `local_root`, `admin_key` |
| `image.list` | admin `ImageService.ListImages` | the install | `local_root`, `admin_key`, an admin `session` |
| `pool.create` | `PoolService.CreatePool` | the install | `local_root`, `admin_key`, an admin `session` |
| `pool.get` | `PoolService.GetPool` | the install | `local_root`, `admin_key`, an admin `session` |
| `pool.list` | `PoolService.ListPools` | the install | `local_root`, `admin_key`, an admin `session` |
| `pool.delete` | `PoolService.DeletePool` | the install | `local_root`, `admin_key`, an admin `session` |
| `size.create` | `SizeService.CreateSize` | the install | `local_root`, `admin_key`, an admin `session` |
| `size.get` | `SizeService.GetSize` | the install | `local_root`, `admin_key`, an admin `session` |
| `size.list` | admin `SizeService.ListSizes` | the install | `local_root`, `admin_key`, an admin `session` |
| `size.retire` | `SizeService.RetireSize` | the install | `local_root`, `admin_key`, an admin `session` |
| `health.check` | `grpc.health.v1.Health`, `Check` and `Watch` | the install | every caller, also `anonymous` |
| `reflection.read` | `grpc.reflection.v1.ServerReflection` and `grpc.reflection.v1alpha.ServerReflection` | the install | every caller, also `anonymous` |
| `enrollment_token.create` | `ServerService.CreateEnrollmentToken` | the install | `local_root`, `admin_key` |
| `monitor.get_overview` | `MonitorService.GetOverview` | the install | `local_root`, `admin_key`, an admin `session` |
| `monitor.get_capacity` | `MonitorService.GetCapacity` | the install | `local_root`, `admin_key`, an admin `session` |
| `monitor.list_top_machines` | `MonitorService.ListTopMachines` | the install | `local_root`, `admin_key`, an admin `session` |
| `monitor.get_account_activity` | `MonitorService.GetAccountActivity` | the install | `local_root`, `admin_key`, an admin `session` |
| `server.list` | `ServerService.ListServers` | the install | `local_root`, `admin_key`, an admin `session` |
| `server.get` | `ServerService.GetServer` | the install | `local_root`, `admin_key`, an admin `session` |
| `server.cordon` | `ServerService.CordonServer` | the install | `local_root`, `admin_key`, an admin `session` |
| `server.uncordon` | `ServerService.UncordonServer` | the install | `local_root`, `admin_key`, an admin `session` |
| `server.set_labels` | `ServerService.SetServerLabels` | the install | `local_root`, `admin_key`, an admin `session` |
| `server.set_peer_address` | `ServerService.SetServerPeerAddress` | the install | `local_root`, `admin_key` |
| `server.decommission` | `ServerService.DecommissionServer` | the install | `local_root`, `admin_key` |
| `control.list_processes` | `ServerService.ListControlProcesses` | the install | `local_root`, `admin_key` |
| `server.enroll` | `EnrollmentService.Enroll` | the install | every caller, also `anonymous`. The enrollment token is the credential ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). |
| `server.channel` | `ControlService.Channel` | the server of the caller | `server`, for its own server only |
| `server.heartbeat` | a heartbeat on `ControlService.Channel` | the server of the caller | `server`, for its own server only |
| `server.renew_certificate` | a certificate renewal on `ControlService.Channel` | the server of the caller | `server`, for its own server only |
| `server.attach_stream` | `ControlService.AttachStream` | the server that the stream was asked of | `server`, for its own server only |
| `control.forward_stream` | `ControlPeerService.ForwardStream` ([the control channel](/reference/control-channel/index.html.md#streams-through-another-control-process)) | the install | `control` only |
| `server.read_dns_zones` | a read of the DNS zones before a push on `ControlService.Channel` | the server of the caller | `server`, for its own server only, with a certificate that carries the role `dns` ([the control channel](/reference/control-channel/index.html.md#dns-zones)) |
| `server.read_mesh_peers` | a read of the mesh before a push of `mesh_peers` | the server of the caller | `server`, for its own server only ([the control channel](/reference/control-channel/index.html.md#mesh-pushes)) |
| `server.read_control_urls` | a read of the live control URLs before a push of `control_urls` | the server of the caller | `server`, for its own server only ([the control channel](/reference/control-channel/index.html.md#control-urls)) |
| `server.read_disk_transfers` | a read of the cold moves of a node before a push of `disk_transfers` | the server of the caller | `server` with the role `node`, for its own server only ([the control channel](/reference/control-channel/index.html.md#disk-transfers)) |
| `server.read_machine_specs` | a read of the machine specs before a push of `reconcile_machine` | the server of the caller | `server`, for its own server only |
| `server.report_machine` | a `MachineReport` on `ControlService.Channel` | the server of the caller | `server`, for its own server only |
| `server.read_edge_certificate` | a read of the certificate before a push of `edge_certificate` | the server of the caller | `server`, for its own server only, with a certificate that carries the role `edge` ([the control channel](/reference/control-channel/index.html.md#edge-pushes)) |
| `server.read_edge_routes` | a read of the routes before a push of `edge_routes` | the server of the caller | as for `server.read_edge_certificate` |
| `server.forward_requests` | a request with the header `Hopbox-Client-Addr` on the internal listener ([the internal listener](/reference/internal-listener/index.html.md#forwarded-requests)) | the server of the caller | as for `server.read_edge_certificate` |
| `server.exchange_machine_url_code` | `EdgeService.ExchangeMachineCode` | the server of the caller | as for `server.read_edge_certificate` |
| `server.authorize_machine_url` | `EdgeService.AuthorizeMachineRequest` | the server of the caller | as for `server.read_edge_certificate` |
| `server.read_edge_ssh_keys` | a read of the keys of the SSH front door before a push of `edge_ssh_keys` | the server of the caller | as for `server.read_edge_certificate` |
| `server.authorize_ssh` | `EdgeService.AuthorizeSSH` ([the SSH front door](/reference/ssh-front-door/index.html.md#edgeserviceauthorizessh)) | the server of the caller | as for `server.read_edge_certificate` |

The `server.*` actions of the control channel and of the edge role have a server as their resource. The policy allows them only to a caller of kind `server` whose server is that resource. So the certificate of one server never acts for another ([the control channel](/reference/control-channel/index.html.md)).

Each key-encryption key file lives on one control server, so only root on that server can rotate it, stage it, promote it, list it, or rewrap with it. An admin API key reaches whichever control server answers the call, so it cannot touch a keyring. The first-admin login link is for root on a control server only.

An `org_key` acts as a member without private machines, and is never an org admin. An instance admin has only the rights of its own roles in the `machine.*` rules of the public API. A caller that `machine.get` refuses gets `not_found` instead of `permission_denied`, so it learns nothing about the private machines of others ([the machine API](/reference/machine-api/index.html.md#who-can-do-what)).
For an org, the service layer reads the role of the caller in the org before the check. A caller that is not a member has no role. The check then refuses an org that does not exist in the same way as an org of others. Some changes in an org, such as a role change, lock the org first. They then read the role of the caller again, and run the check again ([the org API](/reference/org-api/index.html.md#every-org-keeps-an-owner)). A member key has the role of its account, and only in the org of the key. An org key has no role. It has the rights of a member in its own org, without personal resources. So it can read its org with `org.get`, `org.get_usage`, and `org.member_list`. It can also list the shared and all values of the org with `var.list`. Only people manage keys: no key makes, lists, or deletes an API key or an admin API key. A call without an `org_id` acts in the org of its API key, or else in the personal org of the caller.

Health and reflection do not call the service layer, so the API runs the policy check for them before their handler.

An admin `session` is the `session` of an instance admin whose sign-in is at most 10 minutes old ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). The Admin area of the console calls the admin API with it. Every other `session` is refused on these actions, and so is the `cli_token` of an instance admin. [The admin API](/reference/admin-api/index.html.md#admin-sessions) lists the calls.

A refused `anonymous` caller gets `unauthenticated` with the fixed message `this call needs a credential`, which names no action. Any other refused caller gets `permission_denied`, and the message names the caller kind and the action. One case is different. Sometimes the check refuses the `session` of an instance admin only because its sign-in is too old. The check then returns the error of a recent sign-in instead ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). That error has the detail `hopbox.v1.RecentSignInRequired`, so the console asks the person to sign in again.

The `session` of a pending account is the other case ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#pending-accounts)). The gate reads the state of the account with the session on each call. The check allows such a session only `account.get`, `account.set_sign_up_note`, `account.delete`, `auth.sign_out`, and the actions that `anonymous` can do. Each other action gets `permission_denied` with the message `permission denied: the account waits for approval by an instance admin`. That error has the detail `hopbox.v1.AccountPending` ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#accountpending)), so the console shows the page of a pending account.

## Idempotency keys

A method with the option `(hopbox.v1.idempotent)` ([API definitions](/reference/api-definitions/index.html.md#definitions)) runs once for each idempotency key. Today these are `KeyringService.RotateKEK`, `ServerService.CreateEnrollmentToken`, `OrgService.CreateOrg`, `AdminKeyService.CreateAdminKey`, `AdminKeyService.RegisterAdminKey`, `APIKeyService.CreateAPIKey`, `SSHKeyService.AddSSHKey`, and `MachineService.CreateMachine`. The responses of `CreateEnrollmentToken`, `CreateAdminKey`, and `CreateAPIKey` hold a secret, so the server keeps no copy of them. A replay of these methods fails with `already_exists` instead of the first response ([the admin API](/reference/admin-api/index.html.md#createenrollmenttoken)).

| Header | Direction | Value |
| --- | --- | --- |
| `Idempotency-Key` | request | 1 to 255 printable ASCII characters without spaces, for example a UUID. Required on an idempotent method. |
| `Idempotent-Replayed` | response | `true` on a response that the server replayed from an earlier call. Absent on a first call. |

The server handles an idempotent call in these steps:

1. An anonymous caller gets `unauthenticated` with the message `an idempotent call needs a credential, so that its key belongs to one caller`.
2. A call without the `Idempotency-Key` header gets `invalid_argument`.
3. The server claims the key for the caller in a transaction ([Control store tables](/reference/control-store/index.html.md#run-an-idempotent-call)). The key covers the method and the request body, encoded as deterministic protobuf.
4. For a new key, the handler runs in the same transaction. Its change, its audit entry, the key, and the response commit together. If the handler fails, nothing commits, and a retry with the same key runs again.
5. For a known key with the same method and body, the server returns the first response with `Idempotent-Replayed: true`. The handler does not run.
6. For a known key with another method or body, the call gets `invalid_argument`.

Each key belongs to one scope, so two callers never share a key:

| Caller | Scope of its keys |
| --- | --- |
| `session`, `cli_token` | `account:<account ID>`. These credentials act as the account, so they share its keys. |
| `member_key` | `member_key:<key ID>`. Each script has its own key, so two scripts never replay each other. |
| `org_key` | `org_key:<key ID>` |
| `admin_key` | `admin_key:<key ID>` |
| `server` | `server:<server ID>`. No method of the cluster API has the idempotent option. |
| `local_root` | `local_root`, one scope for every local call |

The server keeps each key for 24 hours. The leader deletes the expired keys every 10 minutes ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)).

## Errors

Each error reaches the client as a Connect error, with a code and a message. The server maps the errors of the service layer like this. The first match wins.

| Error | Code | HTTP status in the Connect protocol |
| --- | --- | --- |
| A call that needs a credential, from an anonymous caller | `unauthenticated` | 401 |
| An `Authorization` header that no resolver knows, or a credential that a resolver refused | `unauthenticated` | 401 |
| A credential that a resolver failed to check | `unavailable` | 503 |
| A stream whose credential ended while it ran, or a credential that cannot hold a stream ([Streams and their credential](#streams-and-their-credential)). The message starts with `the credential of this stream ended`. | `unauthenticated` | 401 |
| A caller that the policy refuses | `permission_denied` | 403 |
| A call that needs a recent sign-in, from a session whose sign-in is more than 10 minutes old ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#recent-sign-in)). The message starts with `permission denied: this action needs a sign-in within the last 10 minutes`, and the error has the detail `hopbox.v1.RecentSignInRequired`. Other callers of such a call get the result of its policy check, without the detail. | `permission_denied` | 403 |
| A sign-in call that does not come from the console of the install ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#calls-with-the-cookie)) | `permission_denied` | 403 |
| A code or a token of a private machine URL that does not work: malformed, unknown, used, expired, replaced, for another machine, or of a console session that ended ([the internal listener](/reference/internal-listener/index.html.md#edgeservice)). The message starts with `the sign-in of the machine URL is not valid`. This rule comes before the rule of `not_found`. | `unauthenticated` | 401 |
| A sign-in link that is unknown, used, or expired | `unauthenticated` | 401 |
| A device code that is unknown or used, or a refresh token that is unknown, replaced, or expired ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)) | `unauthenticated` | 401 |
| An object that does not exist, such as an unknown sign-in method, server, or pool, or an invitation that is unknown, used, revoked, or expired. The message starts with `not found:`. | `not_found` | 404 |
| A state that does not allow the request yet. Examples are a link without a domain, the last instance admin, and an enrollment token before the first key-encryption key. Others are a cordon of a decommissioned server, the last owner of an org, a suspended org, and an expired device code. A machine action that the states of the machine do not allow is one too, and so is a create without an image. The message starts with `failed precondition:`. | `failed_precondition` | 400 |
| Too many sign-in links for one email or one client ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md#rate-limits)). Too many OAuth starts of one client or of the install ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#rate-limits)). Too many machine URL codes of one console session ([the edge role](/reference/edge/index.html.md#private-machine-urls)). Too many invitations of one org, inviter, or email ([the org API](/reference/org-api/index.html.md#invitations)). Too many device sign-ins or wrong user codes ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#limits-of-the-device-flow)). | `resource_exhausted` | 429 |
| A usage limit, such as the most orgs of one account ([the org API](/reference/org-api/index.html.md#createorg)). Also the most API keys of an org, and the most SSH keys of an account ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). | `resource_exhausted` | 429 |
| An object that another one holds already, such as an SSH key on an account. The message starts with `already exists:`. | `already_exists` | 409 |
| An outside service that failed, such as the SMTP server | `unavailable` | 503 |
| A refused enrollment token: unknown, expired, used, of another CA, or not a token. The message starts with `the enrollment token is not valid` ([Enrollment and the install PKI](/reference/enrollment/index.html.md)). | `unauthenticated` | 401 |
| A server that is decommissioned or unknown, in a call of the control channel. The message is `the server is decommissioned or unknown`. | `unauthenticated` | 401 |
| A field that breaks a rule, for example a bad domain | `invalid_argument` | 400 |
| A missing or bad `Idempotency-Key` header, or a key that another request used | `invalid_argument` | 400 |
| A keyring that refused a rotation, for example a dir that others can read | `failed_precondition` | 400 |
| A row that changed after the caller read it, for example a stale `expected_version` | `aborted` | 409 |
| An idempotency key that another call holds for more than 10 s | `unavailable` | 503 |
| A retry of `CreateEnrollmentToken`, `CreateAdminKey`, or `CreateAPIKey` with an idempotency key that made a token or a key already | `already_exists` | 409 |
| A machine name that is taken ([the machine API](/reference/machine-api/index.html.md#createmachine)). The message starts with `already exists:`. | `already_exists` | 409 |
| No node with room for a machine, or no free SSH port ([Placement](/reference/placement/index.html.md#when-no-node-has-room)). The message starts with `no capacity:`. | `unavailable` | 503 |
| A request message larger than 4 MiB | `resource_exhausted` | 429 |
| A health `Watch` while 64 others are open on the listener | `resource_exhausted` | 429 |
| The deadline of the call ran out | `deadline_exceeded` | 504 |
| The client canceled the call | `canceled` | 499 |
| A Connect error from a library or from the decode of a stream | its own code | its own status |
| An error before the interceptors, such as a request that Connect cannot decode, a message larger than 4 MiB, or an unknown content type | the code that Connect gives it | the status that Connect gives it |
| Any other error | `internal` | 500 |

An `unavailable` error on a busy key means that the first call with the key still runs. A retry with the same key gets the result of that call, and the SDKs retry `unavailable`.

### Answers of calls that changed nothing

Some answers carry the header `Hopbox-Not-Run: 1`. The header says that the call changed nothing on the server. A client can then run the call again, also a call that changes state and has no idempotency key. The Go client and the TypeScript SDK do this for `unavailable`, `resource_exhausted`, and `deadline_exceeded` ([Go client](/reference/go-client/index.html.md#retries), [TypeScript SDK](/reference/ts-sdk/index.html.md#retries)).

These answers carry the header:

| Answer | Code |
| --- | --- |
| Each refusal of the request gate: a credential that a resolver refused or failed to check, and an `Authorization` header that no resolver knows ([Request gate](#request-gate)) | `unauthenticated` or `unavailable` |
| Each rate limit of the table above, the rows with "Too many". Also the limits of open access calls to the node of a machine, for one caller and for one org ([the machine API](/reference/machine-api/index.html.md#who-can-call-them)). The service layer checks each limit before it writes, and a refused call commits nothing. | `resource_exhausted` |
| An idempotency key that another call holds for more than 10 s | `unavailable` |
| A refusal of an edge server for too many open requests ([the edge role](/reference/edge/index.html.md#listeners)) | HTTP 429 or 503, which a client reads as `unavailable` |
| An edge server that got no connection to the control role ([the edge role](/reference/edge/index.html.md#forwarded-requests)) | HTTP 502, which a client reads as `unavailable` |
| Each refusal of maintenance mode ([Maintenance mode](#maintenance-mode)) | `unavailable`, or HTTP 503 on a route that is not a call |

Every other answer can follow a call that ran, also an `unavailable` answer. An example is a sign-in link whose mail failed after the server stored the link. A usage limit, such as the most orgs of one account, carries no header, because a retry soon after gets the same refusal.

Some refusals are not answers of a call, so they carry no header:

- The WebSocket endpoint `/api/ws/exec` reports an error with a close code and a reason ([the machine API](/reference/machine-api/index.html.md#the-websocket-endpoint)).
- The MCP server reports the error of a tool call in the result of the tool ([the MCP server](/reference/mcp/index.html.md#errors)).
- The limit of certificate renewals of a server answers on the control channel, not with an error of a call ([the control channel](/reference/control-channel/index.html.md#certificate-renewal)).

```text
$ curl -si -X POST -H 'Content-Type: application/json' -H 'Authorization: Bearer hbx_unknown' -d '{}' \
    https://example.com/hopbox.v1.OrgService/ListOrgs
HTTP/2 401
hopbox-not-run: 1
...
{"code":"unauthenticated","message":"the credential is not valid"}
```

The message of an error that the service layer refuses, such as a bad domain, says which rule the request broke. Every other error gets a fixed message. So it never shows the internals of the server to a client, such as the address of the database:

| Code | Message |
| --- | --- |
| `deadline_exceeded` | `the call ran out of time` |
| `canceled` | `the call was canceled` |
| `internal` | `internal error. The server log has the details under the request ID` |
| a code from a library | `the call failed with the code <code>. The server log has the details under the request ID` |

The server writes the cause of these errors only to its log, with the request ID ([Logs](#logs)).

Connect refuses some requests before any interceptor runs. Examples are a body that does not decode, a message larger than 4 MiB, and an unknown content type. These errors keep the text of Connect, such as `protocol error: incomplete envelope`, because the error mapping never sees them. Their text describes only the request, never the server. They count only in `hopbox_http_requests_total`, not in `hopbox_api_calls_total` ([Metrics](/reference/metrics/index.html.md#metrics-of-the-api)). They get no `api call` log record.

### Enum names in JSON

In the JSON codec, an enum value is its full name, such as `VAR_SCOPE_PRIVATE`. A request with a name that the enum does not know gets `invalid_argument`, also a name in lowercase, such as `private`. The handler never runs. The message names the field and lists the values of the enum:

```text
unmarshal message: scope is "VAR_SCOPE_PRIVAT", which is not a value of VarScope. Use one of VAR_SCOPE_SHARED, VAR_SCOPE_PRIVATE, VAR_SCOPE_ALL
```

The JSON codec of Connect drops an unknown name, and the field then holds its unspecified value. For some fields, the unspecified value is the default. Examples are the shared scope of `SetVar` and `DeleteVar`, the member kind of `CreateAPIKey`, and the restart policy `always` of `CreateMachine`. So without the check, a typo runs the call with a choice that the client did not make. The API listener, the internal listener, and the admin socket all make this check.

The JSON codec still drops an unknown field, so a client with a newer schema keeps working. The unspecified value itself, or an enum number, passes the codec. The service layer takes the unspecified value as the default of the field, or refuses it where the field has no default. It refuses a number that the enum does not know. The binary protobuf codec carries numbers, not names, so it needs no such check.

The check runs in the codec, before the request gate, as for any body that does not decode. So an anonymous call with an unknown name gets `invalid_argument`, not `unauthenticated`. The same call with a known name gets `unauthenticated`. The refusal counts only in `hopbox_http_requests_total`, as the paragraph "Connect refuses some requests before any interceptor runs" in [Errors](#errors) says.

### When a listener stops

When a listener stops, it lets each running unary call finish. A unary call that still reads its request body gets 2 s more for the read. Then the read fails, so a stalled upload cannot hold the stop. A lifecycle call of a machine that waits for its machine ends its wait at once. It answers with the machine as it is and `done` false ([the machine API](/reference/machine-api/index.html.md#the-wait)). A stream ends at once: the server cancels its context and ends every read of its request body. So a stream that waits for the client, such as a reflection stream, ends too ([hopboxd](/reference/hopboxd/index.html.md#run)). A shutdown of the HTTP server does not track a connection that a handler took over. So the listener ends each open WebSocket itself, with the close code 1001 and the reason `the server shuts down. Connect again`. A running tool call of the MCP server ends at once, as a stream does ([the MCP server](/reference/mcp/index.html.md#when-the-listener-stops)).

Example:

```text
$ curl -s -X POST -H 'Content-Type: application/json' -d '{}' \
    http://127.0.0.1:7080/hopbox.admin.v1.InstallConfigService/GetInstallConfig
{"code":"unauthenticated","message":"this call needs a credential"}
```

### Streams and their credential

The gate checks the credential of a call once, at its start. A stream can run for hours, so it checks its credential again while it runs. `Service.GuardStream` of `internal/service` gives each stream a context. The context ends with the credential:

- At the expiry of the credential ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md)). For a session, that is the end of its lifetime, or 7 days after its last use, whichever comes first ([Sessions, cookies, and sign-in links](/reference/sessions/index.html.md)). For a CLI token, it is the end of the newest live access token of its CLI sign-in. So a stream lives on while the CLI refreshes its tokens. For an API key, it is the expiry of the key.
- At the first check that finds the credential ended. Examples are a sign-out of a session, and the end of a CLI sign-in. The reuse of an old refresh token ends a CLI sign-in too, with its whole family. Others are the revoke of an API key, the leave of the account of a member key from its org, and an expiry. The check runs every 30 s (`service.DefaultCredentialRecheck`). A check that cannot reach the control store keeps the stream. It logs `the check of the credential of a stream failed` at level warn with `component=service` and `err`, and tries again at the next check.

The stream then ends with `unauthenticated` and the message `the credential of this stream ended. Sign in again: ...`. `WatchMachines`, `Exec`, `Upload`, `Download`, `StreamMachineLogs`, and the WebSocket of exec run under such a guard. A session, a CLI token, a member key, and an org key can hold a stream. Any other kind of caller gets `unauthenticated` at once.

## Maintenance mode

Maintenance mode stops every change of the install through the API and the control channels, so that `hopboxd admin db export` can copy a control store that no longer changes ([hopboxd](/reference/hopboxd/index.html.md#admin-maintenance-on)). It is a flag of the install config ([Control store tables](/reference/control-store/index.html.md#install_config)). Only root on a control server turns it on or off, with `SetMaintenance` on the admin socket ([the admin API](/reference/admin-api/index.html.md#setmaintenance)). The code is in `internal/api/maintenance.go` and `internal/service/maintenance.go`.

In maintenance mode, every listener of the control role refuses each call that can change the control store: the API listener, the admin socket, and the internal listener, also for the requests that the edge servers forward. The maintenance gate sorts each call into one of three kinds:

- A call of gRPC health or of gRPC reflection ([Reflection and health](#reflection-and-health)), and `SetMaintenance`, pass as they are. The policy lets only `local_root` call `SetMaintenance`, so the admin socket always takes it. No other write could turn maintenance mode off.
- A call whose method has the option `(hopbox.v1.read_only)` ([API definitions](/reference/api-definitions/index.html.md#hopboxv1read_only)) runs as a read call. Each write that it makes runs under the maintenance mode of the process, such as its audit entry, or the last use of a machine for its auto-delete timeout. In maintenance mode, the call fails at that write, before the write, with the refusal below. A read call that writes nothing works in maintenance mode.
- Every other call is refused before the request gate, so it reads no credential and changes nothing.

So these calls work in maintenance mode:

| Service | Calls |
| --- | --- |
| `hopbox.v1.InstallService` | `GetInstallInfo` |
| `hopbox.v1.AccountService` | `GetAccount`, `ListSignInMethods`, `ListCLISignIns` |
| `hopbox.v1.AuditService` | `ListAuditEntries` |
| `hopbox.v1.APIKeyService` | `ListAPIKeys` |
| `hopbox.v1.MachineService` | `GetMachine`, `ListMachines`, `ListMachineSizes`, `WatchMachines` |
| `hopbox.v1.OrgService` | `GetOrg`, `ListOrgs`, `GetOrgUsage`, `ListOrgMembers`, `ListInvitations` |
| `hopbox.v1.SSHKeyService` | `ListSSHKeys` |
| `hopbox.v1.VarService` | `ListVars` |
| `hopbox.admin.v1.InstallConfigService` | `SetMaintenance` |
| gRPC health and reflection | every call |

Each other call is refused, also these reads: every read of the admin API, such as `GetInstallConfig` and `ListServers`, because it writes its audit entry ([the admin API](/reference/admin-api/index.html.md)), and `Download` and `StreamMachineLogs`, because each records a use of its machine. Their node channels are closed in maintenance mode anyway. The audit log thus never loses an access that it would have recorded.

Each refusal changes nothing. It has the code `unavailable`, the message `the install is in maintenance mode, so it takes only reads. Retry later`, and the headers `Hopbox-Not-Run: 1` and `Retry-After: 30`:

```text
$ curl -si -X POST -H 'Content-Type: application/json' -d '{"email":"ann@example.org"}' \
    https://example.com/hopbox.v1.AuthService/RequestMagicLink
HTTP/2 503
hopbox-not-run: 1
retry-after: 30
...
{"code":"unavailable","message":"the install is in maintenance mode, so it takes only reads. Retry later"}
```

The plain HTTP routes that can change the install answer HTTP 503 with the same headers and message: the sign-in with GitHub and Google at `/auth/<provider>/start` and `/auth/<provider>/callback` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md)), the sign-in of private machine URLs at `/auth/machine` ([the edge role](/reference/edge/index.html.md)), and the WebSocket endpoint `/api/ws/exec`. The console, health, and the MCP endpoint keep serving. Each tool of the MCP server calls the API, so it gets the refusal of its call.

The last use of a session, an API key, an admin API key, or a CLI token is a write of the request gate too, but a lost time of a last use is harmless. So in maintenance mode, the request gate skips that write, and the call goes on ([CLI tokens, API keys, admin API keys, and SSH keys](/reference/credentials/index.html.md#lifetimes)).

A read call can run while maintenance mode turns on. Each of its writes either ends before `SetMaintenance` returns, or is refused before it changes anything. So no write of a read call comes after the turn to on either. A stream such as `Download` that runs at the turn ends at its next record of the use of its machine.

When maintenance mode turns on, the control process ends each call without the option `(hopbox.v1.read_only)` that still runs: the unary calls, the streams such as `Exec`, and the control channels of the servers ([the control channel](/reference/control-channel/index.html.md#maintenance-mode)). A call that maintenance mode ended gets `unavailable` with the message `the install went into maintenance mode, which ended this call. Retry later`, and no `Hopbox-Not-Run` header, because it can have changed something before its end. The leader also pauses its jobs ([hopboxd](/reference/hopboxd/index.html.md#leader-jobs)). A turn to on with `SetMaintenance` succeeds only after each of these ended in every live control process of the install while maintenance mode stayed on, so no change of the API or of a server comes after it. The other control processes report this in their rows of `control_processes` ([the admin API](/reference/admin-api/index.html.md#setmaintenance)). The call gives up with `unavailable` after at most 30 s, the deadline of the call, and maintenance mode stays on. If maintenance mode turns off during the wait, for example by a second root on the socket, the turn to on fails with `unavailable: unavailable: maintenance mode turned off while it waited for the calls, control channels, and leader jobs to end`, because the work runs again. `hopboxd admin db export` also reads the flag in the snapshot of its dump, and refuses a store out of maintenance mode.

The request gate checks a server certificate on the internal listener, also for health and reflection. The first use of a certificate, which retires the other certificates of its server, and the alarm of a retired certificate write to the store. In maintenance mode, the gate refuses such a call with the refusal above, before the write ([Enrollment and the install PKI](/reference/enrollment/index.html.md)).

When maintenance mode turns off, the listeners take every call again, the servers dial their control channels again, and the leader runs its jobs again. Each control process reads the flag at its start, and again at each change event `install.maintenance_changed` ([Control store tables](/reference/control-store/index.html.md#events)). So a control process that starts on a store in maintenance mode, such as a copy of `hopboxd admin db export`, starts in maintenance mode too.

## Reflection and health

gRPC server reflection is on, so `grpcurl` can list and call the services. The server serves the reflection services `grpc.reflection.v1.ServerReflection` and `grpc.reflection.v1alpha.ServerReflection`. They list the services of their listener.

The health service is `grpc.health.v1.Health`, with the methods `Check` and `Watch`. A check can name the whole server, with an empty service name, or one service of the listener. It returns `SERVING` while the control store answers a ping within 2 s, and `NOT_SERVING` otherwise. A check of an unknown service gets `not_found`. `Watch` sends the status at once. Then it checks again every 5 s, and sends the status each time that it changes. Each listener serves at most 64 open `Watch` calls. One more gets `resource_exhausted`.

```sh
grpcurl -plaintext 127.0.0.1:7080 list
grpcurl -plaintext 127.0.0.1:7080 grpc.health.v1.Health/Check
```

## InstallService

`hopbox.v1.InstallService` describes the install to a client before it signs in. The generated reference is [Public API](/api/hopbox.v1.openapi.yaml).

### GetInstallInfo

`GetInstallInfo` returns the facts that a client needs before its first call. It needs no credential. It has no side effects, so the Connect protocol also serves it with HTTP GET.

| Item | Value |
| --- | --- |
| Procedure | `/hopbox.v1.InstallService/GetInstallInfo` |
| Policy action | `install.get_info`, every caller |
| Idempotent option | no |
| Audit entry | none |

The request `GetInstallInfoRequest` has no fields. The response `GetInstallInfoResponse` has these fields:

| Field | Type | Description |
| --- | --- | --- |
| `domain` | string | The domain of the install, for example `example.com`. It is empty until an instance admin sets it with `UpdateInstallConfig` ([the admin API](/reference/admin-api/index.html.md#updateinstallconfig)). |
| `sign_in_methods` | repeated `SignInMethod` | The sign-in methods that anyone can use on the install ([the sign-in, account, and audit API](/reference/auth-api/index.html.md#signinmethod)). `SIGN_IN_METHOD_EMAIL_LINK` is in the list while the install has an SMTP config ([SMTP config](/reference/smtp/index.html.md)) or the file mail transport ([Mail](/reference/mail/index.html.md#mail-transports)). `SIGN_IN_METHOD_GITHUB` and `SIGN_IN_METHOD_GOOGLE` are in it while the install has a domain and the client of the provider ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#config)). The first-admin login link is never in it, because only root can make one. |
| `ssh_host_key` | string | The public host key of the SSH front door, in the `authorized_keys` format, for example `ssh-ed25519 AAAA...`. The edge servers show it on every SSH port ([the SSH front door](/reference/ssh-front-door/index.html.md#keys)). It is empty until a control process made the keys of the front door, which needs a key-encryption key. |
| `version` | string | The version of `hopboxd` on the server that answered, for example `v0.1.0` ([the version command](/reference/version/index.html.md#version-format)). |
| `mail_to_files` | bool | `true` on a dev or test install with the file mail transport ([Mail](/reference/mail/index.html.md#mail-transports)). Then no magic link or invitation reaches its recipient by email. The console uses it to tell the person to ask the operator for the link. |

In the JSON codec, a field with its default value is left out, so an empty `domain` does not appear.

Example with the Connect protocol and HTTP GET:

```text
$ curl -s --http2-prior-knowledge \
    'http://127.0.0.1:7080/hopbox.v1.InstallService/GetInstallInfo?encoding=json&message=%7B%7D'
{"domain":"example.com","version":"v0.1.0"}
```

Example with gRPC:

```sh
grpcurl -plaintext 127.0.0.1:7080 hopbox.v1.InstallService/GetInstallInfo
```

## Logs

The API writes these records with `component=api`. The access log record covers every call, also refused ones.

| Message | Level | Fields |
| --- | --- | --- |
| `api call` | info | `procedure`, `protocol`, `peer`, `caller`, `code`, `duration`, and `request_id`. `protocol` is `connect`, `grpc`, `grpcweb`, or `websocket` for the WebSocket endpoint of exec. `peer` is the address of the client, empty on the admin socket. `code` is `ok` or the error code. `caller` is the caller kind, or `unresolved` after the gate refused the credential. A call of a tool of the MCP server also has `mcp_request_id`, the request ID of its MCP request ([the MCP server](/reference/mcp/index.html.md#logs)). |
| `api call failed` | error | `procedure`, `err`, `request_id`. For an `internal` error. |
| `api call failed` | warn | `procedure`, `err`, `request_id`. For a Connect error from a library, which keeps its code. |
| `api call stopped` | info | `procedure`, `err`, `request_id`. For `deadline_exceeded` and `canceled`. |
| `api credential refused` | info | `err`, `request_id`. A resolver refused a credential. |
| `api credential check failed` | error | `err`, `request_id`. A resolver failed to check a credential. |
| `machine URL sign-in failed` | error | `err`, `request_id`. `GET /auth/machine` failed with `500` ([the edge role](/reference/edge/index.html.md#answers-of-authmachine)). |
| `websocket upgrade failed` | info | `path`, `err`, `request_id`. The upgrade of a request to the WebSocket endpoint `/api/ws/exec` failed ([the machine API](/reference/machine-api/index.html.md#the-websocket-endpoint)). |

The HTTP server writes its own errors, such as a protocol error of a client, at level warn with `component=http`.
