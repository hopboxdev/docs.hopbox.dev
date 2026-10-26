# Add pools and sizes

> How to add a pool for a second server model and a size whose machines land in it, why enrollment refuses a server with other hardware, and how to retire a size and delete a pool.

This guide adds a pool for a second server model, and a size whose machines land only in that pool. A pool is a named group of servers with identical hardware. A size sets the vCPUs, the RAM, and the disk of a machine. An operator size can also name a pool and labels. A user picks a size for a new machine, and never a pool.

[The admin API](/reference/admin-api/index.html.md#poolservice) covers the calls, [hopboxd](/reference/hopboxd/index.html.md#admin-pool-create) the commands, and [hopbox Terraform provider](/reference/terraform-provider/index.html.md#resource-hopbox_pool) the Terraform resources.

## Why pools and sizes

Every install has the pool `default` and the size `default`: 2 vCPUs, 8 GiB of RAM, and a sparse disk of 100 GB. That is enough for an install of one server model. Two rules make a second model need its own pool:

- All servers of a pool have identical hardware. The first server that enrolls into a pool records its CPU model, its CPU threads, its RAM, and its disks. The enrollment of a server with other hardware fails.
- Placement filters use pools and labels. A size that names a pool keeps its machines on the nodes of that pool. A size that names labels keeps them on the nodes with each label.

So a bigger server model goes into its own pool, and a bigger size can keep its machines there.

## What you need

- An install with its first server, as [Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md) sets up.
- Root on a control server, for the commands. Every command below runs with `sudo`.
- For Terraform: the provider with `install_url` and an admin API key ([Install hopbox on an OVHcloud server with Terraform](/guide/install-on-ovhcloud/index.html.md#6-keep-the-admin-api-key)).

## 1. Add the pool

```sh
sudo hopboxd admin pool create big --ovh-plan 24adv01
```

`--ovh-plan` records the OVHcloud plan code of the servers of the pool. Leave it out for a pool of adopted servers. The pool has no hardware until its first server enrolls:

```text
$ sudo hopboxd admin pool list
NAME     OVH PLAN  SERVERS  SIZES  HARDWARE
big      24adv01   0        0      -
default  -         1        0      -
```

## 2. Enroll a server into the pool

Make a token for the pool, and enroll the new server with it ([Enroll the node role on one server](/guide/enroll-node/index.html.md)):

```sh
TOKEN="$(sudo hopboxd admin enrollment-token create --pool big)"
```

The first server of `big` records its hardware. `hopboxd admin pool get big` then shows it, and the log of the control role has the record `recorded the hardware of a pool`. A later server of another model fails, and its token stays unused:

```text
hopboxd: error: enroll at https://10.0.0.1:7443: failed_precondition: failed precondition: the hardware of this server differs from the pool big: it has 16 CPU threads, not 12. All servers of a pool have identical hardware, so enroll it into a pool of its server model
```

RAM and each disk can differ by 2%, because two servers of one model can show a few MiB more or less. The CPU model, the thread count, and the count of disks must be the same.

To put another model in a pool, decommission every server of the pool first. A pool without a live server takes the hardware of its next server. The pool keeps its `ovh_plan`, which then names the old model, and no call changes the plan of a pool. So for another model, prefer a new pool. To keep the name, delete the empty pool and create it again with the new plan, before the first server of the new model enrolls:

```sh
sudo hopboxd admin pool delete big
sudo hopboxd admin pool create big --ovh-plan 24adv02
```

The delete needs a pool without live sizes, so retire the sizes of the pool first. With Terraform, a change of `ovh_plan` of the `hopbox_pool` does the same delete and create.

## 3. Add a size for the pool

```sh
sudo hopboxd admin size create large --vcpus 8 --memory-gib 32 --disk-gb 200 --pool big --label disk=nvme
```

A machine of the size `large` lands only on a node of `big` that has the label `disk=nvme`. Set the labels of a server with `hopboxd admin server set-labels`, or with `labels` of its `hopbox_server`. A user sees the new size, without its pool and labels:

```text
$ hopbox machine sizes
NAME     VCPUS  MEMORY  DISK
default  2      8 GiB   100 GB
large    8      32 GiB  200 GB
```

A machine of the size `large` that finds no node of `big` with room, or no node with the label, fails with `unavailable`. The message names the reason ([Placement](/reference/placement/index.html.md#when-no-node-has-room)):

```text
no capacity: no node has room for a machine of size large (8 vCPUs, 32 GiB of RAM): 1 node is in another pool, 1 node lacks a label
```

## 4. Change or retire a size

A size never changes, because each machine copies the numbers of its size. To change a size, add a size with a new name, and retire the old one:

```sh
sudo hopboxd admin size create large2 --vcpus 16 --memory-gib 64 --disk-gb 400 --pool big
sudo hopboxd admin size retire large
```

A retired size takes no new machine. Its machines keep running and keep their size. Its name stays taken, because those machines name it. `hopboxd admin size list --all` shows the retired sizes too. The size `default` cannot retire, because a create without a size takes it.

## 5. Delete a pool

```sh
sudo hopboxd admin pool delete big
```

The delete frees the name. The install refuses it while the pool holds a server that is not decommissioned, or while a size names it that is not retired. Decommission the servers and retire the sizes first. The pool `default` cannot go.

## The same with Terraform

The provider manages pools and sizes with `hopbox_pool` and `hopbox_size`. Both call the admin API, so the provider block needs `install_url`, and `HOPBOX_ADMIN_API_KEY` in the environment:

```hcl
resource "hopbox_pool" "big" {
  name     = "big"
  ovh_plan = "24adv01"
}

resource "hopbox_size" "large" {
  name       = "large"
  vcpus      = 8
  memory_gib = 32
  disk_gb    = 200
  pool       = hopbox_pool.big.name
  labels     = { disk = "nvme" }
}
```

A `hopbox_server` joins the pool with `pool = hopbox_pool.big.name`. The first server of an install stays in the pool `default`, because no other pool exists before it. A destroy of `hopbox_size` retires the size, and a destroy of `hopbox_pool` deletes the pool. A plan that changes a size and keeps its name fails with `A size never changes`. Give the changed size a new name, and Terraform retires the old one.
