# Back up machines to a bucket

> How an operator gives an install an S3 bucket for backups, and how a user backs up a machine: the bucket and its keys, the lifecycle rule for old versions of deleted objects, the set and the check of the bucket, a private CA, what each failure means, the backup of the keyring, a backup by hand, a backup schedule, and the backups of a deleted machine.

A backup is a copy of the main disk of a machine in a bucket of an S3 server, such as OVHcloud Object Storage. A bucket is a named store of files, called objects, at an object storage provider. Backups need the bucket. Object storage is optional: without it, every other feature of hopbox works.

The first part of this guide is for an instance admin: it connects the install to a bucket, and checks it. The second part is for each user: it backs up a machine, by hand and on a schedule ([Back up a machine](#back-up-a-machine)). [Object storage](/reference/object-storage/index.html.md) holds every rule and error.

## What you need

- An install with a key-encryption key. `hopboxd install` makes the first one. On a dev run, run `sudo hopboxd admin kek rotate` first.
- A shell as root on a control server, or an instance admin account for the Admin area of the console.
- An account at an S3 provider. Examples are OVHcloud, Backblaze B2, Cloudflare R2, and AWS S3. A self-hosted server works too, such as SeaweedFS or MinIO.

## Make the bucket and its keys

Do these steps at the provider:

1. Make a bucket for this install only, such as `bucket1`. Note its region, such as `gra`, and its S3 endpoint, such as `https://s3.gra.io.cloud.ovh.net`.
2. Make an access key for hopbox. Give it the rights to write, read, list, and delete objects in this bucket, and no right to other buckets.
3. Keep the access key ID and the secret key. The provider shows the secret key once.
4. Read the price of storage and of outbound traffic for this region and class of the bucket.

The bucket must take requests from the public address of each server of the install. A backup goes from the node of its machine straight to the bucket, which Q185 of the plan of phase 3 decided.

## Old versions of deleted objects

Some buckets keep the old versions of an object, with a feature called versioning. On such a bucket, a delete removes nothing. It adds a delete marker, and each old version stays and costs money. hopbox deletes the objects of each backup past its retention. Without a rule, the bucket then grows without end.

So if versioning is on, add a lifecycle rule to the bucket. The rule removes each old version one day after the delete, and each delete marker without versions. It also ends each upload that stopped halfway after one day. With the AWS CLI, which also works with other S3 providers, the rule is:

```json
{
  "Rules": [
    {
      "ID": "hopbox-drop-old-versions",
      "Status": "Enabled",
      "Filter": {},
      "NoncurrentVersionExpiration": { "NoncurrentDays": 1 },
      "Expiration": { "ExpiredObjectDeleteMarker": true },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 1 }
    }
  ]
}
```

```sh
aws s3api put-bucket-lifecycle-configuration --endpoint-url https://s3.gra.io.cloud.ovh.net \
  --bucket bucket1 --lifecycle-configuration file://lifecycle.json
```

If your provider has a console page for lifecycle rules, set the same three parts there. A bucket without versioning needs no rule for old versions.

## Set the bucket

On a control server, run this command. It asks for the secret key, and does not show what you type:

```sh
sudo hopboxd admin config object-storage set --endpoint https://s3.gra.io.cloud.ovh.net \
  --region gra --bucket bucket1 --access-key-id 0123456789abcdef --secret-key-stdin
```

The command first checks the bucket. The control server writes one small object under `hopbox-check/`, reads it back, and deletes it. If the check fails, nothing changes, and the message names the cause. Then hopbox seals the secret key and saves the config. The output shows the config without the secret key.

In the console, open the page Config of the Admin area, fill in the section Object storage, and click "Save the object storage". The page runs the same check ([Use the Admin area](/guide/admin-area/index.html.md#set-the-bucket-of-the-backups)).

The command reads the secret key from standard input. No flag takes the secret key, because the process list shows each command line. A script can pipe the key in, such as from the env var `SECRET_KEY`:

```sh
printf '%s\n' "$SECRET_KEY" | sudo hopboxd admin config object-storage set --endpoint https://s3.gra.io.cloud.ovh.net \
  --region gra --bucket bucket1 --access-key-id 0123456789abcdef --secret-key-stdin
```

Each change of the object storage sends a notice to the other instance admins, in the console and by email. A stolen session can point the backups at a bucket of an attacker, so read each such notice.

## An S3 server with a private CA

A self-hosted S3 server can have a certificate from a private CA. Give hopbox that CA in a PEM file:

```sh
sudo hopboxd admin config object-storage set --endpoint https://s3.example.org:8443 \
  --region us-east-1 --bucket bucket1 --access-key-id 0123456789abcdef --secret-key-stdin --ca-file /etc/hopbox/s3-ca.pem
```

The command sends the content of the file. A later `set` without `--ca-file` keeps the certificates, and `--clear-ca` removes them. hopbox refuses a plain `http` endpoint, because each request carries the credentials and the data.

## Check the bucket later

Run the check again at any time, for example after a change at the provider:

```sh
sudo hopboxd admin config object-storage check
```

```text
The bucket bucket1 at https://s3.gra.io.cloud.ovh.net works: a write, a read, and a delete took 184ms.
```

## What a failure means

| Part of the message | What to do |
| --- | --- |
| `the endpoint cannot be reached` | Check the port of the endpoint, and the firewall between the control server and the provider. With `unavailable`, try again later. |
| `the host of the endpoint does not resolve` | Fix the host name of the endpoint, such as a typo. |
| `the certificate of the endpoint is not trusted` | Fix the certificate of the server, or give its CA with `--ca-file`. |
| `the endpoint does not answer as an S3 server` | The URL leads to another kind of server. Check the endpoint at the provider. |
| `the bucket is in another region than the config names` | Set the region of the bucket with `--region`. |
| `the endpoint does not know the access key ID` | Check the access key ID, or make a new key. |
| `the signature does not match, so the secret key is wrong` | Set the secret key again with `--secret-key-stdin`. |
| `the access key has no right to this request in the bucket` | Give the key the rights to write, read, list, and delete objects in the bucket. |
| `the bucket does not exist at the endpoint` | Check the name of the bucket, or make it. |
| `the clock of this server is more than 15 minutes away` | Fix the time sync of the control server, such as with `systemd-timesyncd`. |

## Keep the keyring safe

hopbox encrypts each backup with a data key that the key-encryption key of the install seals ([Object storage](/reference/object-storage/index.html.md#encryption)). If you lose the keyring, you lose the backups too. So back up the keyring dir of each control server, apart from the bucket. [Sealing](/reference/sealing/index.html.md#key-files) names the dir.

## Remove the bucket

To stop the use of the bucket, run:

```sh
sudo hopboxd admin config object-storage clear
```

The bucket and its objects stay at the provider. Delete them there.

## Back up a machine

A backup copies the main disk of the machine, such as `box1`. It holds no memory and no extra disk. A running machine pauses for a moment while its node makes a copy of the disk. The upload of the copy runs after the resume, so the machine runs on during the upload. Each upload sends only the parts of the disk that no earlier backup of the machine holds. A part of the disk that holds only zeros goes nowhere. So a disk of 100 GB with little data makes a small backup.

To back up `box1` and wait up to 10 minutes for the end:

```sh
hopbox machine backup create box1 --wait 10m
```

```text
The backup 01a1d722-d8ed-7a6e-b058-365cdfc60315 of box1 is done: 12.0 MiB of data, 12.0 MiB sent.
```

Without `--wait`, the command returns at once, and the backup goes on. A backup needs the rights to open a shell in the machine. So an org admin cannot back up a private machine of another member ([the machine API](/reference/machine-api/index.html.md#who-can-do-what)).

## List and delete backups

```sh
hopbox machine backup list box1
```

```text
ID                                    CREATED               KIND    STATE  DATA      SENT      EXPIRES
01a1d722-d8ed-7a6e-b058-365cdfc60315  2026-11-13T10:00:00Z  manual  done   12.0 MiB  12.0 MiB  2026-11-20T10:00:00Z
```

The column `SENT` holds the bytes that each backup added to the bucket. A second backup of a disk that did not change sends only a small list of its parts, called the manifest. The column `EXPIRES` is the end of the retention of the backup. After it, hopbox deletes the backup from the bucket.

To delete a backup before the end of its retention:

```sh
hopbox machine backup delete 01a1d722-d8ed-7a6e-b058-365cdfc60315
```

The backup leaves the list at once. Its objects leave the bucket within 10 minutes.

## Back up on a schedule

A machine has no backup schedule by default. To back up `box1` every hour, and keep each backup for 7 days:

```sh
hopbox machine update box1 --backup-period 1h --backup-retention 7d
```

The shortest period is 15 minutes. The retention is 1 hour to 365 days, and the default is 7 days. A new retention also changes the end of the retention of each backup of the machine. To turn the schedule off, run `hopbox machine update box1 --backup-period off`. `hopbox machine show box1` shows the schedule in the row `backups`.

The backups of an org count in its limit `backup_bytes` ([the org API](/reference/org-api/index.html.md#usage-limits)). Read the usage with `hopbox org usage`. A backup above the limit fails, and its error names the limit.

## Backups of a deleted machine

A delete of a machine keeps its backups until the end of their retention. List them with the UUID of the machine, such as `hopbox machine backup list 01a1d716-8aad-7a87-af38-80506cd5787b`. A restore of such a backup comes in a later release. When the creator of a machine leaves the org, or the org or the account goes, hopbox removes the backups of its machines at once. A shared machine that stays in the org keeps its backups.

While an instance admin suspends the org, its backups stay, also past their retention. A delete of a backup then fails. After the suspension, each backup keeps the retention that it had left.

## When a backup fails

`hopbox machine backup list box1` shows a failed backup with the state `failed`. The JSON output, with `--json`, holds its error. These are common errors:

| Error | What to do |
| --- | --- |
| `this install has no object storage, so it makes no backups` | An instance admin sets the bucket ([Set the bucket](#set-the-bucket)). |
| `the machine box1 is pending. A backup needs a machine that is running, in standby, hibernated, or stopped` | Wait until the machine runs or stops. |
| `usage limit exceeded: the backups of the org would use ...` | Delete old backups, set a shorter retention, or ask an instance admin for a higher limit. |
| `the node of the machine moved no part of the backup for 1h0m0s` | The node of the machine cannot reach the bucket, or is down. An instance admin checks the node and the bucket ([What a failure means](#what-a-failure-means)). |
| `the upload failed: ...` | The rest names the error of the bucket, as in [What a failure means](#what-a-failure-means). |
