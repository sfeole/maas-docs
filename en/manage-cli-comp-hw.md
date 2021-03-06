Title: CLI Composable Hardware
table_of_contents: True


# CLI pod management

This is a list of examples of pod-management tasks performed with the MAAS CLI.
See [MAAS CLI][manage-cli] for how to get started with the CLI and
[Pods][composable-hardware] for an overview of the subject.


## Add a pod

To add a pod:

```bash
maas $PROFILE pods create type=$POD_TYPE power_address=$POWER_ADDRESS \
	[power_user=$USERNAME] [power_pass=$PASSWORD] [zone=$ZONE] \
	[tags=$TAG1,$TAG2,...]
```

!!! Note:
    Both USERNAME and PASSWORD are optional for the virsh power type.  ZONE and
    TAGS are optional for all pods.

See the [API reference][api-powertypes] for a listing of available power types.

For example, to create an RSD pod:

```bash
maas $PROFILE pods create type=rsd power_address=10.3.0.1:8443 \
	power_user=admin power_pass=admin
```

And to create a KVM host:

```bash
maas $PROFILE pods create type=virsh power_address=qemu+ssh://ubuntu@192.168.1.2/system
```

Create a KVM host with [overcommitted resources][over-commit]:

```bash
maas $PROFILE pods create type=virsh power_address=qemu+ssh://ubuntu@192.168.1.2/system \
        power_pass=example cpu_over_commit_ratio=0.3 memory_over_commit_ratio=4.6
```

Create a KVM host that uses a default [storage pool][storagepools]:

```bash
maas $PROFILE pods create type=virsh power_address=qemu+ssh://ubuntu@192.168.1.2/system \
        power_pass=example default_storage_pool=pool1
```

## Find pod IDs

Here's a simple way to find a pod's ID by name using `jq`:

```bash
maas $PROFILE pods read | jq '.[] | select (.name=="MyPod") | .name, .id'
```
!!! Note:
    [`jq`][jq] is a command-line JSON processor.

Example output:

```no-highlight
"MyPod"
1
```

## List resources of all pods

```bash
maas $PROFILE pods read
```

A portion of sample output:

```no-highlight
        "id": 93,
        "capabilities": [
            "composable",
            "fixed_local_storage",
            "iscsi_storage"
        ],
        "name": "civil-hermit",
```

## List resources of a pod

To list an individual pod's resources:

```bash
maas $PROFILE pod read $POD_ID
```


## Update pod configuration

Update overcommit ratios for a KVM host:

```bash
maas $PROFILE pod update $POD_ID power_address=qemu+ssh://ubuntu@192.168.1.2/system \
        power_pass=example cpu_over_commit_ratio=2.5 memory_over_commit_ratio=10.0
```

Update the default storage pool used by a KVM host:

```bash
maas $PROFILE pod update $POD_ID power_address=qemu+ssh://ubuntu@192.168.1.2/system \
        power_pass=example default_storage_pool=pool2
```


## List pod connection parameters

To list a pod's connection parameters:

```bash
maas $PROFILE pod parameters $POD_ID
```

Example output:

```no-highlight
{
    "power_address": "10.3.0.1:8443",
    "power_pass": "admin",
    "power_user": "admin"
}
```

## Compose pod virtual machines

### Basic

To compose a basic pod VM:

```bash
maas $PROFILE pod compose $POD_ID
```

Example output for default composing:

```no-highlight
{
    "system_id": "73yxmc",
    "resource_uri": "/MAAS/api/2.0/machines/73yxmc/"
}
```

### Set resources

Compose with resources specified:

```bash
maas $PROFILE pod compose $POD_ID $RESOURCES
```

Where RESOURCES is a space-separated list from:

**cores=**requested cores  
**cpu_speed=**requested minimum cpu speed in MHz  
**memory=**requested memory in MB  
**architecture=** See [Architecture][architecture] below  
**storage=** See [Storage][storage] below  
**interfaces=** See [Interfaces][interfaceconstraints] below  

#### Architecture

To list available architectures:

```bash
maas $PROFILE boot-resources read
```

Then, for example:

```bash
maas $PROFILE pod compose $POD_ID \
	cores=40 cpu_speed=2000 memory=7812 architecture="amd64/generic"
```

#### Storage

