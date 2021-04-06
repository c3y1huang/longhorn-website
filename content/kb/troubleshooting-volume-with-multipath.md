---
title: "Troubleshooting: `MountVolume.SetUp failed for volume` due to multipathd on the node"
author: Clark Hsu
draft: false
date: 2021-03-16
catelogies:
  - "csi"
  - "multipathd"
---

## Applicable versions

All Longhorn versions.

## Symptoms

The pod with the volume is not starting and encounters error messages in `longhorn-csi-plugin`:

```
time="2020-04-16T08:49:27Z" level=info msg="GRPC request: {\"target_path\":\"/var/lib/kubelet/pods/cf0a0b5b-106e-4793-a74a-28bfae21be1a/volumes/kubernetes.io~csi/pvc-d061512e-870a-4ece-bd45-2f04672d5256/mount\",\"volume_capability\":{\"AccessType\":{\"Mount\":{\"fs_type\":\"ext4\"}},\"access_mode\":{\"mode\":1}},\"volume_context\":{\"baseImage\":\"\",\"fromBackup\":\"\",\"numberOfReplicas\":\"3\",\"staleReplicaTimeout\":\"30\",\"storage.kubernetes.io/csiProvisionerIdentity\":\"1586958032802-8081-driver.longhorn.io\"},\"volume_id\":\"pvc-d061512e-870a-4ece-bd45-2f04672d5256\"}"
time="2020-04-16T08:49:27Z" level=info msg="NodeServer NodePublishVolume req: volume_id:\"pvc-d061512e-870a-4ece-bd45-2f04672d5256\" target_path:\"/var/lib/kubelet/pods/cf0a0b5b-106e-4793-a74a-28bfae21be1a/volumes/kubernetes.io~csi/pvc-d061512e-870a-4ece-bd45-2f04672d5256/mount\" volume_capability:<mount:<fs_type:\"ext4\" > access_mode:<mode:SINGLE_NODE_WRITER > > volume_context:<key:\"baseImage\" value:\"\" > volume_context:<key:\"fromBackup\" value:\"\" > volume_context:<key:\"numberOfReplicas\" value:\"3\" > volume_context:<key:\"staleReplicaTimeout\" value:\"30\" > volume_context:<key:\"storage.kubernetes.io/csiProvisionerIdentity\" value:\"1586958032802-8081-driver.longhorn.io\" > "
E0416 08:49:27.567704 1 mount_linux.go:143] Mount failed: exit status 32
Mounting command: mount
Mounting arguments: -t ext4 -o defaults /dev/longhorn/pvc-d061512e-870a-4ece-bd45-2f04672d5256 /var/lib/kubelet/pods/cf0a0b5b-106e-4793-a74a-28bfae21be1a/volumes/kubernetes.io~csi/pvc-d061512e-870a-4ece-bd45-2f04672d5256/mount
Output: mount: /var/lib/kubelet/pods/cf0a0b5b-106e-4793-a74a-28bfae21be1a/volumes/kubernetes.io~csi/pvc-d061512e-870a-4ece-bd45-2f04672d5256/mount: /dev/longhorn/pvc-d061512e-870a-4ece-bd45-2f04672d5256 already mounted or mount point busy.
E0416 08:49:27.576477 1 mount_linux.go:487] format of disk "/dev/longhorn/pvc-d061512e-870a-4ece-bd45-2f04672d5256" failed: type:("ext4") target:("/var/lib/kubelet/pods/cf0a0b5b-106e-4793-a74a-28bfae21be1a/volumes/kubernetes.io~csi/pvc-d061512e-870a-4ece-bd45-2f04672d5256/mount") options:(["defaults"])error:(exit status 1)
time="2020-04-16T08:49:27Z" level=error msg="GRPC error: rpc error: code = Internal desc = exit status 1"
```

## Details

This is caused by multipath creating a multipath device for any eligible device path including every Longhorn volume device not explicitly blacklisted.

### Troubleshooting

1. Find the major:minor number of the Longhorn device. On the node, try `ls -l /dev/longhorn/`. The major:minor number will be shown as e.g. `8, 32` before the device name.

