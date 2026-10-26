# Sealing

> How hopbox seals secrets: the key files and their checks, the places, the sealed columns, the envelope format, rotation, and rewrap.

hopbox seals secrets with envelope encryption. This page gives the key files, the checks on them, the format of a sealed value, and how a rotation works. The code is in `internal/seal`.

## Terms

| Term | Meaning |
| --- | --- |
| data key | A random 32-byte key. Each sealed value gets its own data key. |
| key-encryption key (KEK) | A 32-byte key that seals data keys. It never enters the database. |
| keyring | The set of KEKs that one install has. It holds the current KEK and the older ones. |
| envelope | The bytes of one sealed value: the sealed data key, the sealed value, and the data that opens them. The database stores only envelopes. |
| place | Where a sealed value belongs, for example its org and its name. The place is the associated data of the seal. Associated data is data that the encryption covers but does not hide. |

## How a value is sealed

1. hopbox makes a new random data key.
2. AES-256-GCM seals the value with the data key. The associated data is the place.
3. The keyring seals the data key with the current KEK. The associated data is the KEK ID plus the place.
4. hopbox writes the envelope and clears the data key from memory.

To open a value, hopbox first reads the KEK ID from the envelope. Then it opens the data key with that KEK. Then it opens the value with the data key. Both steps check the place. A value that someone copies to another place does not open. Another place is another org, another name, another scope, or another owner. A copy to another row with the same place still opens. An older envelope that someone puts back in the same place also opens, so the seal does not detect a rollback. A change to any byte of the envelope makes the open fail.

Both seals use AES-256-GCM with a random 12-byte nonce. hopbox does not use AES key wrap (RFC 3394). Go's standard library has no AES key wrap, and AES key wrap cannot bind the associated data. With random nonces, NIST SP 800-38D allows 2^32 seals with one key. Each data key seals one value. The KEK seals one data key for each seal and for each rewrap. Rotate the KEK long before it reaches that limit. hopbox does not count the seals.

## Places

The code gives each kind of place its own Go type. The linter (`exhaustruct_v5` in `.golangci.yml`) makes every literal of these types set every field, and `Seal` refuses a missing field.

