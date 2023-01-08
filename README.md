# ansible-role-fstab_mounts

This Ansible role was created by me in order to easier manage a couple of
different programs who all need to create entries in `/etc/fstab`, and are
therefore somewhat dependent on each other. This role can create the following
types of `fstab` mount entries:

- Boot mounts
- Normal mounts
- Encrypted mounts
  - With options for loading the decryption keys from a remote host.
- ZFS legacy mounts
- Pooled mounts (using [mergerfs][1] for pooling).


### Important Note
This Ansible role is not really "automatic", since it is necessary to perform
a lot of manual steps before this role can be run (like creating a filesystem
on the drives which are to be mounted). So this is designed more as a method of
"record keeping" which drives that are present on each system, and where they
are mounted. By version managing your configs you can easily track the hardware
changes made to your systems.

At the end of this README there are some guides for the manual steps that are
necessary, so you may know what is going on at a more detailed level.



# Installation

This repository does not have any dependencies by itself, so just move into
your `roles/` folder and run the following:

```bash
git clone git@github.com:JonasAlfredsson/ansible-role-fstab_mounts.git fstab_mounts
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: all
  name: Configure all fstab mounts
  roles:
    - fstab_mounts
```



# Usage

There are five "types" of mounts which can be managed with this role, and they
will all have their own section that explains how to properly configure them.
However, the only section that is required to be completed is the
"[boot mounts](#boot-mounts)" one, since otherwise you may end up in a state
where the computer won't be able to boot properly.