```
ls -l /dev/longhorn
brw-rw---- 1 root root 8, 32 Aug 10 21:50 pvc-39c0db31-628d-41d0-b05a-4568ec02e487
```

2. Find what's the device generated by Linux for the same major:minor number. Use `ls -l /dev` and find the device for the same major:minor number, e.g. `/dev/sde`.

```
brw-rw---- 1 root disk      8,  32 Aug 10 21:50 sdc
```

3. Find the process. Use `lsof` to get the list of file handlers in use, then grep for the device name (e.g. `sde` or `/dev/longhorn/xxx`. You should find one there.

```
lsof | grep sdc
multipath 514292                              root    8r      BLK               8,32        0t0        534 /dev/sdc
multipath 514292 514297 multipath             root    8r      BLK               8,32        0t0        534 /dev/sdc
multipath 514292 514299 multipath             root    8r      BLK               8,32        0t0        534 /dev/sdc
multipath 514292 514300 multipath             root    8r      BLK               8,32        0t0        534 /dev/sdc
multipath 514292 514301 multipath             root    8r      BLK               8,32        0t0        534 /dev/sdc
multipath 514292 514302 multipath             root    8r      BLK               8,32        0t0        534 /dev/sdc
multipath 514292 514304 multipath             root    8r      BLK               8,32        0t0        534 /dev/sdc
```

#### Solution

Follow these steps to prevent the multipath daemon from adding additional block devices created by Longhorn.

First check devices created by Longhorn using `lsblk`:

```
root@localhost:~# lsblk
NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda    8:0    0 79.5G  0 disk /
sdb    8:16   0  512M  0 disk [SWAP]
sdc    8:32   0    1G  0 disk /var/lib/kubelet/pods/c2c2b848-1f40-4727-8a52-03a74f9c76b9/volumes/kubernetes.io~csi/pvc-859bc3c9-faa8-4f54-85e4-b12935b5ae3c/mount
sdd    8:48   0    1G  0 disk /var/lib/kubelet/pods/063a181a-66ac-4644-8268-9215305f9b73/volumes/kubernetes.io~csi/pvc-837eb6ac-45fe-4de7-9c97-8d371eb02190/mount
sde    8:64   0    1G  0 disk /var/lib/kubelet/pods/4c80842d-7257-4b91-b668-bb5b111da003/volumes/kubernetes.io~csi/pvc-c01cee3e-f292-4979-b183-6546d6397fbd/mount
sdf    8:80   0    1G  0 disk /var/lib/kubelet/pods/052dadd9-042a-451c-9bb1-2d9418f0381f/volumes/kubernetes.io~csi/pvc-ba7a5c9a-d84d-4cb0-959d-3db39f34d81b/mount
sdg    8:96   0    1G  0 disk /var/lib/kubelet/pods/7399b073-c262-4963-8c7f-9e481272ea36/volumes/kubernetes.io~csi/pvc-2b122b42-141a-4181-b8fd-ce3cf91f6a64/mount
sdh    8:112  0    1G  0 disk /var/lib/kubelet/pods/a63d919d-201b-4eb1-9d84-6440926211a9/volumes/kubernetes.io~csi/pvc-b7731785-8364-42a8-9e7d-7516801ab7e0/mount
sdi    8:128  0    1G  0 disk /var/lib/kubelet/pods/3e056ee4-bab4-4230-9054-ab214bdf711f/volumes/kubernetes.io~csi/pvc-89d37a02-8480-4317-b0f1-f17b2a886d1d/mount
```

Notice that Longhorn device names start with `/dev/sd[x]`

- Create the default configuration file `/etc/multipath.conf` if it does not exist
- Add the following line to blacklist section `devnode "^sd[a-z0-9]+"`

```
blacklist {
    devnode "^sd[a-z0-9]+"
}
```

- Restart multipath service

  `systemctl restart multipathd.service`

- Verify that configuration is applied

  `multipath -t`

> The default configurations for multipath blacklist section is preventing the following device names by default
> ^(ram|raw|loop|fd|md|dm-|sr|scd|st|dcssblk)[0-9]
> ^(td|hd|vd)[a-z]

## Related information

- Related Longhorn issue: https://github.com/longhorn/longhorn/issues/1210
- Related Manpage http://manpages.org/multipathconf/5