Storage parameters look like this:

```no-highlight
storage="<label>:<size in GB>(<storage pool name>),<label>:<size in GB>(<storage pool name>)"
```

For example, to compose a machine with the following disks:

- 32 GB disk from storage pool `pool1`
- 64 GB disk from storage pool `pool2`

Where we want the first to be used as a bootable root partition `/` and the
second to be used as a home directory.

First, create the VM:

```bash
maas $PROFILE pod compose $POD_ID "storage=mylabel:32(pool1),mylabel:64(pool2)"
```

Note that the labels, here `mylabel`, are an ephemeral convenience that you
might find useful in scripting MAAS actions.

MAAS will create a pod VM with 2 disks, `/dev/vda` (32 GB) and `/dev/vdb` (64
GB). After MAAS enlists, commissions and acquires the machine, you can edit the
disks before deploying to suit your needs. For example, we'll set a boot, root
and home partition.

We'll start by deleting the `/` partition MAAS created because we want a separate
`/boot` partition to demonstrate how this might be done.

```bash
maas admin partition delete $POD_ID $DISK1_ID $PARTITION_ID
```

!!! Note:
    To find `$DISK1_ID` and `$PARTITION_ID`, use `maas admin machine read
    $POD_ID`.

Now, create a boot partition (~512MB):

```bash
maas admin partitions create $POD_ID $DISK1_ID size=512000000 bootable=True
```

We'll use the remaining space for the root partition, so create another without
specifying size:

```bash
maas admin partitions create $POD_ID $DISK1_ID
```

Finally, create a partition to use as the home directory. Here we'll use the entire
space:

```bash
maas admin partitions create $POD_ID $DISK2_ID
```

!!! Note:
    To find `$DISK2_ID`, use `maas admin machine read $POD_ID`.

Now, format the partitions. This requires three commands:

```bash
maas admin partition format $POD_ID $DISK1_ID $BOOT_PARTITION_ID fstype=ext2
maas admin partition format $POD_ID $DISK1_ID $ROOT_PARTITION_ID fstype=ext4
maas admin partition format $POD_ID $DISK2_ID $HOME_PARTITION_ID fstype=ext4
```

!!! Note:
    To find the partition IDs, use `maas admin partitions read $POD_ID
    $DISK1_ID` and `maas admin partitions read $POD_ID $DISK2_ID`

Before you can deploy the machine with our partition layout, you need to mount
the new partitions. Here, we'll do that in three commands:

```bash
maas admin partition mount $SYSTEM_ID $DISK1_ID $BOOT_PARTITION_ID "mount_point=/boot"
maas admin partition mount $SYSTEM_ID $DISK1_ID $ROOT_PARTITION_ID "mount_point=/"
maas admin partition mount $SYSTEM_ID $DISK2_ID $HOME_PARTITION_ID "mount_point=/home"
```

Finally, we deploy the machine. MAAS will use the partitions as we have defined
them, similar to a normal Ubuntu desktop install:

```bash
maas admin machine deploy $SYSTEM_ID
```

#### Interfaces

Using the `interfaces` constraint, you can compose virtual machines with
interfaces, allowing the selection of pod NICs.

If you don't specify an `interfaces` constraint, MAAS maintains backward
compatibility by checking for a `maas` network, then a `default` network to
which to connect the virtual machine.

If you specify an `interfaces` constraint, MAAS creates a `bridge` or `macvlan`
attachment to the networks that match the given constraint. MAAS prefers `bridge`
interface attachments when possible, since this typically results in successful
communication.


Consider the following interfaces constraint:

```no-highlight
interfaces=eth0:space=maas,eth1:space=storage
```

Assuming the pod is deployed on a machine or controller with access to the
`maas` and `storage` [spaces][spaces], MAAS will create an `eth0` interface
bound to the `maas` space and an `eth1` interface bound to the `storage` space.

Another example tells MAAS to assign unallocated IP addresses:

```no-highlight
interfaces=eth0:ip=172.16.99.42
```

MAAS automatically converts the `ip` constraint to a VLAN constraint (for the
VLAN where its subnet can be found -- e.g. `172.16.99.0/24`.) and assigns the IP
address to the newly-composed machine upon allocation.

See the [MAAS API documentation][api-allocate] for a list of all constraint
keys.

