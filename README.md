# Install and Configure Apache Cloudstack Private Cloud

People behind the scenes from Kelompok 13:
- Aqshal Ilham Samudera
- Emir Fateen Haqqi
- Muhammad Nadhif Fasichul Ilmi
- Nicholas Samosir

## Install SSH and confirm it
```
sudo apt update
sudo apt install openssh-server -y
sudo systemctl status ssh
```
We use tailscale to remote SSH from our windows device.


## Install Hardware Resource Monitoring Tools
```
apt update -y
apt upgrade -y
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
NETWORK=172.16.0.0/24
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

![Screenshot 2025-05-13 202710](https://hackmd.io/_uploads/r1wetTlbgg.png)
![Screenshot 2025-05-13 202718](https://hackmd.io/_uploads/BJpxFalbee.png)
![Screenshot 2025-05-13 202728](https://hackmd.io/_uploads/SyfZFpeZgl.png)
![Screenshot 2025-05-13 202735](https://hackmd.io/_uploads/HkdWYaxbxl.png)
![Screenshot 2025-05-13 202741](https://hackmd.io/_uploads/Sk3-Kag-lx.png)
![Screenshot 2025-05-13 202747](https://hackmd.io/_uploads/Sy-fYpxWgx.png)

## Installing ISO
On Images tab, go click the "ISOs" and then click the "Register ISO".
![Screenshot 2025-05-13 213713](https://hackmd.io/_uploads/Hk-StAlZll.png)
Register the ISO you want. We use ubuntu 22.04.5 live server. for the "Zone", use the "Zone" that we've configured earlier. 
![Screenshot 2025-05-13 213804](https://hackmd.io/_uploads/BJ9IKAlZlg.png)

## Add Network
![Screenshot 2025-05-13 214337](https://hackmd.io/_uploads/HJ135AxZee.png)
![Screenshot 2025-05-13 214253](https://hackmd.io/_uploads/S1a_qCgWle.png)

Change the **Name**, **Zone**, **Gateway**, **Netmask** and **DNS** to your needed. 

## Create Instance
![Screenshot 2025-05-13 214439](https://hackmd.io/_uploads/SygWoAe-xe.png)
![Screenshot 2025-05-13 214654](https://hackmd.io/_uploads/SyWuj0eZll.png)
![Screenshot 2025-05-13 214724](https://hackmd.io/_uploads/HyHqo0e-le.png)
![Screenshot 2025-05-13 214729](https://hackmd.io/_uploads/rJHcsClZxl.png)

After clicked the "Launch Instance", we'll see the Instance running on the "Instances" tab. 
![Screenshot 2025-05-13 214858](https://hackmd.io/_uploads/B1YJhCe-eg.png)

## Access Instance

![Screenshot 2025-05-13 122505](https://hackmd.io/_uploads/BJKf_Ie-ee.png)
![Screenshot 2025-05-13 215931](https://hackmd.io/_uploads/ryJORAe-gx.png)