Furthermore, as stated in the introduction, this role requires you to manually
prepare the disks (formatting/encrypting them) so that you can obtain their
unique identifier ([UUID](#obtain-the-uuid)) and enter it here in the Ansible
configuration. If you need help creating these drives there are some nice
step-by-step guides available in the [Preparation Steps](#preparation-steps)
section.

If you know what you are looking for you can jump directly the the desired
section, otherwise just continue reading.

- [Boot Mounts](#boot-mounts)
- [Normal Mounts](#normal-mounts)
- [Encrypted Mounts](#encrypted-mounts)
  - [The `cryptdisks` Mount](#the-cryptdisks-mount)
- [ZFS Mounts](#zfs-mounts)
- [Pooled Mounts](#pooled-mounts)

> NOTE: Some extra information about Ansible variables may be found
        [here](#ansible-variables-location).


## Boot Mounts

Before doing anything else to the system you are trying to apply this role to
we need to define the `mounts_boot` variable. SSH to the destination server and
type the following:

```bash
cat /etc/fstab
```

This is necessary since we need to find the entries which were there since the
installation of the system. For me it looked like this:

```
#   <file system>                       <mount point> <type> <options>          <dump> <pass>
# / was on /dev/sda1 during installation
UUID=88f0236b-620a-456e-a4a6-1b5c84996b5c  /           ext4   errors=remount-ro    0      1
# swap was on /dev/sda5 during installation
UUID=b8d210eb-d186-405f-bc42-a55cbe069a27  none        swap   sw                   0      0
```

This means we need to create the following Ansible variable before creating
entries for any other mounts:

```yaml
mounts_boot:
  - uuid: "88f0236b-620a-456e-a4a6-1b5c84996b5c"
    mount_point: "/"
    type: "ext4"
    options: "errors=remount-ro"
    dump: 0
    pass: 1
  - uuid: "b8d210eb-d186-405f-bc42-a55cbe069a27"
    mount_point: "none"
    type: "swap"
    options: "sw"
    dump: 0
    pass: 0
```

This is the minimal configuration necessary in order to use this role, and here
all the individual variables needs to be explicitly set.

> If your `fstab` file is located somewhere else you can also define the
  `mounts_fstab_path` variable to point to the correct location. Look in the
  [`defaults/main.yml`](./defaults/main.yml) file for more "global" variables.


## Normal Mounts

After you have added any "boot" mounts, you may then add additional "normal"
mounts to `fstab`. The procedure is very similar to the previous
"[Boot Mounts](#boot-mounts)" step, except there are now some default values
present, which means you can write a more compact list if you want. Any field
that is not marked with `# Required` may be left out of your configuration if
you are fine with the defaults.

```yaml
mounts_normal:
  - uuid:  # Required
    mount_point:  # Required
    type: "ext4"
    options: "defaults,nofail"
    dump: 0
    pass: 2
    comment: ""
```

> Help for obtaining the UUID may be found [here](#obtain-the-uuid).


## Encrypted Mounts

This role can also handle mounting encrypted partitions/drives as well. The
configuration options are a bit different from the previous
["normal" mounts](#normal-mounts), and this is because the devices will first
have to be "unlocked" by [`cryptsetup`][10] before it can be mounted as usual
([simple visualization](#mount-it-1)).

Here are the options for encrypted mounts, where the defaults are entered and
any field which is not marked with `# Required` may be left out of your
configuration if you are fine with the defaults. The `name` field defaults to
the `uuid` unless something is specifically defined, but if you add something
custom you need to remember that it will be this device's decrypted endpoint
under `/dev/mapper/` so it needs to be unique.

Furthermore, the `mount_point` field is only required in case you want the
decrypted `/dev/mapper/` endpoint mounted somewhere else. For example, the
entire `fstab` section should be omitted in the case you are using ZFS with
these mapper points as the [identifiers](#choose-a-good-disk-identifier).

```yaml
mounts_encrypted:
  - uuid:  # Required
    name: "{{ uuid }}"
    key_file:  # Required
    crypttab:
      options: "luks,nofail"
      comment: ""
    fstab:
      mount_point:  # Only required if you want the decrypted mapper point mounted
      type: "ext4"
      options: "defaults,nofail"
      dump: 0
      pass: 2
      comment: ""
```

### The Cryptdisks Mount
If the `key_file`(s), in the settings above, is located on a USB key or network
drive, which needs to be mounted **before** anything else can be mounted, you
will have to specify the `CRYPTDISKS_MOUNT` within the `/etc/default/cryptdisks`
file. This can be done by just writing this:

```yaml
mounts_cryptdisks:
  mount_point: "/mnt/usb/drive"
  uuid: "37033980-8380-476d-a851-840815c328b1"
  type: "ext4"
  options: "defaults,noexec"
  dump: 0
  pass: 2
```

or if it is on a Samba network drive which needs credentials:

```yaml
mounts_cryptdisks:
  mount_point: "/mnt/network/drive"
  net_path: "//192.168.0.1/secret_keys"
  net_credentials:
    username: "user"
    password: "user_smb_pass"
    path: "/root/.smbcredentials"
  type: "cifs"
  options: "vers=3.11,uid=root,gid=root,_netdev,noexec"
  dump: 0
  pass: 0
```


## ZFS Mounts

ZFS is a feature rich filesystem which comes with its own mounting service. It
is recommended to let that service be responsible for managing the mounting
of any ZFS datasets instead of `fstab`, so in order to tell this role that you
only want ZFS installed and then let this mounting service be responsible for
the rest is easily achieved by the following configuration:

```yaml
mounts_zfs: {}
```

However, if you want to keep everything centralized through this role it is
possible to set the `legacy` mounting option on the ZFS dataset which would
let `fstab` be responsible for its mounting. Read more about how to activate
`legacy` mounting for each relevant dataset in the
[Create ZFS Array](#create-zfs-array) section, but after that it should be
as simple as this to configure the mount point. Any field which is not marked
with `# Required` may be left out of your configuration if you are fine with
the defaults.

> The `dataset` value here start with the pool and then the dataset's path, e.g.
  `pool1/datasets/data1`.

```yaml
mounts_zfs:
  - dataset:  # Required
    mount_point:  # Required
    type: "zfs"
    options: "defaults,x-systemd.requires=zfs-mount.service"
    dump: 0
    pass: 0
    comment: ""
```

You most likely want the [`x-systemd.requires=zfs-mount.service`][26] option on
all ZFS mounts, as else `fstab` might try to mount datasets before the ZFS
service is ready to handle it and will thus fail with errors. Also you might
want to know about [`x-systemd.requires-mounts-for`][38] that is
[automatically added][37] during boot by the [`systemd.generator`][40] in case
a mount point is beneath another one in the filesystem hierarchy, but this is
also something you can specify yourself if you want
([more systemd options][39]).


## Pooled Mounts

"Pooling" drives means that you make multiple separate disks look like a single
one for the computer. Important to know that "pooling" is **not** RAID, since
this method basically only places single whole files on the different drives
available in the pool with no redundancy.

> Take a look at my [SnapRAID role][21] if you are interested in adding some
  data redundancy to pooled drives.

The program making this possible is [`mergerfs`][1], and first of all you will
need to define which version of it you want to be installed. Choose one from
the list [here][17], and define it like this:

```yaml
mounts_mergerfs_version: "2.29.0"
```

There are a ton of [options][18] available for `mergerfs`, so I suggest you
read up on them for best experience. However, I have supplied some sane
defaults here that most people could live with. Any field that is not marked
with `# Required` may be left out of your configuration if you are fine with
the defaults.

```yaml
mounts_pooled:
  - name: ""
    options: "defaults,allow_other,use_ino"
    dump: 0
    pass: 0
    mount_point:  # Required
    branches: []  # Required
```

The `branches` variable is a list of strings that correspond to mount points
that have been defined in either the [normal](#normal-mounts) or the
[encrypted](#encrypted-mounts) mounts sections.

#### Example:
```yaml
mounts_pooled:
  - name: "first_pool"
    mount_point: "/mnt/pool"
    branches:
      - "/mnt/disk1"
      - "/mnt/disk2"
      - "/mnt/disk3"
```



# Preparation Steps

In this section you will find step-by-step guides on how to prepare the raw
drives in order for them to be usable by the different mounting options
supported by this role.

These steps might not be necessary for you to do, or you have another preferred
way of doing it, which is why these steps should be considered more of general
suggestions on how to do these things. If you want more advanced options you
will need to do some extra research by yourself.

- [Create Normal Drive](#create-normal-drive)
  - [Create Partition Table](#create-partition-table)
  - [Create Partition](#create-partition)
  - [Create Filesystem](#create-filesystem)
  - [Mount It](#mount-it)
- [Create Encrypted Drive](#create-encrypted-drive)
  - [Create a Keyfile](#create-a-keyfile)
  - [Encrypt Device/Partition](#encrypt-devicepartition)
  - [Unlock Encrypted Device](#unlock-encrypted-device)
  - [Create Normal Filesystem](#create-normal-filesystem)
  - [Mount It](#mount-it-1)
- [Create ZFS Array](#create-zfs-array)
  - [Choose a Good Disk Identifier](#choose-a-good-disk-identifier)
  - [Create a Pool](#create-a-pool)
    - [Dataset Properties](#dataset-properties)
    - [The Mountpoint Property](#the-mountpoint-property)
    - [Creating the Pool](#creating-the-pool)
  - [Create Dataset](#create-dataset)
  - [Mount It](#mount-it-2)
    - [Import Pool](#import-pool)
  - [Maintenance](#maintenance)
    - [Some Everyday Commands](#some-everyday-commands)
    - [Health Monitoring](#health-monitoring)
    - [Scrubbing](#scrubbing)
    - [Trimming](#trimming)
    - [Snapshotting](#snapshotting)
  - [Replace a disk](#replace-a-disk)


## Create Normal Drive

1. [Create Partition Table](#create-partition-table) - Only once per drive.
2. [Create Partition](#create-partition) - As many partitions as you want.
3. [Create Filesystem](#create-filesystem) - Once per partition on the drive.
4. [Mount It](#mount-it) - Once per partition on the drive.

In order to be able to perform these steps you will need to find the physical
device you want to use located under `/dev/`. We will then launch [`parted`][9],
which can create both partition tables as well as partitions. So begin with
issuing the starting command:

```bash
sudo parted -a optimal /dev/sdX
```

You should now be inside the "`parted`" command prompt, so any commands made
here will be preceded by "`(parted)`" to indicate that we are still there.

> If you need to completely wipe the drive beforehand you could run the
  following command: \
  `sudo dd if=/dev/zero of=/dev/sdX bs=4M status=progress`

### Create Partition Table
A drive needs a partition table before any partitions can be made. This will
only have to be done once per drive, regardless how many partitions you intend
to have on the drive later.

Unless you intend to install the drive in a computer that is >15 years old you
should use the "`gpt`" option, otherwise you may use "`msdos`" which will
install a [Master Boot Record][6] instead of the [GUID Partition Table][7].

```bash
(parted) mklabel gpt
```

### Create Partition
Now it is time to actually create a partition. It will be much easier if you
already know how many partitions you want on each drive, so you don't have to
fiddle around with this when there is important data on the drive.

In the first example I will just create a single partition which spans the
entire drive. It is [good practice][8] to leave a little room in the beginning
and the end of the drive for alignment purposes.

```bash
(parted) mkpart primary 1 -1
```

Or you can create two partitions.

```bash
(parted) mkpart primary 1 50%
(parted) mkpart primary 50% -1
```

> The "primary" name has no special meaning for GPT, but
  [is important in the case you are using MBR][11].

It is then possible to check so that the partitions are aligned before
continuing. Here I test both the first and the second partition.

```bash
(parted) align-check optimal 1
(parted) align-check optimal 2
```

In both these cases it should return "`# aligned`".

You may now quit `parted`.

```bash
(parted) quit
```

### Create Filesystem
Now you can create a filesystem on the partition. If you are using Linux you
most likely want to go with the `ext4` filesystem. Here we begin with a
bog standard method of doing it for the first partition.

```bash
sudo mkfs.ext4 /dev/sdX1
```

But for the second partition we don't want to [reserve][20] any of the space for
the super user, and we also want to reduce the number of [inodes][19] to only 1
per 4MiB.

```bash
sudo mkfs.ext4 -m 0 -T largefile4 /dev/sdX2
```

These are advanced options that you should read more into before using them, but
as a quick summary the `largefile4` option frees up some space on the drive if
you only intend to store few very large files (like a movie collection). The
`-m` option change the limit of how much a "normal" user may fill the disk, so
without this the last 5% of the drive would only be usable by "root". These
two options should remain untouched on the OS drive, otherwise you may lock up
the system.

### Mount It
To test if you can use the newly created filesystem you may now mount it.

```bash
sudo mount /dev/sdX1 /mnt/first-drive
```

Try going into the desired mount point and create a file, just to be certain
everything works as intended.


## Create Encrypted Drive

You can either target a raw drive, and make it use all the space just like a
single big encrypted partition, or you can target an already existing partition
(that was formed according to the steps in
[Create Normal Drive](#create-normal-drive)) which allows you to mix both
un-encrypted and encrypted partitions on the same drive. Choose the one you
feel best suite your usecase and then continue with this guide.

1. [Create a Keyfile](#create-a-keyfile) - One for each encrypted device.
2. [Encrypt Device/Partition](#encrypt-devicepartition) - As many encrypted devices you want.
3. [Unlock Encrypted Device](#unlock-encrypted-device) - Once per encrypted device.
    - (Optional) [Securely Format a Drive](#securely-format-a-drive) - Once per encrypted device.
4. [Create Normal Filesystem](#create-normal-filesystem) - Once per encrypted device.
5. [Mount It](#mount-it-1) - Once per encrypted device.

> For an enjoyable experience it is important that your system supports
  [hardware accelerated encryption](#test-for-hardware-acceleration) (AES-NI).


### Create a Keyfile
In order to encrypt/decrypt a drive we will first need a password or a key. I
suggest you use a key, since that is more secure, and it can be easily generated
from the following command:

```bash
dd if=/dev/urandom bs=512 count=1 > /path/to/first-drive.key
```

The default max size of a key file is 8 KiB, but there is no point in using a
key which is larger than LUKS's "master key" that is the one being unlocked by
the key file. The maximum master key size for LUKS is 512 bits, but it defaults
to [256 bits][13].

### Encrypt Device/Partition
Now we can encrypt the entire block device

```bash
sudo cryptsetup luksFormat -v -y --key-file=/path/to/first-drive.key /dev/sdX
```

> If you intend to use the entire drive you will **not** need to create a
  [partition table](#create-partition-table) beforehand.

or we can encrypt an already existing partition

```bash
sudo cryptsetup luksFormat -v -y --key-file=/path/to/first-drive.key /dev/sdX2
```

Both of these operations are of course destructive, so any existing data on
the drive/partition will be lost.

### Unlock Encrypted Device
You now have to unlock the encrypted device/partition in order to be able to
use it in a way the rest of the computer can understand.

```bash
sudo cryptsetup luksOpen --key-file=/path/to/first-drive.key /dev/sdX first-drive
```

Here you need to change the X to either the device (or the partition) that
you have [just created](#encrypt-devicepartition). The name "`first-drive`", at
the end of the command, needs to be a unique name since this will be used as
sort of an "UUID" of the decrypted device.

> If you are really adamant about security you should proably take a quick look
  at the [Securely Format a Drive](#securely-format-a-drive) section now.

### Create Normal Filesystem
After you have unlocked the encrypted part you may take a look at the
[Create Filesystem](#create-filesystem) section under the
[Create Normal Drive](#create-normal-drive) chapter again, since right now the
mapped point will behave just like a normal device partition. Otherwise just
use the following command to create a standard filesystem on it:

```bash
sudo mkfs.ext4 /dev/mapper/first-drive
```

### Mount It
To test if you can use the newly created (and encrypted) filesystem you may
now mount it.

```bash
sudo mount /dev/mapper/first-drive /mnt/first-drive
```

The importance of the unique drive name is because [`crypttab`][54] will first
mount a "encrypted filesystem" to a point in `/dev/mapper/`, which then `fstab`
will use to mount to the "correct" location later. So something like this is
what is happening:


```
                           crypttab
            UUID     --------> | ----> /dev/mapper/first-drive
     (encrypted device)        |      (un-encrypted identifier)
```
```
                             fstab
 /dev/mapper/first-drive  ---> | --->  /mnt/first-drive
(un-encrypted identifier)      |     (desired mount point)
```

Try going into the desired mount point and create a file, just to be certain
everything works as intended.


## Create ZFS Array

1. [Choose a Good Disk Identifier](#choose-a-good-disk-identifier) - Once per drive.
2. [Create a Pool](#create-a-pool) - Once per array.
    - (Optional) [Enable `legacy` mounting](#the-mountpoint-property) - Once per pool or dataset.
3. [Create Dataset](#create-dataset) - As many as you like.
4. [Mount it](#mount-it-2) - You can mount all datasets at once.
5. [Read About Maintenance](#maintenance) - As many times you want to.

[ZFS][27] is touting to be "the last filesystem you will ever need", and it sure
does have a lot of features that will make the job of storing data much nicer.
However, it requires you to do quite a bit of research and configuring in order
for it to actually be as good as it advertise, so do not just follow this guide
blindly as my settings might not be the right ones for your setup.

Before continuing I would also just like to point out that the terminology of
ZFS components might not be totally straight forward if you are completely new
to this, so a suggestion is to have a look at [this documentation][24] for a
quick overview with perhaps a longer read of this [Ars Technica][28] article if
you want to start out somewhere.

But to keep it simple we will first make so that each drive we want to include
in the array has a non-changing identifier, then we will create the root
pool/tank with the `zpool` program before we finally create usable filesystems
(datasets) that reside inside this pool with the `zfs` program. Some decisions
cannot be changed after the creation of the pool (without starting over from
scratch) so it might be good to read through this entire section before
actually doing anything.

### Choose a Good Disk Identifier

When creating a pool we will need to provide target paths to each device we want
to include in the array, but since the classic `/dev/sdX` paths provide no
guarantee that a disk will get the same letter after a reboot we should really
try to use something that will not change without our knowledge.

There are a [few ways][32] to get an identifier that is constant, but I think
the best one is to use UUIDs. However, in order to get a UUID on a device we
first need to create a partition on the drive, and while you can target a naked
drive with ZFS (like most guides I have found do), doing it like this is a much
more robust solution and you only lose a megabyte or so of usable space. Just
follow the [Create Normal Drive](#create-normal-drive) guide section up until
the point where it mentions the creation of a filesystem, then you come back
and continue from here.

Another solid type of identifier, that is predictive, is the one you assign to
encrypted disks. While ZFS do have native support for encryption of each
separate dataset, it actually seems [more secure][22] and [faster][23] to just
put the ZFS filesystem on a LUKS encrypted drive and then use the decrypted
"mapper" mount point as target when referencing devices in ZFS. Follow the
[encryption guide](#create-encrypted-drive) until the step where it is time to
[unlock](#unlock-encrypted-device) the device, and then use its
`/dev/mapper/<id>` path as the identifier when you continue this guide.

### Create a Pool

After the preferred identifier has been chosen we will continue with creating
a pool. This will be the root location inside of which we will later create our
filesystems/datasets. Most of the [properties of the pool][35] are not
changeable later, so these need to be configured correctly from the beginning.
However, the default values are very reasonable with the notable exception being
[`ashift`][30] which should be manually configured by you according to the
sector size of your disks. It is a **very high** probability that you want
`ashift=12`, but please double check before continuing.

What is a little bit confusing is that the root of the pool is actually a usable
filesystem, but it appears like noone use this directly just because the
filepaths makes more sense in case you do not use it. However, it is still
possible to [set its properties][36], and this can be exploited in order to
create sane defaults for the future datasets in this pool since everything
inherits from its parent's properties unless something else is explicitly
specified.

So in the coming command, where a pool is created with the help of `zpool`,
the filesystem options are given with capital O and the pool options are with
a small letter (quite difficult to differentiate at a quick glance). But before
the command I will just quickly go through why I have provided the options that
I have, so I may look back at this and remember in the future.

#### Dataset Properties
There is a paper [here][25] which discuss which recordsize/blocksize to use for
best performance, but unless you are running a database on the disks you can be
quite satisfied with 128K. Compression is a nice bonus to have automatically
on everything you write and [`lz4`][29] is awfully fast in addition to providing
good compression ratios.

The `utf8only`, `casesensitivity` and `normalization` should give you a nice
filesystem where you can basically name your files whatever and have them sorted
in a sane fashion. The `sharesmb` and `sharenfs` options are in case you want
the ZFS mount program to automatically expose these as SMB or NFS shares as
well, but I prefer to handle that separately.

The `atime=on` option will make so that each file is updated with the timestamp
of when the file was last accessed. This cause a slight performance hit as a
write needs to happen every time a file is accessed, and in the case of a
database, where a lot of small files are accessed repeatedly and you don't care
of when these are accessed, you should turn this off. Having `relatime=on` in
addition to `atime` is a compromise of turning the latter off, as now the
access time is only updated if the modified time or changed time changes, or if
the existing access time has not been updated within the past 24 hours.

#### The Mountpoint Property
Finally there is the [`mountpoint`][34] property. Unless this value is set to
`legacy` or `none` the `zfs` program will automatically mount the dataset to
the path specified. How inheritance works for this property is like this: if
the dataset `pool1/media` has the mountpoint set to `/media/zfs`, then the
`pool1/media/movies` dataset will get `/media/zfs/movies` as its mountpoint.
It is also fully possible to mount the datasets `pool1/media` and
`pool1/media/movies` to two completely separate locations like `/media/zfs` and
`/mnt/movies` if you want, even though it looks weird.

Setting the property to `legacy` delegates the task of mounting the disk to
the normal `mount` program, which means you will have to manually mount the
dataset like this:

```
sudo mount -t zfs pool1/media/movies /mnt/movies
```

To have the `legacy` dataset mounted at boot you have to add an entry in
`fstab`, however, this is not a very common option and actually specifying
the mountpoint property on the dataset should be preferred unless you have a
good reason to despise it. Some more info about stuff to think about in case
you go this route is present in the [ZFS mounts](#zfs-mounts) section where
you also manage the `fstab` entries.

#### Creating the Pool
We are now finally ready to actually create the pool. Looking at the last line
I first specify the name of the pool (`pool1`), the redundancy of the array
([`raidz`][41]) and then finally all the devices (disks) that should be part
of it. As was mentioned before I use the `/dev/disk/<uuid>` path in order to
keep consistency after reboots.

```
sudo zpool create \
      -o ashift=12 \
      -O recordsize=128k \
      -O compression=lz4 \
      -O utf8only=on \
      -O casesensitivity=sensitive \
      -O normalization=formD \
      -O sharesmb=off \
      -O sharenfs=off \
      -O atime=on \
      -O relatime=on \
      -O mountpoint=legacy \
    pool1 raidz /dev/disk/<uuid1> /dev/disk/<uuid2> /dev/disk/<uuid3>
```

If this command succeed you may then list the info about the current pool with
the help of the following command:

```
sudo zfs get all pool1
```


### Create Dataset

With the pool now created we may continue to create datasets which work
basically like filesystems in normal drive terms. We have actually already
touched upon some [dataset properties](#dataset-properties) since the root pool
is usable as a dataset and has all the same options available (the capital `-O`
options above) as we [will have here][36].

I am usually quite happy with the same defaults we defined in the
[previous section](#creating-the-pool) for basically all my datasets, so I will
just add some options to the command below to show how it might look like if
you were to tune the dataset for housing databases instead.

```
sudo zfs create \
      -o recordsize=8k \
      -o atime=off \
      -o primarycache=metadata \
      -o logbias=throughput \
    pool1/postgreql
```

As you can see we use the `zfs` program here, as this only expose options for
creating datasets and thus we minimize the risk of doing something stupid.
But you may now create as many datasets you want, and they are very independent
of each other even if they appear to be located "inside" each other like
`pool1/media` and `pool1/media/movies`.

Remember that properties are inherited, so if you specified `legacy` as the
[mountpoint for the pool](#creating-the-pool) you will need to either set
a specific mountpoint for each dataset or make sure you  handle the mounting
manually via [`fstab`](#zfs-mounts).

### Mount It

As was mentioned earlier the ZFS service will automatically mount all datasets
that are not set to `legacy` or `none`, but you can trigger it again with

```
sudo zfs mount -a
```

or the following command if you have a dataset with `legacy` mounting options

```
sudo mount -t zfs pool1/media/movies /mnt/movies
```

Changing a datasets mount point is as simple as this

```
sudo zfs set mountpoint=/mnt/movies pool1/media/movies
```

which should trigger a mount automatically.

#### Import Pool

If you cannot mount anything and it complains about no pools being present you
can have ZFS scan your system and import any pools it finds:

```
sudo zpool import -a
```

and if you are transferring it from another computer you may need to forcefully
import it with its ID (which you get by using `-f` instead of `-a` in the
command above)

```
sudo zpool import -f 5536839315307152828
```

### Maintenance

I mentioned briefly in the beginning of this section that ZFS is a very solid
filesystem if you want to store a lot of data, but you will need to do some
proper configuring if you want it to keep the data safe over long periods of
time. And to help you get started with this never ending task I will just list
some of the things I though were important here.

#### Some Everyday Commands
Before diving too deep I thought it perhaps could be useful to list some simple
commands that may be used to just get a quick grasp of the system and make you
comfortable moving around.

**List Available Pools**
```
sudo zpool list
```

**List All ZFS Datasets**
```
sudo zfs list
```

**Get Info About Dataset**
```
sudo zfs get all pool1/media/movies
```

#### Health Monitoring
The most important thing you can do is to actually monitor the health of your
pools so you can act when something is awry. The simplest [command][43] you
can run to get an overview of your arrays is this:

```bash
sudo zpool status -x
```

This will print information about anything which ZFS thinks needs your
attention, or just "all pools are healthy" if everything is fine.

However, there is then no limit on how advanced you could make this type of
health monitoring, with remote logging services that send you mail based on
trigger words or similar so that you do not have to check manually every time.
But stuff like that is out of scope of this guide so I will just put [this][45]
link here for you to read if you want.

#### Scrubbing
[Scrubbing][44] is a process which will go through the data in the pool and
verify (agains checksums) that it is correct or try to fix it if it is not.
This should be done on a monthly basis and is triggered like this:

```bash
sudo zpool scrub pool1
```

This is very I/O intensive, so run this when there is little activity in your
pool. Furthermore, it is possible to pause the scrub whenever you like (with the
`-p` flag), and it will then resume from its latest checkpoint instead of the
beginning when you ask for a scrub again. This way you can incrementally scrub
you pool in case you have so much data it does not finish during its allotted
time.

#### Trimming
If you have SSDs (or some other flash based storage) as your backing disks you
should probably look into the [`trim`][42] command as this can help keep write
speeds stable if you perform a lot of deleting and rewriting of data.

You can enable the `autortim` property on each dataset, which will trigger small
and quick trimming quite often, but it is also recommended that you run a
proper `zpool trim` job [at the same frequency][31] as you scrub your pool.

#### Snapshotting
A very nice feature of ZFS is snapshoting, where you (almost) instantly create
sort of a checkpoint in time of your data, and then you can roll back to it in
case you accidentally removed something.

There is a quick guide on how to do this manually [here][46], but the TL;DR of
that one is basically the following two commands:

**Create Snapshot**
```
sudo zfs snapshot pool1/media/movies@snapshot-name
```

[**List Snapshots**][53]
```
sudo zfs list -r -t snapshot pool1/media/movies
```

**Restore Snapshot**
```
sudo zfs rollback pool1/media/movies@snapshot-name
```

However, you should also look into the following two programs which are able
to automate this process as well as clean up old snapshots.

1. [sanoid](https://github.com/jimsalterjrs/sanoid)
2. [zrepl](https://zrepl.github.io/)

### Replace a disk

Sooner or later you will reach a point where you need to replace a disk, and
while [there][47] are [a lot][48] of [guides][49] out [there][50] I will add
a small TL;DR with the same names as we have had in this guide up till now.

We will be using the [`replace`][51] command to tell ZFS that it should replace
the old drive with this new one:

```bash
sudo zpool replace rpool /dev/disk/by-uuid/<old_uuid> /dev/disk/by-uuid/<new_uuid>
```

The old path provided here is of course the same one we used when we first
added this disk to the pool, but this can be one of the other possible
[identifiers](#choose-a-good-disk-identifier) you have chosen. You can also
use `zdb` to find and reference the disk by its [GUID][52] in case all else
fails.

ZFS will immediately try to resilver the array with this new disk, so wait
patiently until that is done and monitor it so you know everything went well.

# Additional Info

This section contains additional information that may be good to know even when
you are not in the process of creating normal/encrypted drives.


## Obtain the UUID

In order to obtain the UUID ([universally unique identifier][16]) for the drives
connected to your system you can simply use the following command, which will
list all available/mountable devices:

```bash
sudo blkid
```

In my case it looked something like this.

```
/dev/sda1: UUID="88f0236b-620a-456e-a4a6-1b5c84996b5c" TYPE="ext4"
/dev/sda5: UUID="b8d210eb-d186-405f-bc42-a55cbe069a27" TYPE="swap"
```

However, if the drive you are looking for doesn't have any (mountable)
partitions on it you will first need to create them manually. A detailed guide
for how to produce a fully functional "normal" drive may be found in the
[Create Normal Drive](#create-normal-drive) section of the
[Preparation Steps](#preparation-steps). Afterwards you should be able
to find the newly created partition by running the command above again.

The same goes for an encrypted partition. After having completed the
[Encrypt Device/Partition](#encrypt-devicepartition) step, in the
[Create Encrypted Drive](#create-encrypted-drive) guide, you should be able to
find a UUID associated with the newly formed encrypted device.


## Test for Hardware Acceleration
If you intend to have encrypted drives attached to your system, you should
probably make sure that your CPU has support for hardware acceleration
([AES-NI][15]) and that [it is enabled][12]. To quickly check if everything is
working as expected you can run the following command:

```bash
openssl speed -evp aes-256-cbc
```

If the numbers returned from that test is lower than 500 MB/s something is
either wrongly configured, or you don't have support for hardware acceleration.
In that case I would strongly suggest you get a more modern CPU in order to
get a good experience with encrypted drives.


## Securely Format a Drive
If you have a new drive, or are re-purposing an old un-encrypted drive, you may
want to format it in such a way that the entire drive is filled with random
data. This is an important step if you are trying to achieve
"[plausible deniability][14]" for your encrypted drives, but should probably be
done anyways since it will erase any traces of old data that was there before.

The following command expects you to have completed the
[Create Encrypted Drive](#create-encrypted-drive) guide until the point where
this section is mentioned in the
[Unlock Encrypted Device](#unlock-encrypted-device) sub-section. Because
when the drive/partition has been unlocked, and it exists under `/dev/mapper/`,
you should run the following command to fill it with random data.

```bash
sudo dd if=/dev/zero of=/dev/mapper/first-drive bs=4M status=progress
```

Even if the input stream of the `dd` command is just zeroes, the encryption
algorithm will produce encrypted (i.e. it looks random) data that is written to
the drive. It is also possible to achieve the same result with the following
command (notice we target the actual device and not the decrypted mount point
now):

```bash
sudo dd if=/dev/urandom of=/dev/sdX bs=4M status=progress
```

However, this is usually slower than just spewing zeroes through the
[hardware accelerated](#test-for-hardware-acceleration) encryption algorithm.

If you are trying to create an encrypted drive you should now start that guide
over again from the [beginning](#create-encrypted-drive), so you get a
completely new encryption key afterwards. This is a paranoid security step
taken so there is absolutely no connection between the "encrypted zeroes" and
your real data.



## Ansible Variables Location
I prefer to add all the necessary variables, used by this role, in the
`host_vars/<hostname>` file, since these variables are usually unique for each
host. This way it is easy to separate the information on a host by host basis,
without creating any conflicts between them.

An important thing to remember is that Ansible will overwrite, and not merge,
hashes/dictionaries (like `mounts_boot` or `mounts_normal`) if there are two
with the same name. You can therefore not have a part of them be defined in the
`group_vars/` and then other parts in the `host_vars/`. If you do not like this
behavior you may look into setting [`hash_behaviour = merge`][2], but be aware
that this is not a very [good solution][3]. Instead you should probably look
into the [`combine`][5] filter or the [`merge_vars`][4] action plugin.






[1]: https://github.com/trapexit/mergerfs
[2]: https://docs.ansible.com/ansible/latest/reference_appendices/config.html#default-hash-behaviour
[3]: https://medium.com/uptime-99/3-things-ive-learned-about-ansible-the-hard-way-bae341524a86
[4]: https://pypi.org/project/ansible-merge-vars/
[5]: https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#combining-hashes-dictionaries
[6]: https://en.wikipedia.org/wiki/Master_boot_record
[7]: https://en.wikipedia.org/wiki/GUID_Partition_Table
[8]: https://rainbow.chard.org/2013/01/30/how-to-align-partitions-for-best-performance-using-parted/
[9]: https://man7.org/linux/man-pages/man8/parted.8.html
[10]: https://linux.die.net/man/8/cryptsetup
[11]: https://wiki.archlinux.org/index.php/Parted#Partition_schemes
[12]: https://www.cyberciti.biz/faq/how-to-find-out-aes-ni-advanced-encryption-enabled-on-linux-system/
[13]: https://askubuntu.com/a/653986
[14]: https://blog.linuxbrujo.net/posts/plausible-deniability-with-luks/
[15]: https://en.wikipedia.org/wiki/AES_instruction_set
[16]: https://en.wikipedia.org/wiki/Universally_unique_identifier
[17]: https://github.com/trapexit/mergerfs/releases/
[18]: https://github.com/trapexit/mergerfs/#options
[19]: https://www.howtogeek.com/465350/everything-you-ever-wanted-to-know-about-inodes-on-linux/
[20]: https://unix.stackexchange.com/a/7965
[21]: https://github.com/JonasAlfredsson/ansible-role-snapraid
[22]: https://old.reddit.com/r/zfs/comments/gnzl6m/zfs_native_encryption_or_zfs_on_luks/fre4mf9/
[23]: https://zfsonlinux.topicbox.com/groups/zfs-discuss/T154a5d1dad54570e-M84b984fca63d5c1cf94749bd/has-anyone-benchmarked-zfs-native-encryption-vs-zfs-on-luks
[24]: https://docs.freebsd.org/en/books/handbook/zfs/#zfs-term
[25]: https://www.usenix.org/system/files/login/articles/login_winter16_09_jude.pdf
[26]: https://wiki.archlinux.org/title/ZFS#Bind_mount
[27]: https://wiki.archlinux.org/title/ZFS
[28]: https://arstechnica.com/information-technology/2020/05/zfs-101-understanding-zfs-storage-and-performance/
[29]: https://www.xigmanas.com/wiki/doku.php?id=zfs:compression
[30]: https://forum.level1techs.com/t/i-need-to-set-ashift-values-for-ssds-in-zfs-proxmox-what-value-smartctl-is-confusing/178414/2
[31]: https://askubuntu.com/a/1200415
[32]: https://wiki.archlinux.org/title/ZFS#Identify_disks
[33]: https://wiki.archlinux.org/title/Persistent_block_device_naming#by-id_and_by-path
[34]: https://docs.oracle.com/cd/E19253-01/819-5461/gaztn/index.html
[35]: https://openzfs.github.io/openzfs-docs/man/7/zpoolprops.7.html
[36]: https://openzfs.github.io/openzfs-docs/man/7/zfsprops.7.html
[37]: https://www.golinuxcloud.com/mount-filesystem-in-certain-order-systemd/
[38]: https://www.thegeeksearch.com/how-to-manage-order-of-mounting-in-centos-rhel-78-using-systemd/
[39]: https://www.freedesktop.org/software/systemd/man/systemd.mount.html
[40]: https://www.freedesktop.org/software/systemd/man/systemd.generator.html
[41]: http://www.raidz-calculator.com/raidz-types-reference.aspx
[42]: https://openzfs.github.io/openzfs-docs/man/8/zpool-trim.8.html
[43]: https://docs.oracle.com/cd/E19253-01/819-5461/gamno/index.html
[44]: https://openzfs.github.io/openzfs-docs/man/8/zpool-scrub.8.html
[45]: https://old.reddit.com/r/zfs/comments/ifhlb4/whats_your_favorite_monitoring_tool_for_zfs/
[46]: https://ramsdenj.com/2016/08/29/arch-linux-on-zfs-part-3-followup.html
[47]: https://docs.joyent.com/private-cloud/troubleshooting/disk-replacement
[48]: https://www.adminbyaccident.com/freebsd/how-to-freebsd/how-to-replace-a-disk-on-a-zfs-mirror-pool/
[49]: https://dannyda.com/2020/05/16/how-to-replace-dead-physical-disk-from-proxmox-pve-for-zfs-pool-easily/
[50]: https://jordanelver.co.uk/blog/2018/11/26/how-to-replace-a-failed-disk-in-a-zfs-mirror/
[51]: https://openzfs.github.io/openzfs-docs/man/8/zpool-replace.8.html
[52]: https://askubuntu.com/questions/305830/replacing-a-dead-disk-in-a-zpool
[53]: https://docs.oracle.com/cd/E19253-01/819-5461/gbiqe/index.html
[54]: https://www.freedesktop.org/software/systemd/man/crypttab.html
