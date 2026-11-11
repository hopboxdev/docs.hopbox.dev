# Object storage

> The S3 client of hopbox: its requests, signatures, TLS, retries that resume a download, errors, logs, and the fake server of the tests.

Object storage is a network service that stores files, called objects, in buckets. hopbox uses a bucket of an S3 server for off-site disk backups. Object storage is optional. Without it, every other feature works.

The code is in `internal/objectstore`. It is a small S3 client with only the requests that hopbox needs, and no SDK.

## The config of an install

An instance admin sets the bucket of the install through the admin API: `sudo hopboxd admin config object-storage set` on a control server ([hopboxd](/reference/hopboxd/index.html.md#admin-config-object-storage-set)), the call `SetObjectStorageConfig` ([the admin API](/reference/admin-api/index.html.md#setobjectstorageconfig)), or the page Config of the Admin area ([Console](/reference/console/index.html.md#admin-area)). The config lives in the row of `install_config` ([Control store tables](/reference/control-store/index.html.md#install_config)). A change adds 1 to the version of the install config.

Without object storage, a backup fails with `failed_precondition` and the message `this install has no object storage, so it makes no backups. An instance admin sets it with hopboxd admin config object-storage set, or on the page Config of the Admin area`. Every other feature works.

[Give an install a bucket for backups](/guide/backups/index.html.md) sets up a bucket.

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

The control role opens the secret key for each check, and keeps it in memory only for that check. For a backup, the control role will open the key and send it to the node on the control channel. The node will hold it in memory for that one job, and never log it. A rotation of the key-encryption key leaves the secret key sealed by the old key, until a rewrap moves it ([Sealing](/reference/sealing/index.html.md#rewrap)).

### The check

A new config takes effect only after a check (Q226 of the plan of phase 3). The control process that takes the call writes one object of 64 random bytes under the key `hopbox-check/<random>`, reads it back, compares the bytes, and deletes it. So the check proves the endpoint, the certificate, the region, the bucket, both keys, and the rights to write, read, and delete. A failed check changes nothing. `CheckObjectStorage` runs the same check on the saved config at any time ([the admin API](/reference/admin-api/index.html.md#checkobjectstorage)).

The write and the read of the check take at most 20 s, and the delete at most 5 s more, so a check takes at most 25 s. The delete has its own time, so a read that stalled to the end of its 20 s leaves no object in the bucket. Each of its requests has at most 3 attempts, a stall timeout of 10 s, and a connect timeout of 5 s. A check that gets no answer, or an answer that says that the server failed, fails with `unavailable`. Each other failure is `failed_precondition`, because the config must change first. A host that does not resolve is `failed_precondition` too, because such a name is almost always a typo. A closed port and a timeout stay `unavailable`. The message holds the error of the request ([Errors](#errors)), such as:

```text
failed_precondition: failed precondition: the check of the bucket failed: object storage: put hopbox-check/3kotin4h4ws2b4kzvv2eayp7kg: the signature does not match, so the secret key is wrong (HTTP 403 SignatureDoesNotMatch: The request signature we calculated does not match the signature you provided.). Nothing changed
```

The control server sends the check. A backup runs on the node of its machine, and the node sends its uploads to the bucket directly. So the bucket must take requests from the public address of each server. Backups are planned, and not built yet.

## Requests

| Request | S3 call | What it does |
| --- | --- | --- |
| put | `PUT /<bucket>/<key>` | Writes one object of at most 5 GiB. |
| get | `GET /<bucket>/<key>` | Reads one object. After a cut, it asks for the rest only ([Retries](#retries)). |
| head | `HEAD /<bucket>/<key>` | Reads the size, the ETag, and the time of the last change of one object. |
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

## Logs

| Message | Level | Fields |
| --- | --- | --- |
| `an object storage request failed, so it runs again` | warn | `component=objectstore`, `op`, `key`, `attempt`, `wait`, and `err` |

## The fake server of the tests

The package `internal/objectstore/s3test` is a fake S3 server for unit tests. It serves the requests of the client over TLS, and checks each signature with its own code. A test can queue faults: an error answer, a cut after a number of bytes, or a stall. The local S3 server of the test fleet is SeaweedFS.
