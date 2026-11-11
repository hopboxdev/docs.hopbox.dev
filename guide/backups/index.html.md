# Give an install a bucket for backups

> How an operator gives an install an S3 bucket for backups: the bucket and its keys, the lifecycle rule for old versions of deleted objects, the set and the check of the bucket, a private CA, what each failure means, and the backup of the keyring.

This guide connects an install to a bucket of an S3 server, such as OVHcloud Object Storage. A bucket is a named store of files, called objects, at an object storage provider. Backups need the bucket. Object storage is optional: without it, every other feature of hopbox works.

Backups themselves come in a later release. This guide sets the bucket and checks it, so the install is ready for them. [Object storage](/reference/object-storage/index.html.md) holds every rule and error.

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

hopbox will encrypt each backup with a data key that the key-encryption key of the install seals. If you lose the keyring, you lose the backups too. So back up the keyring dir of each control server, apart from the bucket. [Sealing](/reference/sealing/index.html.md#key-files) names the dir.

## Remove the bucket

To stop the use of the bucket, run:

```sh
sudo hopboxd admin config object-storage clear
```

The bucket and its objects stay at the provider. Delete them there.
