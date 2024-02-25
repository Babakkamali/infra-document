# Setup glusterfs

## Installation

### Plan and setup the infrastructure

It's better to add seperate disks to VMs for gluster fs for each of your vms
Rescan All SCSI Buses: To trigger a scan on all SCSI buses, to add the new disks ,you can issue the following command:

```bash
echo '- - -' | sudo tee /sys/class/scsi_host/host*/scan
```

then use this command lsblk to find the name of the new dervier like sdb

```bash
lsblk
```

Update the /etc/hosts files on each VM to reflect the private IPs of each VM. This is important for GlusterFS, and you may encounter bugs or issues if you give private IPs directly to Gluster volumes. After editing the files should look like:

```bash
(gluster1)# cat /etc/hosts
127.0.0.1       localhost
192.168.2.100   gluster1
192.168.2.101   gluster2
192.168.2.102   gluster3

(gluster2)# cat /etc/hosts
127.0.0.1       localhost
192.168.2.100   gluster1
192.168.2.101   gluster2
192.168.2.102   gluster3

(gluster3)# cat /etc/hosts
127.0.0.1       localhost
192.168.2.100   gluster1
192.168.2.101   gluster2
192.168.2.102   gluster3
```

Format the disks with xfs filesystem on each VM in case you haven’t already. You can also use ext4 if you prefer:

```bash
# mkfs.xfs /dev/sdb
```

### Create directories for GlusterFS storage

```bash
(gluster1)# mkdir -p /gluster/bricks/1
(gluster1)# echo '/dev/sdb /gluster/bricks/1 xfs defaults 0 0' >> /etc/fstab
(gluster1)# mount -a
(gluster1)# mkdir /gluster/bricks/1/brick

(gluster2)# mkdir -p /gluster/bricks/2
(gluster2)# echo '/dev/sdb /gluster/bricks/2 xfs defaults 0 0' >> /etc/fstab
(gluster2)# mount -a
(gluster2)# mkdir /gluster/bricks/2/brick

(gluster3)# mkdir -p /gluster/bricks/3
(gluster3)# echo '/dev/sdb /gluster/bricks/3 xfs defaults 0 0' >> /etc/fstab
(gluster3)# mount -a
(gluster3)# mkdir /gluster/bricks/3/brick
```

### Install GlusterFS

Install GlusterFS on all VMs by executing following commands on each VM:

```bash
# apt -y update && apt -y upgrade
# apt install -y software-properties-common
# add-apt-repository ppa:gluster/glusterfs-11 && apt update # Use the latest glusterFS version instead of 6, which was the latest at the time of writing this tutorial
# apt install -y glusterfs-server
# systemctl enable glusterd && systemctl start glusterd && systemctl status glusterd # automatically start glusterfs on boot
```

### Install GlusterFS

```bash
(gluster1)# gluster peer probe gluster2
peer probe: success.

(gluster1)# gluster peer probe gluster3
peer probe: success.

(gluster1)# gluster peer status
Number of Peers: 2

Hostname: gluster2
Uuid: 60861905-6adc-4841-8f82-216c661f9fe2
State: Peer in Cluster (Connected)

Hostname: gluster3
Uuid: 572fed90-61de-40dd-97a6-4255ed8744ce
State: Peer in Cluster (Connected)
```

### Setup the Gluster “replicated volume”

GlusterFS has multiple volume types. For our HA architecture, we want to setup a “replicated” volume that stores the files created on each of the 3 VMs and hence the file is available to any app or container running on these VMs. Create the replicated volume named “gfs” with 3 replicas:

```bash
(gluster1)# gluster volume create gfs \
replica 3 \
gluster1:/gluster/bricks/1/brick \
gluster2:/gluster/bricks/2/brick \
gluster3:/gluster/bricks/3/brick
```

you should get the message
volume create: gfs: success: please start the volume to access data

```bash
(gluster1)# gluster volume start gfs
(gluster1)# gluster volume status gfs
```

### Setup security and authentication for this volume

GlusterFS will allow any clients to connect to volumes by default. However, you will need to authorize the three infra VMs running GlusterFS to connect to the GlusterFS Volumes on each node. You can do it by authorizing the private IPs of each VM to connect to the volume. This will allow replication to happen. Execute:


```bash
(gluster1)# gluster volume set gfs auth.allow 192.168.2.100,192.168.2.101,192.168.2.102
```

### Mount the glusterFS volume where applications can access the files

We’ll mount the volume onto /mnt on each VM, and also append it to our /etc/fstab file so that it mounts on boot:


```bash
(gluster1)# echo 'localhost:/gfs /mnt glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0' >> /etc/fstab
(gluster1)# mount.glusterfs localhost:/gfs /mnt

(gluster2)# echo 'localhost:/gfs /mnt glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0' >> /etc/fstab
(gluster2)# mount.glusterfs localhost:/gfs /mnt

(gluster3)# echo 'localhost:/gfs /mnt glusterfs defaults,_netdev,backupvolfile-server=localhost 0 0' >> /etc/fstab
(gluster3)# mount.glusterfs localhost:/gfs /mnt
```

### Verify

Verify mounted glusterfs volume:

```bash
# df -Th
```