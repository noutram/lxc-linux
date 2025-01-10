[BACK](./README.md)

---

# Install and Configure LXC

```
sudo apt update
sudo apt install lxc
```

## Create a Container

I will create a container for gitlab.

```
sudo lxc-create -n gitlab -t download
```

Answer the questions. I chose:

```
Distribution: 
ubuntu
Release: 
noble
Architecture: 
amd64
```

You will then see information as it progresses:

```
Downloading the image index
Downloading the rootfs
Downloading the metadata
The image cache is now ready
Unpacking the rootfs

---
You just created an Ubuntu noble amd64 (20250110_07:42) container.

To enable SSH, run: apt install openssh-server
No default root or user password are set by LXC.
```

You can confirm this with:

```
sudo lxc-ls
```

## Start the container

You can start the container as follows:

```
sudo lxc-start -n gitlab
```

You can attach to the container to set the root password:

```
sudo lxc-attach -n gitlab
passwd
exit
```


If you look at the IP address of this LXC container, you will see it is on a private subnet:

```
root@gitlab:/# ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 76:8d:e6:d9:60:cd brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.3.9/24 metric 100 brd 10.0.3.255 scope global dynamic eth0
       valid_lft 3492sec preferred_lft 3492sec
    inet6 fe80::748d:e6ff:fed9:60cd/64 scope link 
       valid_lft forever preferred_lft forever
```

## Stop the container

```
sudo lxc-stop -n gitlab
```

## Change the network to bridged

When creating or configuring your LXC containers, specify the use of the `br0` bridge interface. You do this by editing the the container configuration file in `/var/lib/lxc/<container_name>/config`.

For this example, modify the network configuration for the gitlab container by editing `/var/lib/lxc/gitlab/config`:


```
# Network configuration
lxc.net.0.type = veth
lxc.net.0.link = br0
lxc.net.0.flags = up
#lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx  # Optional: you can specify a MAC address. Useful for static allocation of IP from your router.
#lxc.net.0.ipv4.address = 192.168.1.2/24
#lxc.net.0.ipv4.gateway = 192.168.1.254    
```      
       
This configuration sets up a virtual Ethernet device (veth) for the container and links it to the `br0` bridge interface. Uncomment the last two lines if you want to use a static IP.

Now start the LXC container and attach to it:

```
sudo lxc-start -n gitlab
sudo lxc-attach -n gitlab
```

Note the IP address for the ethernet interface. It should be on the same subnet as the host server.

## Setup SSH
To access and administer the container remotely, install `ssh`

```
apt install openssh-server
```

It is reccomended to create a new user that is also a member of the `sudo` group. 

```
# Create a new user (replace 'newuser' with the desired username)
 adduser newuser

# Add the new user to the sudo group
usermod -aG sudo newuser
```

`ssh` as `root` should be blocked.

Edit the SSH configuration file

```
nano /etc/ssh/sshd_config
```

Ensure the following lines are present and uncommented

```
PermitRootLogin no AllowUsers newuser
```

If you make any edits, then restart the SSH service to apply changes

```
systemctl restart ssh
```

## Snapshop

Once we have out conrainer working well, this is a good time to take a snapshot.

```
lxc-snapshot <container-name> -N <snapshot-name>
```

So in this case:

```
lxc-snapshot gitlab -N initial_setup
```

We can list (L) and restore (r) the snapshot as follows:

```
lxc-snapshot -L <container-name>
lxc-snapshot -r <container-name> <snapshot-name>
```


---

[NEXT - Setup Gitlab](./setup_gitlab.md)
