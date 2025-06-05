# Install and Configure Apache Cloudstack Private Cloud

![LOGO DTE](https://hackmd.io/_uploads/rJnrQf_Gxe.png)

Kelompok 13 Cloud Computing:
- Nicholas Samosir
- Emir Fateen Haqqi
- Muhammad Nadhif Fasichul Ilmi
- Aqshal Ilham Samudera

Video Installation: 
[![Watch the video](https://img.youtube.com/vi/X6GXFGl09Zc/maxresdefault.jpg)](https://youtu.be/X6GXFGl09Zc)


## Install SSH and configure root password

Installing openssh-server
```
sudo apt update
sudo apt install openssh-server -y
sudo systemctl status ssh
```

configuring root
```
passwd root
#this tutorial will change it to Pa$$w0rd
```

enabling ssh root login
```
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
systemctl restart ssh
```



## Configure Network

Access the file in the netplan directory and edit it
```
cd /etc/netplan
sudo nano ./*.yaml
```

Edit the file
```
# This is the network config written by 'subiquity'
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.1.16/24]  #Your host IP address
      routes:
        - to: default
          via: 192.168.1.1  #Your network gateway
      nameservers:
        addresses: [1.1.1.1,8.8.8.8]
      interfaces: [eno1]  #The ethernet interfaces
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```

Validate the netplan configuration
```
sudo netplan get
```

Apply the netplan configuration
```
sudo netplan generate
sudo netplan apply
reboot
```


## Install Hardware Resource Monitoring Tools
```
apt install htop lynx duf -y
apt install bridge-utils
```
This helps to see and manage what's happening inside your system in real time like CPU, RAM, Disk, Network and Processes.


## Installing Network Services and Text Editor
```
apt-get install openntpd sudo vim tar -y
apt-get install intel-microcode -y
```

- Installing openntpd sets up an NTP client to synchronize the system time with servers across the internet. 
- vim is a powerful text editor used for editing configuration files and code. 
- tar is a tool used to compress and decompress files, and it is commonly used for extracting downloaded archives. 
- intel-microcode is a collection of low-level instructions written in x86 assembly that improve processor functionality and security.

## Cloudstack Installation
```
sudo -i
mkdir -p /etc/apt/keyrings 
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
```

- Switch to root shell.
- Create a directory /etc/apt/keyrings to store cloudstack public key.
- Download the given URL and redirect the output to 'gpg --dearmor' command. 'gpg --dearmor' command will convert from ASCII armored to binary format. And 'sudo tee' command will redirect 'gpg --dearmor' command to file /etc/apt/keyrings/cloudstack.gpg
- Adds the CloudStack APT repository to the system's list of package sources.

### Install Cloudstack-Management
```
apt-get update -y
apt-get install cloudstack-management
```

## Install and Configure MySQL Database
```
apt-get install mysql-server
```

The code below modifies the MySQL server configuration by adjusting settings such as server ID, SQL mode, timeout behavior, maximum connections, binary logging, and log format. These changes impact the server's behavior, performance, and replication capabilities.

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'
```

Sure! Here's a more concise version of the explanations:

- **[mysqld]**: Marks settings for the MySQL server.
- **server-id = 1**: Unique ID for the server, used in replication.
- **sql-mode**: Defines SQL behavior, enabling strict and validation rules.
- **innodb\_rollback\_on\_timeout = 1**: Rolls back transactions if they time out.
- **innodb\_lock\_wait\_timeout = 600**: Waits up to 600 seconds for a lock before timing out.
- **max\_connections = 1000**: Allows up to 1000 simultaneous connections.
- **log-bin = mysql-bin**: Enables binary logging for replication/recovery.
- **binlog-format = 'ROW'**: Logs changes at the row level for detailed replication.

Restart mysql service to apply changes.
```
systemctl restart mysql
```

## Deploy Database
After install and configure database, deploy the database as root and then create cloud user with password. 

**Change the IP Address to your own device's IP Address. Because our CloudStack management server and database is on the same machine so we use localhost**
```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:password -i localhost
```

## Configure Storage

### Config Primary and Secondary Storage
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

### Config NFS Server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

## Configure Cloudstack Host with KVM Hypervisor
In this setup, Install the necessary packages for QEMU/KVM and the CloudStack agent. Next, we'll update the `/etc/libvirt/qemu.conf` file to allow the VNC server to listen on all network interfaces. Then, we'll modify the `/etc/default/libvirtd` file to enable the libvirtd service to listen for connections, either by adding or uncommenting a specific line based on the Ubuntu version.

### Install KVM, Cloudstack Agent, and Configure KVM Virtualization Management

```
apt-get install qemu-kvm cloudstack-agent -y

sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
```

### Configure Default Libvirtd Config
**libvirtd** is a background service that manages virtual machines (VMs), virtual networks, and storage across different virtualization platforms like KVM, QEMU, and Xen. It's a component of the **libvirt** project, which provides a unified toolkit for handling various virtualization technologies. The main goal of libvirtd is to offer a secure and consistent API for managing VMs and related resources, independent of the specific virtualization backend.

```
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf
```

- First line will disable TLS listening on libvirt daemon
- Second line will enable TCP listening on libvirt daemon
- Thrid line will specify a port where daemon will listen for a TCP connection
- Fourth line will disable multicast DNS, libvirtd service will not found through nDNS
- Last, disable authentication for TCP connection to libvirt daemon

### Restart Libvirtd
```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

## Configuration to Support Docker and Other Services
```
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

## Generate Unique Host ID
The following code generates the host ID using the uuid package. It then generates a universally unique identifier (UUID) using the uuid command and assigns it to the variable $UUID. The generated host ID then stored to the /etc/libvirt/libvirtd.conf file.
After adding the host ID to the configuration file, we need to restart the libvirtd service for the changes to take effect. 
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```
## Configure Iptables Firewall and Make it persistent
**Change the INETWORK to your NETWORK**
```
NETWORK=192.168.106.0/23
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT

apt-get install iptables-persistent
#just answer yes yes
```

## Disable Apparmour on Libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

## Launch Management Server
```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log
```

## Access Cloudstack Dashboard
Open your web browser and type:
```
http://<YOUR_IP_ADDRESS>:8080
```
You should see this cloudstack dashboard
![Screenshot 2025-05-13 130617](https://hackmd.io/_uploads/SkdvWwl-eg.png)

## Completing CloudStack Installation
Next, on the dashboard we'll see **"Continue with Installation"** and then follow the following steps. Change IP Address as per your requirement.

### Section 1 and 2
For section 1 and 2, you can just follow the screenshot below.
![Screenshot 2025-05-13 202710](https://hackmd.io/_uploads/r1wetTlbgg.png)

### Section 3 (Zone details)
- Start by put name as "NEW-ZONE" or anything as you want for the "Name" 
- Put "IPv4 DNS", you can use 8.8.8.8 (google DNS) 
- Put your network default gateway as "Internal DNS 1"
- Set KVM as the "Hypervisor".  

### Section 4 (Network)
- Set the default gateway as Gateway and the netmask. 
- Start IP and End IP is use as reserved IP Address for instances. 
![Screenshot 2025-05-13 202718](https://hackmd.io/_uploads/BJpxFalbee.png)

Continue for Section 4-for the Pod sub-section
- Set the Pod name 
- The "reserved system gateway" and "reserved system netmask" can just follow it same as Zone (previous section). 
- For the IP Address it should be different (you can continue after the End IP of the Zone).
- Last for the guest traffic, you can use VLAN 3300 - 3399. 
![Screenshot 2025-05-13 202728](https://hackmd.io/_uploads/SyfZFpeZgl.png)

### Section 5 (Add resources)
This process begins with defining a cluster and specifying the host IP address. Start by creating a cluster. A cluster is used to group hosts that have identical hardware, run the same hypervisor, are on the same subnet, and share the same storage.
- Enter the Cluster's name.
- Enter the host IP Address as the "Host Name".
- Set the username (you can use 'root') and password.
![Screenshot 2025-05-13 202735](https://hackmd.io/_uploads/HkdWYaxbxl.png)

Continue for Section 5
Next step is to configure Primary Storage. 
- Set the name of the primary storage.
- Set the "Scope" with the name of the Zone you've set up earlier.
- Set the "Protocol" into NFS. 
- Set the "Provider" with DefaultPrimary.
- Specify the "Server" with your server storage's IP Address.
- Set the /export/primary as you "Path" for primary storage.
![Screenshot 2025-05-13 202741](https://hackmd.io/_uploads/Sk3-Kag-lx.png)

Next step is to configure Secondary Storage.
- Select NFS as the "Provider"
- Set the name of the secondary storage.
- Specify the "Server" with your server storage's IP Address.
- Set the /export/secondary as you "Path" for secondary storage.
![Screenshot 2025-05-13 202747](https://hackmd.io/_uploads/Sy-fYpxWgx.png)

## Installing ISO
1. On Images tab, go click the "ISOs" and then click the "Register ISO".
![Screenshot 2025-05-13 213713](https://hackmd.io/_uploads/Hk-StAlZll.png)
2. Register the ISO you want. 
- URL: put the URL that leads into the .iso file 
- Name: set the name of the ISO
- Description: set the description for the ISO
- Zone: select the zone that you want/needed for the ISO
- OS type: select the OS type and make sure to set it correctly (double check with the url)
![Screenshot 2025-05-13 213804](https://hackmd.io/_uploads/BJ9IKAlZlg.png)

## Add Network
![Screenshot 2025-05-13 214337](https://hackmd.io/_uploads/HJ135AxZee.png)
![Screenshot 2025-05-13 214253](https://hackmd.io/_uploads/S1a_qCgWle.png)

Change the **Name**, **Zone**, **Gateway**, **Netmask** and **DNS** to your needed. 

## Create Instance
Next, we will create instance. 
1. Open the instance tab in the Compute section. Then click the "Add Instance" button.
![Screenshot 2025-05-13 214439](https://hackmd.io/_uploads/SygWoAe-xe.png)
2. Setup the instance. For Zone, Pod, and Cluster, use the one that you set up earlier. And for the ISO, you can use the one that you set up earlier too. 
![Screenshot 2025-05-13 214654](https://hackmd.io/_uploads/SyWuj0eZll.png)
3. Selecting for the VM computer like compute (CPU) and Disk size.
![Screenshot 2025-05-13 214724](https://hackmd.io/_uploads/HyHqo0e-le.png)
4. Select the network that you set up earlier and adjust it according to your needed. 
![Screenshot 2025-05-13 214729](https://hackmd.io/_uploads/rJHcsClZxl.png)
5. Click the "Launch Instance" button
6. After clicked the "Launch Instance", we'll see the Instance running on the "Instances" tab. 
![Screenshot 2025-05-13 214858](https://hackmd.io/_uploads/B1YJhCe-eg.png)

## Access Instance

![Screenshot 2025-05-13 122505](https://hackmd.io/_uploads/BJKf_Ie-ee.png)
![instance](https://hackmd.io/_uploads/r1j3Z7nbee.png)
![ssh instance](https://hackmd.io/_uploads/SkxTbQ3Wxe.png)
