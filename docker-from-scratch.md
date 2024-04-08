# Docker from scratch

This is a hands on guide showing how to do containers from scratch. The original blueprint from this guide is based on the excellent talk [Cgroups, namespaces, and beyond: what are containers made from?](https://www.youtube.com/watch?v=sK5i-N34im8&t=2708s
) by Jérôme Petazzoni, but I will try to make more comprehensive in future iterations.

## Pre-requisites

> Do NOT run this on your workstation or production environment. **Use a VM**. I personally Use [OrbStack](https://orbstack.dev/) light VMs, they are great for this kind of thing, but [Qemu](https://www.qemu.org/) or some other would do, whatever floats your VM boat.

Most commands used here are part of most Linux distributions except for the following two:

* btrfs-progs --> `apt install btrfs-progs -y`
* [Docker](https://docs.docker.com/engine/install/)
* A lot of the commands below require sudo, so just run as root. 

## Creating a container

Each step in this guide has one or more commands, it is designed so you can just copy and paste the entire block of commands and it will work.

### 1. Name your container:

This will set the name of our container, the variable is used in many of the commands below.

```bash
export CONTAINER_NAME=goofy-container
```

### 2. Create directories and btrfs subvolume

btrfs is a [copy-on-write](https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Copy_on_Write_.28CoW.29
) (CoW) filesystem that supports snapshots. Whenever you create or modify a file with CoW, the filesystem makes a copy of the blocks, writes the changes and then replaces the old block by referencing to the new one and marking the old block available.

[Snapshots](https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Snapshots) are references to an entire [subvolume](https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Subvolumes) in the btrfs filesystem. Thanks to CoW, when you create a new snapshot you are not actually copying the entire subvolume, you are only copying data when you make changes to it, essentially only writing the differences. 

The user experiences this by having two separate folders, one subvolume and a snapshot of that subvolume (which is also a subvolume), both directories can be changed independently, but behind the scenes the filesystem only writes the differences to disk, meaning that you don't use twice as much disk space.

btrfs is not Docker's default [storage driver](https://docs.docker.com/storage/storagedriver/btrfs-driver/
). We are using it here cause it's simpler than setting up the default overlay2.

Below we create the directories we are going to be using and then we create a subvolume for our container.

```bash 
mkdir -p /btrfs/{containers,images}
btrfs subvolume create /btrfs/containers/$CONTAINER_NAME
```

### 3. Create the image

In this step we untar the alpine image from the docker repository and then we create a read-only snapshot for our image, that way we prevent any changes to the image.

```bash
CID=$(docker create alpine)
docker export $CID | tar -xf - -C /btrfs/containers/$CONTAINER_NAME

btrfs subvolume snapshot -r /btrfs/containers/$CONTAINER_NAME /btrfs/images/alpine
```

> Why not create a read-only subvolume first and then do a read-write snapshot of it? You cannot, and since a snapshot of a subvolume is just another subvolume, it doesn't matter anyway. 


### 4. Setting up network namespace for our container

The main idea here is that we are creating a bridge in the host and a network pair `host (hveth0) <--> container (cveth0)` to communicate with it from the container namespace. We are also setting up an iptables forwarding rule to connect to the Internet from the container via the host.

In this section we are going to be creating and configuring the following: 

1. A network namespace named `$CONTAINER_NAME` for our container.
1. A `host (hveth0) <--> container (cveth0)` virtual ethernet pair
1. Add an ip address of `172.18.0.10/16` to our container virtual ethernet (`cveth0`)
1. A bridge at the host namespace for our containers (`cbridge`)
1. Attach the host virtual ethernet (`hveth0`) to the container bridge (`cbridge`)
1. Add an ip address `172.18.0.1/16` to our container bridge at the host 
1. Add a default route for our container namespace that points to our bridge ip address `172.18.0.1`
1. Set up nat iptables so we can route traffic from the container to the internet via the bridge.

> When you install docker you will notice that it creates a bridge called `docker0`. You can check it out by running the command `ip a show docker0`. This bridge interface is always down unless you run a container. When you do, docker brings the bridge interface up and creates one veth pair each container you create for as long as the container exists. So essentially docker is doing something similar to what you are going to be doing in this step. 

If you want to understand how container networking works in a bit more detail, I recommend this [resource](https://labs.iximiuz.com/tutorials/container-networking-from-scratch)


```bash
# 1. Create network namespace
ip netns add $CONTAINER_NAME

# 2. create host (hveth0) <--> container (cveth0) virtual ethernet pair 
ip link add hveth0 type veth peer name cveth0 netns $CONTAINER_NAME

# Bring all devices up
ip link set hveth0 up
ip -n $CONTAINER_NAME link set cveth0 up
ip -n $CONTAINER_NAME link set lo up

# 3. Add an ip address of 172.18.0.10/16 to our container virtual ethernet (cveth0)
ip -n $CONTAINER_NAME addr add 172.18.0.10/16 dev cveth0

# 4. Create bridge at the host namespace for our containers (cbridge)
ip link add cbridge type bridge
ip link set cbridge up

# 5. Attach the host virtual ethernet (hveth0) to the container bridge (cbridge)

ip link set hveth0 master cbridge

# 6. Add an ip address 172.18.0.1/16 to our container bridge at the host

ip addr add 172.18.0.1/16 dev cbridge

# 7. Add a default route for our container namespace that points to our bridge ip address 172.18.0.1

ip -n $CONTAINER_NAME route add default via 172.18.0.1

# 8. Set up nat iptables so we can route traffic from the container to the internet via the bridge.
iptables -t nat -A POSTROUTING -s 172.18.0.0/16 ! -o cbridge -j MASQUERADE

# test internet connectivity:
ip netns exec $CONTAINER_NAME ping -c 2 8.8.8.8

```
> If the above doesn't work you may need to enable port forwarding by entering the command `echo 1 > /proc/sys/net/ipv4/ip_forward` although this is usually enabled by default, especially if you have docker installed


### 5. Namespace mounts

You can use a special type of mount namespace filesystem to mount namespaces. This allows to persist the namespace after the process has died. If this is not done, the namespace does not persist after the process that holds it terminates.

When we created a network namespace in the above step, it automatically created a mount for the namespace under `/run/ns/${CONTAINER_NAME}` which we will use when we create our container, this step will mount all other namespaces. Here we will use the command `unshare`, which is used to create a process with separate namespace we provide. When you run unshare specifying a file path for a namespace it will mount the namespace in that location.

> Note if you run the command unshare multiple times specifying a path for the namespace it will keep mounting on top of that file, so if you want to revert to the previous mount you will need to unmount it. You can do this by running `umount /run/ns/*`

In this step we will do the following:

1. create a directory under `/run/ns` to place all our namespace mounts. 
1. Create a bind mount of directory `/run/ns` in itself so we can set the [propagation](https://www.kernel.org/doc/Documentation/filesystems/sharedsubtree.txt
) to private.
1. create the files for the namespaces mount, ipc and uts under `/run/ns`
1. Run the unshare command once to create the [namespace mounts](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html)  and set the hostname to `${CONTAINER_NAME}`

```bash
mkdir /run/ns
mount --bind /run/ns/ /run/ns/
mount --make-private /run/ns/

touch /run/ns/${CONTAINER_NAME}-{mount,ipc,uts}


unshare --mount=/run/ns/${CONTAINER_NAME}-mount --ipc=/run/ns/${CONTAINER_NAME}-ipc --uts=/run/ns/${CONTAINER_NAME}-uts hostname $CONTAINER_NAME
```

> We are not setting up a mount for the pid namespace because it is not persistent after the PID process that holds it dies. A new namespace needs to be created for it every time you stop and restart the container.

### 6. Configure the mount namespace (chrooting)

If we don't do this step, we will still have access to all the files at the host. We will be using `nsenter` to enter a process in the mount namespace, run a set up command and then exit, then the last command will enter the container in the shell. The last command can be repeated as many times as desired to enter and exit the container.


In the step we will do the following:

In the host:
1. Make a directory `oldroot` to swap our root mount to.

In the container namespace:
1. Make a bind mount in the /btrfs/containers/$CONTAINER_NAME into itself so we can use it as a mount point to pivot to.
1. Run the command to pivot (swap) the root mount.
1. Unmount every other mount inherited from the host.
1. Mount back the proc filesystem and check that the processes are isolated.
1. Enter our container, poke around all you want, and `exit` when you are done.
1. You can rerun last command as many times as you want for as long as you keep all other namespaces alive.


```bash
mkdir /btrfs/containers/$CONTAINER_NAME/oldroot 

nsenter --mount=/run/ns/${CONTAINER_NAME}-mount mount --bind /btrfs/containers/{$CONTAINER_NAME,$CONTAINER_NAME}/

nsenter --mount=/run/ns/${CONTAINER_NAME}-mount pivot_root /btrfs/containers/{$CONTAINER_NAME,$CONTAINER_NAME/oldroot}

nsenter --mount=/run/ns/${CONTAINER_NAME}-mount sh -c "mount -t proc none /proc && umount -a ; umount -l /oldroot"

nsenter --mount=/run/ns/${CONTAINER_NAME}-mount unshare --pid --fork sh -c "mount -t proc none /proc && ps -ef"


# We need to first umount the first proc
nsenter --mount=/run/ns/${CONTAINER_NAME}-mount --ipc=/run/ns/${CONTAINER_NAME}-ipc --uts=/run/ns/${CONTAINER_NAME}-uts --net=/run/netns/${CONTAINER_NAME} unshare --pid --fork sh -c "umount /proc && mount -t proc none /proc && sh"

```
> Notice this is as easy as 


## clean up

### 1. Clean up namespaces and network

If you want to start the tutorial again from step number 4 run these commands:

```bash
ip netns delete $CONTAINER_NAME
ip link delete cbridge
umount --recursive /run/ns/
rm -rf /run/ns
rmdir /btrfs/containers/$CONTAINER_NAME/oldroot
```

Continue to step 3 if want to clean completely, or number 2 if you also want to start over with a fresh container from the image we created earlier.

### 2. Revert container to image.

This would be the equivalent of deleting a container and recreating it from the image, only do this step if you want to start over from the read only snapshot image we created earlier.

```bash
btrfs subvolume delete /btrfs/containers/$CONTAINER_NAME
btrfs subvolume snapshot /btrfs/images/alpine /btrfs/containers/$CONTAINER_NAME
```

### 3. Delete both subvolumes
If you want to clean completely do this after clean up step number 1 above:

```bash
btrfs subvolume delete /btrfs/containers/$CONTAINER_NAME /btrfs/images/alpine
rm -r /btrfs
```

## Roadmap 

* Adding cgroups mount and test some cgroup functionality
* Adding user namepaces and explore functionality. 
* Replace btrfs with overlay
* Provide more detailed explanations for IPC and mount namespaces in a different section
* Add more mounts that are missing to the container, for example /sys /tmp, etc and explore the implications



## Additional reading

https://docs.docker.com/storage/storagedriver/btrfs-driver/

https://fedoramagazine.org/working-with-btrfs-snapshots/

https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Snapshots

https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Copy_on_Write_.28CoW.29

https://archive.kernel.org/oldwiki/btrfs.wiki.kernel.org/index.php/SysadminGuide.html#Subvolumes

https://tbhaxor.com/breaking-out-of-chroot-jail-shell-environment/

https://yarchive.net/comp/linux/pivot_root.html

https://tbhaxor.com/pivot-root-vs-chroot-for-containers/