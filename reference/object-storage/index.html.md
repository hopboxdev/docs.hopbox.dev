# Object storage

> The S3 client of hopbox: its requests, signatures, TLS, retries that resume a download, errors, logs, and the fake server of the tests. Also the disk backups: their objects and encryption, their steps on the node and in the control role, the jobs of the leader, and the prune.

Object storage is a network service that stores files, called objects, in buckets. hopbox uses a bucket of an S3 server for off-site disk backups. Object storage is optional. Without it, every other feature works.

The code is in `internal/objectstore`. It is a small S3 client with only the requests that hopbox needs, and no SDK.

## The config of an install

An instance admin sets the bucket of the install through the admin API: `sudo hopboxd admin config object-storage set` on a control server ([hopboxd](/reference/hopboxd/index.html.md#admin-config-object-storage-set)), the call `SetObjectStorageConfig` ([the admin API](/reference/admin-api/index.html.md#setobjectstorageconfig)), or the page Config of the Admin area ([Console](/reference/console/index.html.md#admin-area)). The config lives in the row of `install_config` ([Control store tables](/reference/control-store/index.html.md#install_config)). A change adds 1 to the version of the install config.

Without object storage, a backup fails with `failed_precondition` and the message `this install has no object storage, so it makes no backups. An instance admin sets it with hopboxd admin config object-storage set, or on the page Config of the Admin area`. Every other feature works.

[Back up machines to a bucket](/guide/backups/index.html.md) sets up a bucket.

### Fields

| Field | Rule |
| --- | --- |
| endpoint | The https URL of the S3 server, such as `https://s3.gra.io.cloud.ovh.net`, with an optional port. It has no user info, no path, no query, no fragment, no whitespace, and at most 2048 characters. A plain `http` URL fails. hopbox keeps the host in lowercase and drops the port 443, so `https://S3.Example.org:443/` becomes `https://s3.example.org`. |
| region | The region of the signature: 1 to 63 lowercase letters, digits, and dashes, such as `gra`. A server without regions takes any region. Use `us-east-1` there. |
| bucket | The name of the bucket: 3 to 63 lowercase letters, digits, dots, and dashes. It starts and ends with a letter or a digit, has no two dots in a row, and is no IP address. |
| access key ID | 1 to 256 printable ASCII characters without spaces, commas, equals signs, or slashes. The signature header uses these characters to split its fields. |
| secret key | 1 to 256 printable ASCII characters without spaces. hopbox seals it ([The secret key](#the-secret-key)). |
| CA certificates | Optional PEM certificates, at most 64 KiB, for an S3 server with a private CA (Q225 of the plan of phase 3). The client trusts them beside the roots of the system. Each PEM block must be a `CERTIFICATE`. |

### The secret key

hopbox seals the secret key with envelope encryption ([Sealing](/reference/sealing/index.html.md)). The place of the value is the install secret `object_storage.secret_key`. Only the column `install_config.object_storage_secret_key` holds it.

The admin API never returns the secret key (Q228 of the plan of phase 3). `GetObjectStorageConfig` returns only `secret_key_set`. A call without a secret key keeps the secret key that is set now, for example to change the region. A kept secret key never goes to another endpoint, bucket, or account: a new endpoint, bucket, or access key ID needs the secret key again. The audit log records `secret_key_changed`, never the secret key.

The control role opens the secret key for each check, and keeps it in memory only for that check. For the backups of a node, the control role opens the key and sends it to the node on the control channel ([the control channel](/reference/control-channel/index.html.md#node-backups)). The node holds it in memory for the channel that carried it, and never logs it. A rotation of the key-encryption key leaves the secret key sealed by the old key, until a rewrap moves it ([Sealing](/reference/sealing/index.html.md#rewrap)).

### The check

A new config takes effect only after a check (Q226 of the plan of phase 3). The control process that takes the call writes one object of 64 random bytes under the key `hopbox-check/<random>`, reads it back, compares the bytes, and deletes it. So the check proves the endpoint, the certificate, the region, the bucket, both keys, and the rights to write, read, and delete. A failed check changes nothing. `CheckObjectStorage` runs the same check on the saved config at any time ([the admin API](/reference/admin-api/index.html.md#checkobjectstorage)).

The write and the read of the check take at most 20 s, and the delete at most 5 s more, so a check takes at most 25 s. The delete has its own time, so a read that stalled to the end of its 20 s leaves no object in the bucket. Each of its requests has at most 3 attempts, a stall timeout of 10 s, and a connect timeout of 5 s. A check that gets no answer, or an answer that says that the server failed, fails with `unavailable`. Each other failure is `failed_precondition`, because the config must change first. A host that does not resolve is `failed_precondition` too, because such a name is almost always a typo. A closed port and a timeout stay `unavailable`. The message holds the error of the request ([Errors](#errors)), such as:

```text
failed_precondition: failed precondition: the check of the bucket failed: object storage: put hopbox-check/3kotin4h4ws2b4kzvv2eayp7kg: the signature does not match, so the secret key is wrong (HTTP 403 SignatureDoesNotMatch: The request signature we calculated does not match the signature you provided.). Nothing changed
```

The control server sends the check. A backup runs on the node of its machine, and the node sends its uploads to the bucket directly. So the bucket must take requests from the public address of each server ([Backups](#backups)).

## Requests

| Request | S3 call | What it does |
| --- | --- | --- |
| put | `PUT /<bucket>/<key>` | Writes one object of at most 5 GiB. |
| get | `GET /<bucket>/<key>` | Reads one object. After a cut, it asks for the rest only ([Retries](#retries)). |
| head | `HEAD /<bucket>/<key>` | Reads the size, the ETag, and the time of the last change of one object. A miss before any answer that found the bucket asks for the bucket too, to tell `no_bucket` from `not_found`. After such an answer, the client knows the bucket for its whole life, so a miss costs one request (docket #211). |
| delete | `DELETE /<bucket>/<key>` | Removes one object. A key that does not exist is no error. |
| list | `GET /<bucket>?list-type=2` | Lists the objects of a prefix, 1000 at a time, in the order of their keys (ListObjectsV2). |

Each request uses the path-style address `https://<endpoint>/<bucket>/<key>`. So the server needs no DNS record and no certificate for each bucket (Q227 of the plan of phase 3). SeaweedFS, OVHcloud, Backblaze B2, Cloudflare R2, MinIO, and AWS S3 take this form.

A key is 1 to 1024 bytes of letters, digits, and the characters `!-_.*'()/`. A key has no empty part between two slashes, and no part `.` or `..`.

## Signatures and TLS

Each request carries a signature of AWS Signature Version 4 ([the AWS docs](https://docs.aws.amazon.com/AmazonS3/latest/API/sig-v4-header-based-auth.html)). The signature is in the `Authorization` header. The signature covers the host, the time, and the SHA-256 of the payload. So the server refuses a payload that changed on the way. The URL holds no credential.

The client sends requests only over https, with TLS 1.2 or later. It refuses a plain `http` endpoint, also in a test. It checks the certificate of the server against the roots of the system, and against the CA certificates of the config ([Fields](#fields)). It follows no redirect and uses no proxy.

The clock of the server that sends a request must be within 15 minutes of the clock of the S3 server. Otherwise the S3 server refuses the signature.

## Retries

A request that gets no answer, or an answer that says that the server failed, runs again. Each other answer ends the request at once, such as a wrong key or a missing bucket.

| Setting | Value |
| --- | --- |
| Attempts in a row | 5 |
| Attempts in all | 100. A download that moves bytes at each attempt still ends after them. |
| Wait before a retry | It starts at 250 ms and doubles up to 8 s, with a random part. |
| Timeout of a connect and of a TLS handshake | 10 s each |
| Stall timeout | 30 s. An attempt in which no byte moves for this long ends. This catches a cut link that drops packets without an answer. |

A download that a cut link stops resumes where it stopped. The next attempt asks for the bytes from the first missing byte on, with the header `Range`. It also sends the header `If-Match` with the ETag of the first answer. So a download never joins two versions of an object. A first answer without an ETag allows no resume: the get then fails with the kind `not_s3` at the first cut. An object that changes between two attempts fails the get with the kind `changed`. An attempt that moved bytes starts the count of attempts again, so a link that drops now and then never ends a long download.

A put sends the whole object again at each attempt. A backup sends chunks of 4 MiB, so a retry sends at most one chunk again.

## Errors

Each error starts with `object storage: <request> <key>: ` and then names its cause. The text never holds a credential. The kind of an error tells a caller what to fix:

| Kind | Cause | Text after the key | Retry |
| --- | --- | --- | --- |
| `unreachable` | No answer: nothing listens at the port, a timeout, or a stall | `the endpoint cannot be reached: ...`, such as `nothing listens at the port: ...` | yes |
| `no_host` | The host of the endpoint does not resolve. Such a name is almost always a typo. | `the host of the endpoint does not resolve. Check the endpoint: ...` | no |
| `tls` | The certificate of the server is not trusted | `the certificate of the endpoint is not trusted. Give the CA of the server in the CA certificates of the config, or fix the certificate: ...` | no |
| `not_s3` | The endpoint answers, but not as S3, such as a web server, or a server without TLS | `the endpoint does not answer as an S3 server. Check the endpoint ...` | no |
| `region` | The bucket lives in another region | `the bucket is in another region than the config names. Fix the region ...` | no |
| `access_key` | The server does not know the access key ID | `the endpoint does not know the access key ID ...` | no |
| `secret_key` | The signature does not match | `the signature does not match, so the secret key is wrong ...` | no |
| `denied` | The key has no right to the request | `the access key has no right to this request in the bucket ...` | no |
| `clock` | The clocks differ by more than 15 minutes | `the clock of this server is more than 15 minutes away from the clock of the endpoint. Fix the time sync of this server ...` | no |
| `no_bucket` | The bucket does not exist. A head of an object then asks for the bucket too, because its answer has no body that names the error. | `the bucket does not exist at the endpoint ...` | no |
| `not_found` | The object does not exist | `the object does not exist ...` | no |
| `changed` | The object changed during a download | `the object changed during the download ...` | no |
| `server` | The server failed, or asked for fewer requests | `the server failed or asked for fewer requests ...` | yes |
| `request` | Another refusal of the server | `the server refused the request ...` | no |

An answer can hold an S3 error. Then the text adds its HTTP status, its S3 code, and its message, such as `(HTTP 403 SignatureDoesNotMatch: ...)`.

## Backups

A backup copies the main disk of a machine to the bucket, without its memory and without its extra disks. The calls are `CreateBackup`, `ListBackups`, and `DeleteBackup` ([the machine API](/reference/machine-api/index.html.md#backups)). The code is in `internal/backup` for the format, `internal/node/backups` for the node, and `internal/service/backups.go` for the control role.

### Objects in the bucket

The objects of the backups of one machine share the prefix `backups/<org>/<machine>/`, with the UUIDs of the org and of the machine:

| Key | Object |
| --- | --- |
| `backups/<org>/<machine>/chunks/<name>` | One chunk of the disk, sealed. Each backup of the machine that holds the same bytes names the same chunk. |
| `backups/<org>/<machine>/manifests/<backup>` | The manifest of one backup, sealed. |

A chunk holds 4 MiB of the disk, at an offset that is a whole number of chunks. The last chunk of a disk can be shorter. A region of the disk that holes cover gets no chunk, and so does a chunk of zeros. A restore reads them as zeros. So a disk of 100 GB with little data needs few chunks.

The name of a chunk is the SHA-256 HMAC of its bytes, in 64 lowercase hex digits. The key of the HMAC comes from the data key of the machine. So the name tells nothing of the bytes without the key, and two machines never share a chunk (Q177 of the plan of phase 3).

### Encryption

Each machine has a data key of 32 random bytes. The control role makes it at the first backup of the machine. The key-encryption key of the install seals it, with the place kind `backup-key`, the org, and the machine ([Sealing](/reference/sealing/index.html.md#places)). The column `backup_sets.sealed_key` holds it, and a rewrap of the keyring moves it too. Two keys come from the data key with HKDF-SHA256: the key that names the chunks, and the key that seals the objects.

Each object uses AES-256-GCM. A sealed object starts with 4 magic bytes: `HBC1` for a chunk, or `HBM1` for a manifest. A random nonce of 12 bytes comes next, then the encrypted bytes with their tag of 16 bytes. So a chunk grows by 32 bytes. The associated data holds the format and the kind of the object. It also holds the org, the kind and the ID of the owner of the key, and the name of the object: the chunk name or the backup ID. So the provider cannot swap two chunks, or put the chunk of one machine in the place of another. A chunk that opens must also give its own name again, so a reader checks each chunk.

The manifest is JSON, sealed as a chunk is:

| Field | Description |
| --- | --- |
| `version` | `1` |
| `backup_id`, `org_id`, `machine_id` | The backup, its org, and its machine. |
| `copied_at` | The time of the copy of the disk. |
| `disk_bytes` | The size of the disk. |
| `chunk_bytes` | The size of each chunk but the last: 4194304. |
| `chunks` | The chunks that hold data, in the order of their index. Each one has `i`, its index on the disk, `n`, its name, and `b`, the bytes of the disk in it. |

A lost key-encryption key loses the backups. So keep the keyring apart from the bucket ([Back up machines to a bucket](/guide/backups/index.html.md#keep-the-keyring-safe)).

### The steps of a backup

| State | What happens | Who moves it on |
| --- | --- | --- |
| `pending` | `CreateBackup` or the backup period added the backup. The control role sends it to the node of the machine. | The node, once it made the copy of the disk. |
| `copied` | The node made the copy, and counted its chunks of data. The backup counts at its worst size in the backup usage of its org. | The control role, after the heartbeat that reports the copy. If the worst size passes the backup limit of the org, the backup fails. It waits while a prune holds the set of the machine, and tries again at each heartbeat. |
| `uploading` | The node uploads the copy. | The node, once it wrote the manifest, or once the upload failed for good. |
| `done` | The backup is whole in the bucket. Its uploaded bytes count in the usage. | The prune, once its retention ends or a user deleted it. |
| `failed` | The backup failed, and its error says why. | The prune, as for a done backup. |

The worst size of a backup is each chunk of data of its copy, whole and sealed. It adds 128 bytes of manifest for each chunk, and 4 KiB more. A backup can stall, such as on a node that never came back. After 1 hour without progress, it fails with the error `the node of the machine moved no part of the backup for 1h0m0s`. An upload that waits for a free slot on its node is no stall: each report of its node moves the clock, while the install has object storage. Without object storage, the backup job fails each backup that copies or uploads at once. Its error starts with `the install has no object storage, so the backup cannot upload`, and names the command that sets it.

### The copy and the upload on the node

The node copies the disk through the executor of the machine, between two actions, with a valid lease ([the node executor](/reference/executor/index.html.md#the-copy-of-a-disk)). A running machine pauses only for the reflink copy. The upload runs after the resume, apart from the machine.

Each backup has a dir `<data_dir>/backups/<backup>/` on the file system of the disks:

| File | What it holds |
| --- | --- |
| `disk.raw` | The reflink copy of the disk. The node removes it once the backup ends. |
| `copy.json` | The machine, the org, the time of the copy, the size of the disk, and the count of chunks of data. |
| `chunks.log` | One line for each chunk that the upload went through: its index, its name or `-` for zeros, its bytes, and the bytes that the upload sent for it. |
| `done.json` | The counts of a done upload. |
| `failed.json` | The reason of a failed backup. |

At the start of an upload, the node lists the chunk prefix of the machine once, in pages of 1000. A chunk that the bucket holds goes up no second time, and a new chunk costs one `PUT`, with no `HEAD` (docket #211). A backup of a disk that did not change since the last backup writes only its manifest. The node writes the manifest last, so a backup is done only with every chunk in the bucket.

A restart of `hopboxd` keeps the dir. The next upload goes on from the chunk after the last line of `chunks.log`. A line that a crash cut goes, and its chunk runs again. A chunk that went up without its line is in the list of the bucket, so it does not go up again. A node runs at most 2 uploads at once. A copy that fails runs again after 30 s, and the fifth failure in a row fails the backup. A cut link and other causes that a retry can fix stop the upload. It goes on after 30 s from its last chunk. Any other failure, such as a refused key, fails the backup.

A push holds the data key of a backup only once its upload may start. It holds the bucket credentials only with such a backup. The node holds them in memory, only for the channel that pushed them. At the end of the channel, the uploads stop, and the copies stay for the next channel. A backup that the last push leaves out ends: the node stops its upload, and removes its dir.

### The jobs of the leader

| Job | Every | What it does |
| --- | --- | --- |
| `backups` | 1 minute | Starts the backup of each machine whose backup period passed since its last backup, at most 100 in a run. A check that refuses such a backup fails a new backup at once with the reason, so the next try waits a period. Fails the stalled backups. Reads the manifest of each done backup, at most 20 in a run, and records its chunks. Marks for purge each set whose org is gone, such as after a delete of the org by a release before backups. Purges the sets of the removals of Q189. Counts the overdue machines. A count that fails keeps the last value of the gauge. A machine whose set waits for its purge gets no scheduled backup. |
| `backup_prune` | 10 minutes | Prunes at most 20 sets in a run ([The prune](#the-prune)). |

A run of each job ends after 5 minutes. The jobs run apart from the other jobs of the leader, so a slow bucket holds back no other job. A scheduled backup has the actor `system` in its audit entry. Without object storage, the jobs record and prune nothing, and they still count as a run that worked.

### The prune

The control role records the chunks of each done backup from its manifest, in `backup_chunks`. Each chunk keeps `last_used_at`, the creation time of the newest done backup that names it. A done backup whose manifest is not in the bucket, or does not open, fails.

A prune of a set takes the prune lease of the set for 15 minutes. It takes no lease while an upload of the set runs, or while a done backup waits for the record of its chunks. An upload never starts while a prune holds the lease, by the time of the database. The prune first locks the row of the set, then checks the uploads in a new statement. So it sees an upload that started while it waited for the lock, and a prune never deletes a chunk that an upload counts on. A deleted backup that still copies or uploads holds the prune back for 2 minutes, until its node stopped.

The prune takes these steps:

1. It removes each backup past its retention, and each backup that a user deleted: first its manifest, then its row.
2. It reads the creation time of the oldest done backup that it keeps.
3. It lists the objects of the set. It deletes each chunk whose `last_used_at` is older than that time, because no kept backup can name it. It also deletes each chunk with no row in `backup_chunks`, such as one of a failed upload. It also deletes each manifest of a backup without a row. It keeps an object under the chunk prefix whose key is no chunk name.
4. It records the bytes of each object that stays as the bytes of the set.

A backup that a user deleted between two kept backups frees its own chunks only once each older backup goes. A step that fails leaves the set marked, and the next run starts again. An object that is gone already is no error.

The backup usage of an org is the bytes of its sets, and the worst size of each backup that copies or uploads ([Domain rules](/reference/domain-rules/index.html.md#limits-of-saved-state)). A done backup adds its uploaded bytes at once. The prune counts the bytes of the bucket again, so the count is exact after each prune. Until then, the chunks of a failed upload do not count.

A set of a suspended org waits: the prune skips it, and its retention clock stops ([Domain rules](/reference/domain-rules/index.html.md#org-suspension)).

### Backups after a delete

A delete of a machine by a user keeps its backups until their retention ends (Q189). `ListBackups` with the ID of the machine lists them. A restore of such a backup comes in milestone T22. Other removals take the backups at once: a leave of the creator, a removal of the creator, the delete of the account of the creator, and the delete of the org. A live shared machine keeps its backups. Such a removal drops the sealed data key and the backup rows in its transaction, so no backup of the set opens again. 2 minutes later, the job `backups` deletes each object under the prefix of the set, and then the set.

## Logs

| Message | Level | Fields |
| --- | --- | --- |
| `an object storage request failed, so it runs again` | warn | `component=objectstore`, `op`, `key`, `attempt`, `wait`, and `err` |

The control role writes these records of the backups, with `component=service`, or `component=leader` for the jobs:

| Message | Level | Fields |
| --- | --- | --- |
| `a backup is whole in the bucket` | info | `backup`, `server`. The heartbeat that reports the end. |
| `a backup failed on its node` | warn | `backup`, `server`, `err` |
| `the upload of a backup starts` | info | `backup`, `server` |
| `the backup limit of the org refused the upload of a backup` | warn | `backup`, `server`, `err`. The backup fails. |
| `cannot start the upload of a backup. The next heartbeat tries again` | warn | `backup`, `server`, `err` |
| `cannot start the scheduled backup of a machine. The next run tries again` | warn | `machine`, `err` |
| `a check refused the scheduled backup of a machine, so it fails` | warn | `machine`, `err` |
| `a backup failed, because its node moved no part of it for an hour` | warn | `backup`, `machine`, `state` |
| `cannot fail a stalled backup` | warn | `backup`, `err` |
| `the org of a backup set is gone, so its backups go` | info | `machine` |
| `a backup failed, because the install has no object storage` | warn | `backup`, `machine`, `state` |
| `a done backup has no manifest that opens in the bucket, so it fails` | error | `backup`, `machine`, `err` |
| `pruned the backups of a machine` | info | `machine`, `backups`, `objects` |
| `the prune of the backups of a machine failed. The next run tries again` | warn | `machine`, `err` |
| `cannot release the prune lease of a backup set` | warn | `machine`, `err`. The lease ends after 15 minutes. |
| `removed the backups of a machine for good` | info | `machine`. The purge of a removal of Q189. |
| `the backup jobs failed` | warn | `component=leader`, `err` |
| `the prune of the backups failed` | warn | `component=leader`, `err` |

The node writes these records, with `component=backups` and the fields `backup` and `machine`:

| Message | Level | Fields |
| --- | --- | --- |
| `copied the disk of a backup` | info | `disk_bytes`, `chunks` |
| `the copy of the disk of a backup failed. It runs again` | warn | `err`, `try` |
| `the copy of the disk of a backup failed for good` | error | `err` |
| `the backup is whole in the bucket` | info | `chunks`, `new_chunks`, `uploaded_bytes` |
| `the upload of a backup failed. It goes on from its last chunk at the next try` | warn | `err` |
| `the upload of a backup failed for good` | error | `err` |
| `cannot record the failure of a backup` | warn | `backup`, `err` |
| `the control plane pushed a backup that this node cannot read, so it leaves it out` | warn | `backup`, `err` |
| `the dir of a backup is damaged, so the backup starts again` | warn | `backup`, `err` |
| `cannot remove the dir of a backup that ended` | warn | `backup`, `err` |
| `the backups stopped` | error | `err` |

## The fake server of the tests

The package `internal/objectstore/s3test` is a fake S3 server for unit tests. It serves the requests of the client over TLS, and checks each signature with its own code. A test can queue faults: an error answer, a cut after a number of bytes, or a stall. The local S3 server of the test fleet is SeaweedFS.
