# Self Hosting with lxc and Linux

Notes on setting up LXC containers with bridged networking on Ubuntu 24.04, with the aim of running a set of services on a private network.

The purpose of this is to set up a number of private and public services that include:

* NextCloud - for control and sharing of documents
* GitLab (Community Edition) - for software version control
* Remind - for managing tasks and project management

For this to be cost effective, the hardware needs to be modest. A NUC would be ideal, but in my own case, I am reusing an old Mac Mini (2014) running Ubuntu 24.04 connected to home router over **ethernet**. This has an internal SSD plus an additional NVMe drive fitted.

For bulk storage, I am using a budget Synology NAS on the same network.

## Bridged Networking with Netplan

Ubuntu 24.04 should have Netplan installed by default. 
You should have a file 50-cloud-init.yaml in the director `/etc/netplan`.

**Make a back up your Netplan configuration file**

`sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.original`

**Check the name of your ethernet device**

On the host PC, you can list all the network interfaces using the command `ip link show` as follows:
   
```
> ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp3s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP mode DEFAULT group default qlen 1000
    link/ether ac:87:a3:11:7f:32 brd ff:ff:ff:ff:ff:ff
3: wlp2s0: <NO-CARRIER,BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state DORMANT mode DORMANT group default qlen 1000
    link/ether 60:f8:1d:b8:3d:cc brd ff:ff:ff:ff:ff:ff
```

I can see from the output that my ethernet device is named `enp3s0f0` (another common name might be `eth0`). My private subnet is `192.168.1.0/24`, so I can also check which interface has an address on that subnet:

```
> ip -4  a 
```

Look for the device with an appropriate IP address. Now if we examine the current Netplan interface file, we will see the ethernet interface featured:

```
> sudo cat /etc/netplan/50-cloud-init.yaml.previous 
```

For my machine, I see the following:

```
network:
    ethernets:
        enp3s0f0:
            dhcp4: true
    version: 2
```

Here you can see that the interface `enp3s0f0` obtains it's IP address via DHCP (the server is running on my router). Now we want to create a network bridge.

**Edit the Netplan Config File**

```
> sudo nano /etc/netplan/50-cloud-init.yaml
```

In my own case, I add the bridge interface `br0` as follows:

```
network:
    ethernets:
        enp3s0f0:
            dhcp4: false
    version: 2
    bridges:
        br0:
            dhcp4: true
            interfaces: [enp3s0f0]
            parameters:
                stp: true
                forward-delay: 4
```

Note the following:

* enp3s0f0 no longer acquires an IP address from the DHCP server
* the bridge interface `br0` connects (via the ethernet) to the DHCP server on the router.
* The Spanning Tree Protocol (stp) is reccomended to protect against "network loops", but is not really necessary for this simple setup.

Now apply the changes:

```
sudo netplan apply
```

We can see the effect of this as follows:

```
> ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp3s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP group default qlen 1000
    link/ether ac:87:a3:11:7f:32 brd ff:ff:ff:ff:ff:ff
3: wlp2s0: <NO-CARRIER,BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state DORMANT group default qlen 1000
    link/ether 60:f8:1d:b8:3d:cc brd ff:ff:ff:ff:ff:ff
4: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 66:a9:74:fa:fb:98 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.223/24 brd 192.168.1.255 scope global dynamic 
    ...
```

Note that the `br0` interface has been allocated an IP address on my local network (starting `192.168.1`).

```
> brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.66a974fafb98       yes             enp3s0f0
```

```
> nmcli connection show
NAME              UUID                                  TYPE      DEVICE   
netplan-br0       00679506-5c05-3c3d-bdfe-474849762078  bridge    br0      
netplan-enp3s0f0  d235d842-f3b8-3192-a46c-301acf30866a  ethernet  enp3s0f0 
lo                7eeff30e-3dcb-4f71-82e2-0d67843be18e  loopback  lo 
```


