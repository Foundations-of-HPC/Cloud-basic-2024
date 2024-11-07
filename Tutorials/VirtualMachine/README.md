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

You can use Ubuntu 24.04.XX LTS server (https://ubuntu.com/download/server)  (PAY ATTENTION TO YOUR ARCHITECURE arm64/amd64)

Make sure to set up the network as follows:

 * Attach the downloaded Ubuntu ISO to the "ISO Image".
 * Type: is Lunux
 * Version Ubuntu 24.04 LTS

When you configure the VM you can choose the authomatic installation, in this case set a user and a password and a hostname.
Sometimes the authomatic installation is not working in this case use the manual one as discussed below.

In case of manual installation, 
when you start the virtual machines for the first time you are prompted to instal and setup Ubuntu. 
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

You will get an output similar (may be different according to the OS of the host machine) to this:
```
10.0.2.15
```

This is the default IP address assigned by your network DHCP. Note that this IP address is dynamic and can change or worst still, get assigned to another machine. But for now, you can connect to this IP from your host machine via SSH.

Now install some useful additional packages:

```
$ sudo apt install net-tools

$ sudo apt install gcc make 
```

If everything is ok, we can proceed cloning this template. We will create 2 clones (Master and node prototype).

You must shutdown the node to clone it, using VirtualBox interface (select VM and right click) create 2 new VMs. 

```
$ sudo shutdown -h now
```


Right click on the name of the VM and clone it. 
The first clone will be the login/master node the other twos will be computing nodes.

## Configure the VBOx internal network.

Open Tools-> Network, on the Host-only-network panel create a new one. Call it CloudBasicNet with:

```
Mask: 255.255.255.0
Lower Bound: 192.168.56.2
Upper Bound: 192.168.56.199
```
Note that the lower bound starts from x.x.x.2 and not from x.x.x.1.

## Configure the nodes

Once the 2 machines has been cloned we add a new network adapter on each machine: enable "Adapter 2" "Attached to" internal network and name it "CloudBasicNet".

### Port forwarding 
To enable ssh from host to guest VM you need to create a port forwarding rule in VirtualBox. 
To do this open 
```
VM settings -> Network -> Advanced -> Port Forwarding 
```

create a forwarding rule from host to the Master VM: 
* Name --> ssh 
* Protocol --> TCP
* HostIP --> 127.0.0.1
* Host Port --> 3022
* Guest Port --> 22


and to the Node1 VM
create a forwarding rule from host to the Master VM: 
* Name --> ssh 
* Protocol --> TCP
* HostIP --> 127.0.0.1
* Host Port --> 4022
* Guest Port --> 22

Then you should be able to ssh to your VM. 

### Master node

Bootstrap the VM and ssh from a terminal of your Host Machine (if you like).


```
ssh -p 3022 user01@127.0.0.1
```

If you want a passwordless access you need to generate a ssh key or use an ssh key if you already have it.

If you don’t have public/private key pair already, run ssh-keygen and agree to all defaults. 
This will create id_rsa (private key) and id_rsa.pub (public key) in ~/.ssh directory.

Copy host public key to your VM:

```
scp -P 3022  ~/.ssh/id_rsa.pub user01@127.0.0.1:~
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


### Configure the  network 
In the example below the interface is enp0s8, to find your own one:

```
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:2b:e5:36 brd ff:ff:ff:ff:ff:ff
3: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:6e:cf:82 brd ff:ff:ff:ff:ff:ff
```

You are interested to link 2 and 3. Link 2 is the NAT device for external connectivity, 
Link 3 is the internal network device.
We configure  Link 3.

Now we configure the adapter, it will be the master so it will have 192.168.56.1 IP address.
To do this we will edit the netplan file:

```
$ sudo vim /etc/netplan/50-cloud-init.yaml

# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s8:
      dhcp4: true
    enp0s9:
     dhcp4: false
     addresses: [192.168.56.1/24]
  version: 2
```

check the indentation and apply the configuration

```
$ sudo netplan apply
```
We change the hostname:
```
$ sudo vim /etc/hostname

master
```


Edit the hosts file to assign names to the cluster that should include names for each node as follows:

```
$ sudo vim /etc/hosts

127.0.0.1 localhost
127.0.1.1 master
192.168.56.1 master

192.168.56.2 node02
192.168.56.3 node03
192.168.56.4 node04
192.168.56.5 node05
192.168.56.6 node06
192.168.56.7 node07
192.168.56.8 node08
192.168.56.9 node09
192.168.56.10 node10

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```


#### DNS on master node

Then we install a DNSMASQ server to dynamically assign the IP and hostname to the other nodes on the internal interface and create a cluster [1].

Install dnsmasq

```
$ sudo apt install dnsmasq -y
```

To find and configuration file for Dnsmasq, navigate to /etc/dnsmasq.conf. Edit the file by modifying it with your desired configs. Below is minimal configurations for it to run and support minimum operations.

```
$ sudo vim /etc/dnsmasq.conf


bogus-priv
bind-interfaces
cache-size=1000
resolv-file=/etc/resolv.dnsmasq
listen-address=127.0.0.1, 192.168.56.1
log-queries

```

When done with editing the file, close it and modify the resolv.conf file:

```
sudo unlink /etc/resolv.conf

sudo vim /etc/resolv.conf

nameserver 127.0.0.1
options edns0 trust-ad
search .

```
Configure the resolve file for external access

```
$ sudo ln -s  /run/systemd/resolve/resolv.conf /etc/resolv.dnsmasq

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

$ host 192.168.56.4
4.56.168.192.in-addr.arpa domain name pointer node04.
```

Enable the service and Rreboot  the VM:

```
$ sudo systemctl enable dnsmasq
Synchronizing state of dnsmasq.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable dnsmasq

$ sudo reboot
```
Afther bootstrap may happen the dnsmasq service start before the interfaces, just restart the service (this is a known preoblem for ubuntu):
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

/shared/  192.168.56.0/255.255.255.0(rw,sync,no_root_squash,no_subtree_check)

```
Restart the server

```
$ sudo systemctl enable nfs-kernel-server
$ sudo systemctl restart nfs-kernel-server
```
prepare some directories to share:
```
$ sudo mkdir /shared/data /shared/home
$ sudo chmod 777  /shared/data /shared/home
```

### The nodes
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
          addresses: [192.168.56.1]
  version: 2
```
check the indentation and apply the configuration

```
$ sudo netplan apply
```
Empty the /etc/hostname file
```
$ sudo vim /etc/hostname


```

Set the proper dns server (assigned with dhcp):

```
$ sudo unlink   /etc/resolv.conf

$ sudo vim /etc/resolv.conf

nameserver 192.168.56.1
search .
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
$ sudo apt install nfs-common -y
$ sudo mkdir /shared
```

Mount the shared directory adn test it
```
$ sudo mount 192.168.56.1:/shared  /shared

$ df -h
Filesystem            Size  Used Avail Use% Mounted on
tmpfs                 196M 1016K  195M   1% /run
efivarfs              256K  6.5K  250K   3% /sys/firmware/efi/efivars
/dev/sda2              24G  2.5G   20G  11% /
tmpfs                 978M     0  978M   0% /dev/shm
tmpfs                 5.0M     0  5.0M   0% /run/lock
/dev/sda1             1.1G  6.4M  1.1G   1% /boot/efi
tmpfs                 196M   12K  196M   1% /run/user/1000
192.168.56.1:/shared   24G  2.5G   20G  11% /shared
```

Check that you have R/W access:
```
$ touch /shared/pippo
```
If everything will be ok you will see the "pippo" file in all the nodes.
Unmount the mount pont:
```
$ sudo umount /shared/
```

To automatically mount 
```
$ sudo apt -y install autofs
$ sudo  vim /etc/auto.master
# add to last line
/shared    /etc/auto.shared
```
then

```
$ sudo vim /etc/auto.mount
# create new : [mount point] [option] [location]
data    192.168.56.1:/shared   


$ sudo  systemctl restart autofs
```

## Create a load balancing service

In this tutorial, we are going to create a load balancing service on VMs.

* Only the master VM is capable of connecting to the internet and able to connect with each other privately (we expose only one machine with no data)
* Nodes hosts the service


## Install the load balancer on Master

To enable http access from host to guest VM you need to create a port forwarding rule in VirtualBox. 
To do this open 
```
VM settings -> Network -> Advanced -> Port Forwarding 
```

create a forwarding rule from host to the Master VM: 
* Name --> http 
* Protocol --> TCP
* HostIP --> 127.0.0.1
* Host Port --> 8080
* Guest Port --> 80


### Configuring nginx as a load balancer

```
$ sudo apt-get -y install nginx
$ sudo systemctl start nginx.service
$ sudo systemctl status nginx.service
```
Check from your browser that you can connect to http://localhost:8080...if ok nginx is running.

When nginx is installed and tested, start to configure it for load balancing. In essence, all you need to do is set up Nginx with instructions on which type of connections to listen to and where to redirect them. 

Stop the service and disable the default configuration
```
$ sudo systemctl stop nginx.service
$ sudo  rm /etc/nginx/sites-enabled/default
```

Create a new configuration file using whichever text editor you prefer. 

NOTICE: Check the IP addresses of the nodes that you create, new nodes will have new ip addresses. The internal DHCP server is not going to reuse the old ones!!!!

```
sudo vim  /etc/nginx/conf.d/load-balancer.conf

# Define which servers to include in the load balancing scheme. 
# It's best to use the servers' private IPs for better performance and security.
# You can find the private IPs at your UpCloud control panel Network section.
   upstream backend {
      server 192.168.56.7:5001; 
      server 192.168.56.8:5001;
      server 192.168.56.9:5001;
   }

   # This server accepts all traffic to port 80 and passes it to the upstream. 
   # Notice that the upstream name and the proxy_pass need to match.

   server {
      listen 80; 

      location / {
          proxy_pass http://backend;
      }
   }
```
### Configure the node template

Install python Virtual Env,
```
sudo apt install python3.12-venv
```

As user create a virtual enviroment to install Flask:

```
user01@template:~$ python3 -m venv env
user01@template:~$ source env/bin/activate
(env) user01@template:~$ pip install Flask
...
```

Test with a very simple Flask app

```
(env) user01@template:~$ mkdir app
(env) user01@template:~$ vim app/app01.py

from flask import Flask

app = Flask(__name__)
counter = 0  # Global counter for requests

@app.route('/', methods=['POST'])
def increment_counter():
    global counter
    counter += 1
    return f"Server has processed {counter} requests!"
```

Run the app (NOTE THAT THE HOST SHOULD BE THE NODE HOST IP)

```
(env) user01@template:~$ flask --app app/app01.py run --port 5001 --host 192.168.56.7
 * Serving Flask app 'app/app01.py'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://192.168.56.7:5001
Press CTRL+C to quit
```

Now you can test the configuration, from you host machine:

```
$ curl -X POST http://localhost:8080
```

Question: why is it working? Are you sure it is really working?

If it is working you can stop the machine and clone it 3 times.

## Test the configuration

Bootstrap the 3 nodes. And in each one of them activate the virtual env and run the application:

```
user01@template:~$ source env/bin/activate
(env) user01@template:~$ flask --app app/app01.py run --port 5001 --host 192.168.56.X
```

NOTE: change the IP address to the node IP.

Now you can test the configuration, from you host machine:

```
$ curl -X POST http://localhost:8080
```

QUESTION: is it acting as expected? If not why?

### External caching service

Redis is an in-memory data store that can be used as a cache backend. By using Flask and Redis together, you can cache the results of expensive computations and serve them quickly to users.

Select one of the nodes and install redis service:
```
$ sudo apt-get install redis-server
```

configure the server to bind to the proper IP address of the node

```
vim /etc/redis/redis.conf

...
bind 192.168.56.9 127.0.0.1
...
```
Restart the redis service

```
$ sudo systemctl restart redis
```

Now we can use a redis enabled version of the program:

```
from flask import Flask
import redis
import socket

app = Flask(__name__)

# Initialize Redis client
r = redis.Redis(host='node09', port=6379, db=0)

# Set initial value if not set
if not r.exists("counter"):
    r.set("counter", 0)

def get_ip():
    hostname = socket.gethostname()
    ip_address = socket.gethostbyname(hostname)
    return ip_address

@app.route('/', methods=['POST'])
def increment_counter():
    # Increment the counter in Redis
    new_counter = r.incr("counter")
    # Get the IP address of the current node
    node_ip = get_ip()
    return f"Server (IP: {node_ip}) has processed {new_counter} requests!"


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5001)
```

Now you can test again, from your Host machine:

```
$ curl -X POST http://localhost:8080
```

QUESTION: is it acting as expected? If not why?



## References

[1] Configure DNSMASQ https://computingforgeeks.com/install-and-configure-dnsmasq-on-ubuntu/?expand_article=1

[2] Configure NFS Mounts https://www.howtoforge.com/how-to-install-nfs-server-and-client-on-ubuntu-22-04/

[3] Configure network with netplan https://linuxconfig.org/netplan-network-configuration-tutorial-for-beginners