## Compose and allocate a pod VM

In the absence of any nodes in the 'New' or 'Ready' state, if a pod of
sufficient resources is available, MAAS can automatically compose (add),
commission, and acquire a pod VM. This is done with the `allocate` sub-command:

```bash
maas $PROFILE machines allocate
```

Note that all pod [resource parameters][resources] are available to the
`allocate` command, so based on the example above, the following works:

```bash
maas $PROFILE machines allocate "storage=mylabel1:32(pool1),mylabel2:64(pool2)"
```

Once commissioned and acquired, the new machine will be ready to deploy.

!!! Note:
    The labels (i.e. `mylabel1`, `mylabel2`) in this case can be used to
    associate device IDs in the information MAAS dumps about the newly created
    VM. Try piping the output to: `jq '.constraints_by_type'`.

## List machine parameters

MAAS VM parameters, including their resources, are listed just like any other
machine:

```bash
maas $PROFILE machine read $SYSTEM_ID
```

## Libvirt storage pools

### Composing VMs with storage pool constraints

See [Compose pod virtual machines][compose-pod-machines].

### Usage

Retrieve pod storage pool information with the following command:

```bash
maas $PROFILE pod read $POD_ID
```

Example:

```no-highlight
Success.
Machine-readable output follows:
{
    "used": {
        "cores": 50,
        "memory": 31744,
        "local_storage": 63110426112
    },
    "name": "more-toad",
    "id": 5,
    "available": {
        "cores": 5,
        "memory": 4096,
        "local_storage": 153199988295
    },
    "architectures": [],
    "cpu_over_commit_ratio": 1.0,
    "storage_pools": [
        {
            "id": "pool_id-zvPk9C",
            "name": "name-m0M4ZR",
            "type": "lvm",
            "path": "/var/lib/name-m0M4ZR",
            "total": 47222731890,
            "used": 17226931712,
            "available": 29995800178,
            "default": true
        },
        {
            "id": "pool_id-qF87Ps",
            "name": "name-ZMaIta",
            "type": "lvm",
            "path": "/var/lib/name-ZMaIta",
            "total": 98566956569,
            "used": 15466229760,
            "available": 83100726809,
            "default": false
        },
        {
            "id": "pool_id-a6lyw5",
            "name": "name-RmDPfs",
            "type": "lvm",
            "path": "/var/lib/name-RmDPfs",
            "total": 70520725948,
            "used": 30417264640,
            "available": 40103461308,
            "default": false
        }
    ],
    "total": {
        "cores": 55,
        "memory": 35840,
        "local_storage": 216310414407
    },
    "tags": [],
    "type": "virsh",
    "memory_over_commit_ratio": 1.0,
    "pool": {
        "name": "default",
        "description": "Default pool",
        "id": 0,
        "resource_uri": "/MAAS/api/2.0/resourcepool/0/"
    },
    "zone": {
        "name": "default",
        "description": "",
        "id": 1,
        "resource_uri": "/MAAS/api/2.0/zones/default/"
    },
    "capabilities": [
        "dynamic_local_storage",
        "composable"
    ],
    "host": {
        "system_id": null,
        "__incomplete__": true
    },
    "default_macvlan_mode": null,
    "resource_uri": "/MAAS/api/2.0/pods/5/"
}
```

## Delete a pod VM

```bash
maas $PROFILE machine delete $SYSTEM_ID
```

After a machine is deleted, the machine's resources will be available for other
VMs.

## Delete a pod

```bash
maas $PROFILE pod delete $POD_ID
```

!!! Warning:
    Deleting a pod will automatically delete all machines belonging to that pod.

<!-- LINKS -->

[jq]: https://stedolan.github.io/jq/
[api-powertypes]: api.md#power-types
[spaces]: intro-concepts.md#spaces
[resources]: #set-resources
[storage]: #storage
[architecture]: #architecture
[interfaceconstraints]: #interfaces
[compose-pod-machines]: #compose-pod-virtual-machines
[api-allocate]: api.md#post-maasapi20machines-opallocate
[manage-cli]: manage-cli.md
[composable-hardware]: manage-pods-intro.md
[storagepools]: manage-pods-webui.md#configuration
[over-commit]: manage-kvm-pods-webui.md#overcommit-resources