| Type | Fields | Used for |
| --- | --- | --- |
| `OrgSecret` | `Org`, `Scope`, `Owner`, `Name` | A secret of an org. `Scope` is `shared`, `private`, or `all`. `Owner` is the account that owns a `private` value. It must be the nil UUID for `shared` and `all`. |
| `InstallSecret` | `Name` | A value of the install that no org owns. Examples are the SMTP password, the client secret of a sign-in provider, and the key of the install CA. The key of the install CA uses the name `install_ca.key`, in the table `install_ca` ([Control store tables](/reference/control-store/index.html.md#install_ca)). |

These columns of the control store hold sealed values:

| Column | Place |
| --- | --- |
| `install_ca.sealed_key` | `InstallSecret{Name: "install_ca.key"}` ([Control store tables](/reference/control-store/index.html.md#install_ca)) |
| `install_config.smtp_password` | `InstallSecret{Name: "smtp.password"}` ([SMTP config](/reference/smtp/index.html.md#password)) |
| `install_config.github_client_secret` | `InstallSecret{Name: "oauth.github.client_secret"}` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#config)) |
| `install_config.google_client_secret` | `InstallSecret{Name: "oauth.google.client_secret"}` ([Sign-in with GitHub and Google](/reference/oauth/index.html.md#config)) |
| `acme_account.sealed_key` | `InstallSecret{Name: "acme_account.key"}`: the PKCS #8 key of the ACME account ([the certificate of the install](/reference/acme/index.html.md#where-the-certificate-lives)) |
| `install_certificate.sealed_key` | `InstallSecret{Name: "install_certificate.key"}`: the PKCS #8 key of the certificate of the install ([the certificate of the install](/reference/acme/index.html.md#where-the-certificate-lives)) |
| `ssh_front_door_keys.sealed_host_key` | `InstallSecret{Name: "ssh_host.key"}`: the PKCS #8 Ed25519 host key of the SSH front door ([the SSH front door](/reference/ssh-front-door/index.html.md#keys)) |
| `ssh_front_door_keys.sealed_client_key` | `InstallSecret{Name: "ssh_client.key"}`: the PKCS #8 Ed25519 key with which the SSH front door signs in to the machines ([the SSH front door](/reference/ssh-front-door/index.html.md#keys)) |
| `env_vars.sealed_value` | `OrgSecret{Org, Scope, Owner, Name}` of the row: its org, its scope, the owner of a private value or the nil UUID, and its name ([the env var API](/reference/vars-api/index.html.md#sealing), [Control store tables](/reference/control-store/index.html.md#env_vars)) |

Each feature that seals a value adds its column to the list `sealedColumns` in `internal/service/keyring.go`. Then the [rewrap](#rewrap) and the KEK report cover the column.

The design binds a secret to its org ID and its name. Two members can each have a private secret with the same name, so `OrgSecret` also binds the scope and the owner.

A name is 1 to 65535 bytes.

The associated data is these bytes:

| Field | Size | Value |
| --- | --- | --- |
| format | 1 byte | `0x01` |
| kind | 2-byte length, then the bytes | `org-secret` or `install-secret` |
| each field of the place, in the order of the table above | 2-byte length, then the bytes | An org or an owner is its 16 UUID bytes. A scope and a name are their UTF-8 bytes. |

All lengths are big endian.

## Envelope format

An envelope of format 1 has these fields, in this order:

| Field | Size | Value |
| --- | --- | --- |
| format | 1 byte | `0x01`. A new format gets a new byte, and hopbox keeps reading the old formats. |
| key ID length | 1 byte | 1 to 255 |
| key ID | key ID length | The ID of the KEK that sealed the data key, for example `file:2`. |
| wrapped key length | 2 bytes, big endian | 1 to 65535 |
| wrapped key | wrapped key length | The data key, sealed by the keyring. For the file keyring, this is 60 bytes: a 12-byte nonce, the 32 sealed key bytes, and a 16-byte GCM tag. |
| nonce | 12 bytes | The nonce of the value. |
| ciphertext | the rest | The sealed value, then its 16-byte GCM tag. |

For example, the KEK `file:1` seals a value of n bytes into n + 98 bytes: 1 + 1 + 6 + 2 + 60 + 12 + n + 16.

## Key files

The file keyring is the only keyring today. A KMS keyring and a HashiCorp Vault keyring can come later as other implementations of the `Keyring` interface. They will seal the data key in the remote service. Each key ID starts with the kind of its keyring, such as `file:`, and names one KEK for good.

| Path | Owner | Mode | Contents |
| --- | --- | --- | --- |
| `/var/lib/hopbox/kek` | the user of `hopboxd` (root on a server) | `0700` | The keyring dir. |
| `/var/lib/hopbox/kek/<n>.key` | the user of `hopboxd` | `0400` | The 32 raw bytes of the KEK with the ID `file:<n>`. `n` is a decimal number from 1 up, without leading zeros. |
| `/var/lib/hopbox/kek/<n>.staged` | the user of `hopboxd` | `0400` | The 32 raw bytes of a staged KEK with the ID `file:<n>`. A staged KEK opens values but never seals one, until a [promote](#several-control-servers) makes it a `<n>.key` file. |
| `/var/lib/hopbox/kek/.<n>.key.<random>.tmp` and `.<n>.staged.<random>.tmp` | the user of `hopboxd` | `0400` | A key that a rotation or a stage is writing. Each one gets its own random name. The file exists only during the write. If it stops half way, for example on a power loss, you can delete its temp file. |

The paths in the table are for the default `data_dir`. `hopboxd` keeps the keyring in `<data_dir>/kek` ([the bootstrap config of hopboxd](/reference/hopboxd-config/index.html.md#keys)).

The KEK with the highest `n` of the `<n>.key` files is the current KEK. New seals use it. A staged file never makes a KEK current, also when its `n` is higher. A `hopboxd` of an older release reads only the `.key` files, so it does not see a staged KEK. hopbox ignores the files in the dir whose names have neither the form `<n>.key` nor the form `<n>.staged`. A new key number, from a rotation or a stage, is 1 more than the highest number of both kinds of file.

In these cases, hopbox refuses the whole keyring with the error `is not safe to hold key-encryption keys`:

- The keyring dir, a key file, or a staged file is a symlink.
- The keyring dir is not a dir, or a key file or a staged file is not a regular file. For example, a FIFO or a device is refused.
- The mode of the dir, of a key file, or of a staged file gives any permission to the group or to others.
- The dir, a key file, or a staged file belongs to a user other than the user of `hopboxd`.

hopbox also refuses a key file or a staged file that does not hold exactly 32 bytes. Root can read any file, so the mode and owner checks are a policy. If others can read a key, the key is possibly not secret any more.

For each seal, open, and rewrap, the keyring checks the dir and the stat of every key file. Thus a new KEK takes effect without a restart. A deleted key file or a key file with a loose mode also takes effect at once, as it does after a restart. For each call, the keyring also reads the 32 bytes of the KEK that the call needs. A stat can miss a rewrite of the file, for example after the reuse of an inode or a restored modification time. The bytes cannot. The keyring keeps the AES key schedule of each KEK in memory, and uses it again only while the bytes on disk match the bytes that it read before.

Each control server needs the same key files. With more than one control server, the order matters. First copy the new key file to every control server. Then make it the current KEK. Otherwise one server seals values that the other servers cannot open. A rotation makes the new key current at once, so it is only for one control server. With more than one, [stage and promote](#several-control-servers) the key. Two servers that each rotate on their own make two different KEKs with the same ID.

Back up the keyring dir, and keep the backup apart from the database backups. Without the KEKs, no sealed value opens. A thief with both the database and the KEKs can open every value. Only the control role uses the keyring. The node, edge, and DNS roles never read it, and the linter refuses an import of `internal/seal` from their code.

## Rotation

A rotation adds a new KEK and makes it the current KEK. It does not delete the old KEKs, so the values that they sealed still open.

A rewrap moves one sealed value to the current KEK. It opens the data key with the old KEK and seals it again with the current KEK. It keeps the nonce and the ciphertext of the value. It does not decrypt the value. A rewrap of a value that the current KEK already sealed changes nothing and checks nothing. Only an open checks the place and the value fully. A job that rewraps many values asks the keyring for the current KEK once, and passes its ID to each rewrap. So a remote keyring gets one call for the whole batch.

Keep each old key file until no value uses its KEK. If you delete a KEK that a value still uses, that value fails to open with the error `the keyring does not have the key-encryption key of this value`. A rotation does not rewrap the values. The [rewrap](#rewrap) does. It also moves the key of the install CA, the value of the column `install_ca.sealed_key` ([Control store tables](/reference/control-store/index.html.md#install_ca)).

## How a rotation works

An instance admin rotates the KEK on an install with one control server through the admin API: `sudo hopboxd admin kek rotate` on a control server ([hopboxd](/reference/hopboxd/index.html.md#admin-kek-rotate)), or the call `RotateKEK` ([the admin API](/reference/admin-api/index.html.md#rotatekek)). The audit log records each rotation. If the keyring of a control server has no KEK, `hopboxd install` makes the first one with the same library call ([hopboxd](/reference/hopboxd/index.html.md#install)). The install writes no audit entry for it, because the control store does not run yet. A rotation fails while another control process is live ([Several control servers](#several-control-servers)). The library call `seal.RotateFileKeyring` does the work:

1. If the keyring dir does not exist, it creates the dir with mode `0700`, and creates missing parent dirs with mode `0755`.
2. It checks the dir and every key file in it, and refuses an unsafe keyring.
3. It writes the new key to a temp file with a random name, and syncs the file.
4. It links the temp file to the name `<n>.key`, where `n` is 1 more than the highest key. The first KEK gets the ID `file:1`. If that name exists, the link fails and the rotation fails. So a rotation never overwrites a key.
5. It removes the temp file, syncs the dir, and returns the ID of the new KEK.

Each rotation has its own temp file. Thus two rotations at the same time cannot link a key that is not complete. The second one fails, and the caller can retry.

## Several control servers

Each control server holds its own copy of the key files, and all copies must match. A rotation makes the new key current only on the server that runs it. So it fails while the install has a control process on another host, live or not: a control server that stopped cleanly or ended without its stop keeps its row of `control_processes` for 24 hours. A rotation and a stage give the new key the next ID of the install, one after the newest key of the keyring and of `kek_fingerprints`. A rotation writes its key as a staged file, records its fingerprint, and only then makes it current, so a refused fingerprint leaves the current key as it was. Use a stage and a promote instead. The new key goes through these steps:

Every control server must run a release with migration `0045` before a `kek stage` or a `kek promote`. A control process of an older release records no row in `control_processes`, so a promote cannot see that it lacks the new key.

1. Stage the key. `sudo hopboxd admin kek stage` ([hopboxd](/reference/hopboxd/index.html.md#admin-kek-stage)) or the call `StageKEK` ([the admin API](/reference/admin-api/index.html.md#stagekek)) adds the file `<n>.staged` to the key dir of this server. The current KEK stays the same, so the staged key seals nothing yet. The command prints the ID and the path of the file.
2. Copy the staged file to the same path on every other control server, as root, with the owner of `hopboxd` and the mode `0400`. hopbox does not copy a staged key for you. Use `scp` or `rsync` over SSH, and keep the key out of the shell history, the logs, and the database backups.
3. Promote the key. `sudo hopboxd admin kek promote <id>` ([hopboxd](/reference/hopboxd/index.html.md#admin-kek-promote)) or the call `PromoteKEK` ([the admin API](/reference/admin-api/index.html.md#promotekek)) makes the staged key current on every control server.
4. [Rewrap](#rewrap) the sealed values, and remove the old key files.

Each control process lists in the column `kek_ids` of its row the IDs of its key files and its staged files ([Control store tables](/reference/control-store/index.html.md#control_processes)). The column `kek_current` holds the highest `.key` file. A promote refuses while another live control process does not list the key, or lists it with another fingerprint ([Key fingerprints](#key-fingerprints)). Each process renews its row every 5 s, so a copy shows within that time.

A promote does these steps:

1. It checks that the ID is the ID of a file key, that this server has a file of the key, and that no key file with a higher number exists. A key that is not the highest `.key` file would never seal.
2. It records the fingerprint of the key, if the install has none, and checks that it matches. It checks that every other live control process lists the key in `kek_ids`, with the same fingerprint in `kek_fingerprints`.
3. In one transaction, it sets `install_config.promoted_kek` to the key ([Control store tables](/reference/control-store/index.html.md#install_config)), publishes the change event `keyring.kek_promoted`, and writes the audit entry.
4. It promotes the file on this server: it links `<n>.staged` to `<n>.key`, removes the staged name, and syncs the dir. A second promote of the same file changes nothing.
5. It waits until every other live control process reports the key as `kek_current`. It reads the rows every 250 ms, until the deadline of the call.

Each control process applies the promote on its own. It promotes its staged copy of `promoted_kek`: at its start, before anything seals, at each change event `keyring.kek_promoted`, and at each renewal of its row while its key dir has a staged file or while it misses the promoted key. So a server that was down during the promote makes the key current when it starts. A server that has no file of the promoted key logs the error `the install promoted a key-encryption key that this control server has no file of. Copy its key file from another control server` once for each missing key. It keeps sealing with its old key until the copy arrives. Copy the file as `<n>.staged`, and the server applies the promote at its next renewal.

If the deadline of the call ends before every process seals with the key, the call fails with `unavailable`. The promote stays recorded, and the other processes go on. Run `admin kek list` or `admin control list` to see which process is behind.

A start of the control role also checks that its key files fit the control store, in the `store` unit, before anything seals ([hopboxd](/reference/hopboxd/index.html.md#run)). The start fails in each of these cases:

- The install promoted a key, and this server has no file of it.
- A `.key` file has another fingerprint than the one that the install recorded for its ID ([Key fingerprints](#key-fingerprints)). A staged file with another fingerprint does not fail the start, as the list of [Key fingerprints](#key-fingerprints) says.
- A sealed value names a key that this server has no file of. The message is `<n> sealed values name the key-encryption key <id>, and this control server has no file of it. Copy the key files of a running control server here`.
- The key files do not open the install CA ([Control store tables](/reference/control-store/index.html.md#install_ca)). A keyring without any key passes only while the install has no CA. The check makes the CA when none exists.

These checks guard a second control server whose first key came from `hopboxd install` instead of a copy of the key files of the first server. That server would seal values that no other control server can open. Copy the key dir of a running control server to the new server before its first start ([Run the control role on a laptop](/guide/run-hopboxd/index.html.md#add-a-second-control-server)).

### Key fingerprints

Two control servers can each make a key with one ID, such as two stages of `file:3`, or a rotation while the other server is down. The two key files then hold different keys, and each server seals values that the other one cannot open. So the install records a fingerprint of each key in the table `kek_fingerprints` ([Control store tables](/reference/control-store/index.html.md#kek_fingerprints)). The fingerprint is the first 16 bytes, in hex, of an HMAC-SHA256 of the text `hopbox kek fingerprint v1` under the key. It tells whether two key files hold the same key, and nothing of the key.

The first fingerprint of an ID wins. These steps record it, or fail when a key file differs from it:

- A stage. A second stage of the same ID with other bytes fails, and removes its staged file.
- A rotation.
- A promote, for the promoted key.
- The start of the control role, and each renewal of the row of a control process, for every key file of the server. A renewal that finds another key in a `.key` file fails, so the row of the process stops being live, and the process fences itself. Nothing seals with a staged file yet, so a staged file with another key does not fail. The row leaves its ID out of `kek_ids` and `kek_fingerprints`, so a promote of that ID refuses, and the process logs the error `a staged key file of this control server holds another key than the install recorded for its ID, so its row leaves it out. Copy the staged file of the server that staged it here again` with `key_id` and `dir`.
- The apply of a promote on each control process. If its staged copy of the promoted key holds another key, the process keeps sealing with its current key and logs `cannot apply the promoted key-encryption key` with the error. At a start, the start fails.

Each row of `control_processes` lists the fingerprints of the key files of its process in `kek_fingerprints`, as `<key ID>=<fingerprint>`. A promote compares them, and so does a rewrap: a [rewrap](#rewrap) refuses while a live control process does not hold the current key with its fingerprint, because that process could not open the rewrapped values.

An example, with the control servers `cp1` and `cp2`:

```text
cp1$ sudo hopboxd admin kek stage
file:2	/var/lib/hopbox/kek/2.staged
cp1$ sudo hopboxd admin kek list
file:1	3	current
file:2	0	staged
cp1$ sudo scp -p /var/lib/hopbox/kek/2.staged root@cp2:/var/lib/hopbox/kek/2.staged
cp1$ sudo hopboxd admin kek promote file:2
file:2
cp1$ sudo hopboxd admin kek rewrap
rewrapped 3 to file:2
file:1	0	
file:2	3	current
```

If `cp2` has no copy yet, the promote fails:

```text
cp1$ sudo hopboxd admin kek promote file:2
failed_precondition: failed precondition: the control process 7a1f... on cp2 (hopboxd v0.3.0) reports no key file of file:2. Copy the staged file to the key dir of that server with mode 0400, and promote again after its next renewal, within 5 s
```

## Rewrap

After a rotation or a promote, the sealed values keep their old KEK. An instance admin moves them to the current KEK, and then removes the old key files:

1. `sudo hopboxd admin kek rewrap` ([hopboxd](/reference/hopboxd/index.html.md#admin-kek-rewrap)), or the call `RewrapSealedValues` ([the admin API](/reference/admin-api/index.html.md#rewrapsealedvalues)). It rewraps every value of every column in the [list of sealed columns](#places). The audit log records the rewrap with the number of values that moved.
2. `sudo hopboxd admin kek list` ([hopboxd](/reference/hopboxd/index.html.md#admin-kek-list)), or the call `ListKEKs`. It reads the KEK ID of each envelope, without the keyring, and counts the values of each KEK. A KEK that a value names, but whose key file is gone, shows as missing from the keyring.
3. Back up the keyring dir. Then delete the key files of the KEKs that seal no value. Never delete the current KEK.

The rewrap commits each value on its own, with a check that the value did not change since its read. A value that a change seals again during the rewrap keeps its new seal. A second rewrap moves what the first one left. After a full rewrap, a second one finds nothing to do. The table `env_vars` has one row for each secret. The rewrap finds the row of a secret by its ID and its old envelope. A set since the read changed the envelope, so the rewrap leaves that row as it is. The rewrap keeps the version and the update time of the row, because the value stays the same.

```text
$ sudo hopboxd admin kek rotate
file:2
$ sudo hopboxd admin kek rewrap
rewrapped 1 to file:2
file:1	0	
file:2	1	current
```

## Errors

| Error | Meaning |
| --- | --- |
| `<path> is not safe to hold key-encryption keys: ...` | The keyring dir or a key file failed a check above. The rest of the message says which check. |
| `a key file holds another key than the install recorded for its ID` | A stage, a promote, a renewal, a start, or the apply of a promote found a key file whose fingerprint differs from the one of its ID ([Key fingerprints](#key-fingerprints)). The message names the ID and both fingerprints. Copy the key files of a running control server, and remove the other file. |
| `the keyring has no staged key file of this key-encryption key` | A promote of a key whose `<n>.staged` file is not in the dir, and that is no `.key` file. The message is `seal: <dir>: the keyring has no staged key file of this key-encryption key file:<n>`. |
| `the keyring has no key-encryption key` | The keyring dir is missing or holds no key file. A rotation creates the first KEK. |
| `the keyring does not have the key-encryption key of this value` | The envelope names a KEK that the keyring does not have. |
| `the sealed value does not open: it was changed, or it belongs to another place` | A byte of the envelope changed, or the value belongs to another place. |
| `the sealed value is not a valid envelope` | The bytes are cut short, or their format byte is not known. |
| `the place of the sealed value is not complete` | The caller left out a field of the place, or set a field that the scope does not allow. |
