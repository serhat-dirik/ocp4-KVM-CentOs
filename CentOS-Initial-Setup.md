# CentOS Initial Steps

 I preferred to use CentOS 8 minimal deployment as my operating system setup for my OpenShift 4 Cluster deployment.  Below are some recommended configuration steps before you start KVM & OCP deployment. I used a dedicated bare-metal server on a cloud provider for my setup and needed to access it as a remote server. If you're planning to deploy OCP 4 on your local environment/laptop, you can safely skip some steps defined below.    

### Update your OS and Install some useful utilities
```shell
dnf check-update
dnf update
```
Install some useful utilities
```shell
dnf install -y nano vim wget curl net-tools lsof bash-completion zip unzip psmisc bind-utils cockpit jq 
```
After update process finishes, release the disk space
```shell
dnf clean all
```

###  Change the hostname
```shell
hostnamectl set-hostname new-hostname
# check the hostname
cat /etc/hostname
# change the hostname in hosts
vi /etc/hosts
```

### Check Network  
```shell
ifconfig enp98s0f0
ip a
ping -c2 google.com
#check speed
ethtool enp98s0f0
mii-tool enp98s0f0
#check port and who uses
netstat -tulpn
ss -tulpn
lsof -i4 -6
```
change dns if needed
```shell
nmcli con mod "enp98s0f0" ipv4.dns "8.8.8.8 8.8.4.4"
nmcli c show id enp98s0f0
```

Rename you nic if needed. Follow [this guide](https://www.itzgeek.com/how-tos/linux/centos-how-tos/how-to-change-network-interface-name-to-eth0-on-centos-8-rhel-8.html) 
### Set Time Zone

```shell
# list zones
timedatectl list-timezones
# set timezone
timedatectl set-timezone region/timezone
```
Set NTP & NTP Servers for automatic time update
```shell
dnf install chrony
vi /etc/chrony.conf
#put ntp server in config
allow 192.168.56.0/24
#start the service
systemctl start chronyd
systemctl status chronyd
systemctl enable chronyd
# Check sources
chronyc sources
```
### Add a sudoer!
Using root account all the time is not good security practice. Define a new user and make it a sudoer.

```shell
useradd your-user
passwd your-user
usermod -aG wheel your-user
```
Enable passwordless sudo for the user in /etc/sudoers.d, run visudo
```shell
your-user ALL=(ALL) NOPASSWD:ALL
```
### Enable Passwordless ssh
On the client side, if you don't have certs created yet create one

```shell
ssh-keygen -t rsa -b 4096 -N '' -C "your-user@mailserver.com" -f ~/.ssh/id_rsa
```
Copy your id to server

 ```shell
 ssh-copy-id username@serverip
 ```

### Remove undesired services
Fresh CentOS/RHEL 8 server may come with some undesired services.You need to remove and disable to reduce the attacks on the server.Check services

 ```shell
ss -tulpn
# or
netstat -tulpn
```
You can also run ps, top or pstree commands to discover and identify all services.

```shell
dnf install psmisc
pstree -p
```
Check installed services
```shell
systemctl list-units
systemctl list-unit-files -t service
```

Example: removing postfix mail server

```shell
systemctl stop postfix
systemctl disable postfix
dnf remove postfix
```


### Enable Firewall

Firewalld comes as default service with CentOS. Before you enable the firewall service, make sure of that required ports are open.

```shell
#enable ssh first
firewall-offline-cmd --add-service=ssh
systemctl enable firewalld
systemctl start firewalld
systemctl status firewalld
```
If you want to incoming connections to some common network services such as HTTP or SMTP, just add the rules as shown by specifying the service name.

```shell
firewall-cmd --add-service=http
firewall-cmd --add-service=https
firewall-cmd --add-service=smtp
firewall-cmd --add-service=ntp
#Push to permanent config
firewall-cmd --runtime-to-permanent
#Reload and check
firewall-cmd --reload
# Check if all defined and in use
firewall-cmd --list-all
```

### Install Cockpit
Cockpit is an awesome tool to manage your system through browser

```shell
#Install
dnf install -y cockpit
#Start and enable
systemctl start cockpit
systemctl enable cockpit.socket
#Enable firewall access
firewall-cmd --add-service=cockpit
firewall-cmd --add-service=cockpit --permanent

```
At this point, you should be able to access cockpit from https://SERVER_IP:9090 URL.


### Secure SSH service

 Its a wise decision to change the default ssh port for extra security.  

```shell
vi /etc/ssh/sshd_config
```
Add/Change the port definition in the config file

```shell
# SSH Port
Port 2122  # the port you want to change it to
```
Update the firewall, to allow traffic on the new port  
```shell
firewall-cmd --add-port 2122/tcp
firewall-cmd --runtime-to-permanent
  or
firewall-cmd --add-port 2122/tcp --permanent
```

And consider removing ssh for root account

```shell
vi /etc/ssh/sshd_config
```
Add/modify line
```shell
PermitRootLogin no
#  OR
PermitRootLogin without-password
```
The configuration changes are now finished. Restart the SSH server (SSHD)...
```shell
service sshd restart
```
