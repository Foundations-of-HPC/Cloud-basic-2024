# Virtualization Tutorial

In this tutorial, we will learn how to install a set of interoperable Linux virtual machines on our local environment using Virtualbox. 
Each machine will have two NICs one internal and one to connect to WAN .
We will connect to them from our host machine via SSH.

The primary goal will be to test the guest virtual machine performances using standard benckmaks as HPL, STREAM or iozone and compare with the host performances.


## GOALs
In this tutorial, we are going to create a set of four Linux virtual machines.

* Each machine is capable of connecting to the internet and able to connect with each other privately as well as can be reached from the host machine.
* Our machines will be named master, node01, node02,..., node0X.
* The first machine: cluster01 will act as a master node and will have 1vCPUs, 2GB of RAM and 25 GB hard disk.
* The other machines will act as workers and they will have 1vCPUs, 1GB of RAM and 10 GB hard disk.
* We will assign our machines static IP address in the internal network: 192.168.0.1, 192.168.0.22, 192.168.0.23, .... 192.168.0.XX.

## Prerequisite

* VirtualBOX installed in your linux/windows/Apple
* ubuntu server 24.04 LTS image to install
* SSH client to connect

## Create virtual machines on Virtualbox
We create one template that we will use then to deply the cluster and to make some performance tests and comparisons

Create the template virtual machine which we will name "template" with 1vCPUs, 1GB of RAM and 25 GB hard disk. 

You can use Ubuntu 24.04.XX LTS server (https://ubuntu.com/download/server)  (PAY ATTENTION TO YOUR ARCHITECURE arm/amd64)

Make sure to set up the network as follows:

 * Attach the downloaded Ubuntu ISO to the "ISO Image".
 * Type: is Lunux
 * Version Ubuntu 24.04 LTS
When you start the virtual machines for the first time you are prompted to instal and setup Ubuntu. 
Follow through with the installation until you get to the “Network Commections”. 
As the VM network protocol is NAT,  the virtual machine will be assinged to an automatic IP (internal) and it will be able to access internet for software upgrades. 

The VM is now accessing the network to download the software and updates for the LTS. 

When you are prompted for the "Guided storage configuration" panel keep the default installation method: use an entire disk. 

When you are prompted for the Profile setup, you will be requested to define a server name (template) and super user (e.g. user01) and his administrative password.


Also, enable openSSH server in the software selection prompt.

Follow the installation and then shutdown the VM.

Inspect the VM and in particular the Network, you will  find only one adapter attached to NAT. If you look at the advanced tab you will find the Adapter Type (Intel) and the MAC address.

Start the newly created machine and make sure it is running. 

Login and update the software:

```
$ sudo apt update
...

$ sudo apt upgrade
```


When your VM is all set up, log in to the VM and test the network by pinging any public site. e.g `ping google.com`. If all works well, this should be successful.

You can check the DHCP-assigned IP address by entering the following command:

```shell
hostname -I
```

You will get an output similar to this:
```
10.0.2.15
```

This is the default IP address assigned by your network DHCP. Note that this IP address is dynamic and can change or worst still, get assigned to another machine. But for now, you can connect to this IP from your host machine via SSH.

Now install some useful additional packages:

```
$ sudo apt install net-tools

$ sudo apt install gcc make 
```

If everything is ok, we can proceed cloning this template. We will create 3 clones (you can create more than 3 
according to the amount of RAM and cores available in your laptop).

You must shutdown the node to clone it, using VirtualBox interface (select VM and right click) create 3 new VMs. 

```
$ sudo shutdown -h now
```


Right click on the name of the VM and clone it. The first clone will be the login/master node the other twos will be computing nodes.

## Configure the nodes

Once the 2 machines has been cloned we can bootstrap the login/master node and configure it.
Add a new network adapter on each machine: enable "Adapter 2" "Attached to" internal network and name it "clustervimnet"

### Login/master node

Bootstrap the VM and configure the secondary network adapter with a static IP. 

In the example below the interface is enp0s8, to find your own one:

```
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:2b:e5:36 brd ff:ff:ff:ff:ff:ff
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:6e:cf:82 brd ff:ff:ff:ff:ff:ff
```

You are interested to link 2 and 3. Link 2 is the NAT device, Link 3 is the internal network device.
You are interested in Link 3.

Now we configure the adapter. To do this we will edit the netplan file:

```
$ sudo vim /etc/netplan/50-cloud-init.yaml

# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s1:
      dhcp4: true
    enp0s8:
     dhcp4: no
     addresses: [192.168.0.1/24]
  version: 2
```

and apply the configuration

```
$ sudo netplan apply
```
We change the hostname:
```
$ sudo vim /etc/hostname

node01
```


Edit the hosts file to assign names to the cluster that should include names for each node as follows:

```
$ sudo vim /etc/hosts

127.0.0.1 localhost
192.168.0.1 master

192.168.0.21 node01
192.168.0.22 node02
192.168.0.23 node03
192.168.0.24 node04
192.168.0.25 node05

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
#### Port forwarding on  master node
To enable ssh from host to guest VM you need to create a port forwarding rule in VirtualBox. 
To do this open 
```
VM settings -> Network -> Advanced -> Port Forwarding 
```

and create a forwarding rule from host to the VM: 
* Name --> ssh 
* Protocol --> TCP
* HostIP --> 127.0.0.1
* Host Port --> 2222
* Guest Port --> 22

Now you should be able to ssh to your VM. Startup the VM, then

```
ssh -p 2222 user01@127.0.0.1
```

but you will have to enter the password. 
If you want a passwordless access you need to generate a ssh key or use an ssh key if you already have it.

If you don’t have public/private key pair already, run ssh-keygen and agree to all defaults. 
This will create id_rsa (private key) and id_rsa.pub (public key) in ~/.ssh directory.

Copy host public key to your VM:

```
scp -P 2222 ~/.ssh/id_rsa.pub user01@127.0.0.1:~
```

Connect to the VM and add host public key to ~/.ssh/authorized_keys:

```
ssh -p 2222 user01@127.0.0.1
mkdir ~/.ssh
chmod 700 ~/.ssh
cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 644 ~/.ssh/authorized_keys
exit
```

Now you should be able to ssh to the VM without password.

#### DHCP and DNS on master node

Then we install a DNSMASQ server to dynamically assign the IP and hostname to the other nodes on the internal interface and create a cluster [1].

Install dnsmasq

```
$ sudo apt install dnsmasq -y
```

To find and configuration file for Dnsmasq, navigate to /etc/dnsmasq.conf. Edit the file by modifying it with your desired configs. Below is minimal configurations for it to run and support minimum operations.

```
$ sudo vim /etc/dnsmasq.conf

port=53
bogus-priv
strict-order
interface=enp0s9
listen-address=::1,127.0.0.1,192.168.0.1
bind-interfaces
log-queries
log-dhcp
dhcp-range=192.168.0.22,192.168.0.28,255.255.255.0,12h
dhcp-option=option:dns-server,192.168.0.1
dhcp-option=3

```

When done with editing the file, close it and modify the resolv.conf file:

```
sudo ln -fs /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Then modify the network configuration (note the nameservers section):

```
$ sudo vim /etc/netplan/50-cloud-init.yaml

network:
    ethernets:
        enp0s8:
            dhcp4: true
        enp0s9:
            dhcp4: false
            addresses: [192.168.0.1/24]
            nameservers:
                addresses: [192.168.0.1]
    version: 2
```


Save this file and modify the /etc/default/dnsmasq: uncomment the following lines:

```
$ x

# If the resolvconf package is installed, dnsmasq will use its output
# rather than the contents of /etc/resolv.conf to find upstream
# nameservers. Uncommenting this line inhibits this behaviour.
# Note that including a "resolv-file=<filename>" line in
# /etc/dnsmasq.conf is not enough to override resolvconf if it is
# installed: the line below must be uncommented.
IGNORE_RESOLVCONF=yes

# If the resolvconf package is installed, dnsmasq will tell resolvconf
# to use dnsmasq under 127.0.0.1 as the system's default resolver.
# Uncommenting this line inhibits this behaviour.
DNSMASQ_EXCEPT="lo"
```


Now you can restart Dnsmasq to apply the changes. 
```
$ sudo systemctl restart dnsmasq systemd-resolved

$ sudo root@master:~# systemctl status dnsmasq
● dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
     Loaded: loaded (/usr/lib/systemd/system/dnsmasq.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-09-25 10:02:04 UTC; 5min ago
    Process: 2673 ExecStartPre=/usr/share/dnsmasq/systemd-helper checkconfig (code=exited, status=0/SUCCESS)
    Process: 2680 ExecStart=/usr/share/dnsmasq/systemd-helper exec (code=exited, status=0/SUCCESS)
    Process: 2687 ExecStartPost=/usr/share/dnsmasq/systemd-helper start-resolvconf (code=exited, status=0/SUCCESS)
   Main PID: 2685 (dnsmasq)
      Tasks: 1 (limit: 2211)
     Memory: 768.0K (peak: 1.6M)
        CPU: 10ms
     CGroup: /system.slice/dnsmasq.service
             └─2685 /usr/sbin/dnsmasq -x /run/dnsmasq/dnsmasq.pid -u dnsmasq -I lo -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new --local-se>

Sep 25 10:02:04 master dnsmasq[2685]: read /etc/hosts - 14 names
Sep 25 10:02:04 master systemd[1]: Started dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server.
Sep 25 10:02:21 master dnsmasq[2685]: query[A] node05 from 192.168.0.1
Sep 25 10:02:21 master dnsmasq[2685]: /etc/hosts node05 is 192.168.0.25
Sep 25 10:02:21 master dnsmasq[2685]: query[AAAA] node05 from 192.168.0.1
Sep 25 10:02:21 master dnsmasq[2685]: forwarded node05 to 10.0.2.3
Sep 25 10:02:21 master dnsmasq[2685]: reply node05 is NODATA-IPv6
Sep 25 10:02:21 master dnsmasq[2685]: query[MX] node05 from 192.168.0.1
Sep 25 10:02:21 master dnsmasq[2685]: forwarded node05 to 10.0.2.3
Sep 25 10:02:21 master dnsmasq[2685]: reply node05 is NODATA
root@master:~# systemctl restart dnsmasq
root@master:~# systemctl status dnsmasq
● dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
     Loaded: loaded (/usr/lib/systemd/system/dnsmasq.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-09-25 10:07:51 UTC; 1s ago
    Process: 2764 ExecStartPre=/usr/share/dnsmasq/systemd-helper checkconfig (code=exited, status=0/SUCCESS)
    Process: 2772 ExecStart=/usr/share/dnsmasq/systemd-helper exec (code=exited, status=0/SUCCESS)
    Process: 2779 ExecStartPost=/usr/share/dnsmasq/systemd-helper start-resolvconf (code=exited, status=0/SUCCESS)
   Main PID: 2777 (dnsmasq)
      Tasks: 1 (limit: 2211)
     Memory: 752.0K (peak: 1.6M)
        CPU: 8ms
     CGroup: /system.slice/dnsmasq.service
             └─2777 /usr/sbin/dnsmasq -x /run/dnsmasq/dnsmasq.pid -u dnsmasq -I lo -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new --local-se>

Sep 25 10:07:51 master dnsmasq[2777]: started, version 2.90 cachesize 150
Sep 25 10:07:51 master dnsmasq[2777]: compile time options: IPv6 GNU-getopt DBus no-UBus i18n IDN2 DHCP DHCPv6 no-Lua TFTP conntrack ipset nft>
Sep 25 10:07:51 master dnsmasq-dhcp[2777]: DHCP, IP range 192.168.0.21 -- 192.168.0.28, lease time 12h
Sep 25 10:07:51 master dnsmasq-dhcp[2777]: DHCP, sockets bound exclusively to interface enp0s9
Sep 25 10:07:51 master dnsmasq[2777]: reading /etc/resolv.conf
Sep 25 10:07:51 master dnsmasq[2777]: ignoring nameserver 192.168.0.1 - local interface
Sep 25 10:07:51 master dnsmasq[2777]: using nameserver 10.0.2.3#53
Sep 25 10:07:51 master dnsmasq[2777]: using nameserver fd00::3#53
Sep 25 10:07:51 master dnsmasq[2777]: read /etc/hosts - 14 names
Sep 25 10:07:51 master systemd[1]: Started dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server.
lines 1-23/23 (END)
```

Do not worry about the "ignoring nameserver 192.168.0.1".


Check if it is working

```
$ host node02

node02 has address 192.168.0.22
```

Reboot  the VM. 

Afther bootstrap may happen the dnsmasq service start before the interfaces, just restart the service:
```
$ sudo systemctl restart dnsmasq systemd-resolved
```

#### A distributed filesystem

To build a cluster we aldo need a distributed filesystem accessible from all nodes. 
We use NFS.

```
$ sudo apt install nfs-kernel-server
$ sudo mkdir /shared
$ sudo chmod 777 /shared
```

Modify the NFS config file:

```
$ sudo vim /etc/exports

/shared/  192.168.0.0/255.255.255.0(rw,sync,no_root_squash,no_subtree_check)

```
Restart the server

```
$ sudo systemctl enable nfs-kernel-server
$ sudo systemctl restart nfs-kernel-server
```

### Other nodes
Bootstrap the VM  node01 and configure the secondary network adapter with a dynamic IP (this should be standard configuration and nothing should me modified, anyway please check with the "ip link show" command to check the name of the adapters). 

To do this we will edit the netplan file:

```
$ sudo vim /etc/netplan/50-cloud-init.yaml

# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s8:
      dhcp4: true
      dhcp4-overrides:
        use-dns: no
    enp0s9:
     dhcp4: true
     dhcp-identifier: mac
     nameservers:
          addresses: [192.168.0.1]
  version: 2
```

and apply the configuration

```
$ sudo netplan apply
```
Empty the /etc/hostname file
```
$ sudo vim /etc/hostname


```

Set the proper dns server (assigned with dhcp):

```
$ sudo ln -fs /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Reboot the machine.
At reboot you will see that the machine will have a new ip address 

```
$ hostname -I
10.0.2.15 192.168.0.23 
```

Now, from the master you will be able to connect to node01 machine with ssh.

```
$ ssh user01@cluster01
user01@cluster02:~$

```
To access the new machine without password you can proceed described above. Run ssh-keygen and agree to all defaults. 
This will create id_rsa (private key) and id_rsa.pub (public key) in ~/.ssh directory.

Copy host public key to your VM:

```
scp  ~/.ssh/id_rsa.pub user01@cluster03:~
```

Connect to the VM and add host public key to ~/.ssh/authorized_keys:

```
ssh user01@cluster03
mkdir ~/.ssh
chmod 700 ~/.ssh
cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 644 ~/.ssh/authorized_keys
exit
```

#### Configure the shared filesystem

```
$ sudo apt install nfs-common
$ sudo mkdir /shared
```

Mount the shared directory adn test it
```
$ sudo mount 192.168.0.1:/shared  /shared
$ touch /shared/pippo
```
If everything will be ok you will see the "pippo" file in all the nodes.

To automatically mount 
```
root@node0x:~# apt -y install autofs
root@node01:~# vi /etc/auto.master
# add to last line
 /-    /etc/auto.mount
```
then

```
root@node01:~# vi /etc/auto.mount
# create new : [mount point] [option] [location]
 /shared   -fstype=nfs,rw  192.168.0.1:/shared     

root@node01:~# systemctl restart autofs
```


## References

[1] Configure DNSMASQ https://computingforgeeks.com/install-and-configure-dnsmasq-on-ubuntu/?expand_article=1

[2] Configure NFS Mounts https://www.howtoforge.com/how-to-install-nfs-server-and-client-on-ubuntu-22-04/

[3] Configure network with netplan https://linuxconfig.org/netplan-network-configuration-tutorial-for-beginners
