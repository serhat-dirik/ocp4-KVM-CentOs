# OpenShift 4 installation on KVM
Updated for OCP Version: 4.3 , Tested on CentOS 8.1 minimal

This is my personal notes on OCP 4 cluster installation on a KVM. The steps below used to install the required stack on a bare metal, dedicated server that I got from one hosting providers. Although my goal was exposing my services to public internet access, I believe that you can use this guideline to deploy OCP4 on your laptop as skipping some steps.
Thanks to Khizer Naeem's publishing OCP UPI deployment workshop and also OpenShift public bloggers. I heavily used their stuff and copied a lot pulling it all together.

### Prerequsities:

1. If you're planning to expose your OCP deployment to public internet access, you'd need two DNS records as:
*   A wildcard record to access your OpenShift Apps*.apps.your-cluster.your-domain.com
*   A DNS record to access your Kubernetes API api.your-cluster.your-domain.com
2. Firewall Config:
    Hosting providers usually provides a firewall service along with your server. Do not forget to configure it. Here is the minimum list of ports that you need to open: your-ssh-port, 80,443,9090(cockpit),6443(API)  
3. Minimal RHEL or CentOs Setup on your host. I used CentOS 8 minimal setup with this guideline. If you're new in CentOS deployment,[CentOs initial steps](CentOS-Initial-Setup.md) guide might be helpful to configure some required services config such as SSH and Firewall

## Step 1: LibVirt Install

 If you already have libvirt installed, you can perfectly skip this step. If you have a minimal OS deployment, than most probably you don't have one, so follow this steps to setup and enable libvirt. Check if virtualization enabled on your server.

```shell
egrep -c '(vmx|svm)' /proc/cpuinfo
# VCPU num should be more than 1
lscpu | grep Virtualization:
# Virtualization:      VT-x
```
If virtualization is not enabled on your server, I recommend do not go any further and talk to your hosting provider to enable it first. Install libvirt along with some required libraries and start it's service:
```shell
dnf install qemu-kvm qemu-img libvirt virt-install libvirt-client virt-manager libvirt-daemon-driver-network libguestfs-tools network-scripts -y

systemctl start libvirtd
systemctl enable libvirtd

```
Once your packages has been installed successfully, then run the below command to confirm whether KVM module has been loaded into the kernel or not,
```shell
lsmod | grep -i kvm
kvm_intel             245760  0
kvm                   745472  1 kvm_intel
irqbypass              16384  1 kvm

```
Check if Virtual bridge if defined

```shell
nmcli connection show

NAME           UUID                                  TYPE      DEVICE
System enp4s0  b325fd44-30b3-c744-3fc9-e154b78e8c82  ethernet  enp4s0
virbr0         9aec066b-abb3-43d8-bfd8-b5136603d6d5  bridge    virbr0
```

### Step 1.1  Optionally install a new libvirt network. 

```shell
# Pick a random subnet octet (192.168.XXX.0) for this network
NET_OCT="133"
/usr/bin/cp /usr/share/libvirt/networks/default.xml /tmp/new-net.xml
sed -i "s/default/ocp-${NET_OCT}/" /tmp/new-net.xml
sed -i "s/virbr0/ocp-$NET_OCT/" /tmp/new-net.xml
sed -i "s/122/${NET_OCT}/g" /tmp/new-net.xml
virsh net-define /tmp/new-net.xml
virsh net-autostart ocp-${NET_OCT}
virsh net-start ocp-${NET_OCT}
systemctl restart libvirtd
````

## Step 2: Configure Dnsmasq for OCP

Enable dnsmasq that embedded in Network Manager

```shell

echo -e "[main]\ndns=dnsmasq" > /etc/NetworkManager/conf.d/00-dns.conf

cat << 'EOF' | sudo tee /etc/NetworkManager/dnsmasq.d/02-add-hosts.conf
# /etc/NetworkManager/dnsmasq.d/02-add-hosts.conf
# By default, the plugin does not read from /etc/hosts.  
# This forces the plugin to slurp in the file.
#
# If you didn't want to write to the /etc/hosts file.  This could
# be pointed to another file.
#
addn-hosts=/etc/hosts
EOF

firewall-cmd --add-service=dns --permanent

firewall-cmd --reload

systemctl reload NetworkManager
systemctl status NetworkManager
```
At this point dnsmasq should be enabled. Verify if dnsmasq is working

```shell
dnf -y install bind-utils

dig +short example.com @127.0.0.1

time getent hosts foo.example.com

# we should get lesser time on second execution
time getent hosts foo.example.com

```
Log dns queries for verification
```shell
echo log-queries | sudo tee -a /etc/NetworkManager/dnsmasq.d/log.conf
sudo systemctl reload NetworkManager

dig +short example.com @127.0.0.1
tail /var/log/messages
```
Turn off logging
```shell
sudo rm /etc/NetworkManager/dnsmasq.d/log.conf
sudo systemctl reload NetworkManager
```
[for more information about dnsmasq](https://www.getpagespeed.com/server-setup/dns-caching-and-beyond-in-centos-rhel-7-and-8)


## Step 3: Download and Prepare OCP Files

Prepare a directory for OCP deployment and download files.

```bash
mkdir ocp4 && cd ocp4
mkdir rhcos-install
# Download CoreOS
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/4.3.0/rhcos-4.3.0-x86_64-installer-kernel -O rhcos-install/vmlinuz

wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/4.3.0/rhcos-4.3.0-x86_64-installer-initramfs.img -O rhcos-install/initramfs.img

#Generate treeinfo
cat <<EOF > rhcos-install/.treeinfo
[general]
arch = x86_64
family = Red Hat CoreOS
platforms = x86_64
version = 4.3.0
[images-x86_64]
initrd = initramfs.img
kernel = vmlinuz
EOF
```
Download the bios image for Red Hat CoreOS.

```shell
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.3/4.3.0/rhcos-4.3.0-x86_64-metal.raw.gz
```

Download the installer and client binaries. You may want to check if there's a newer version published before you download.

```shell

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.0/openshift-client-linux-4.3.0.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.3.0/openshift-install-linux-4.3.0.tar.gz

tar xf openshift-client-linux-4.3.0.tar.gz

tar xf openshift-install-linux-4.3.0.tar.gz

chmod +x oc
chmod +x openshift-install
chmod +x kubectl
mv -f oc /usr/bin
mv -f kubectl /usr/bin
mv -f openshift-install /usr/bin
rm -f README.md
```

## Step 4: Environment Setup

By default libvirt has only one "default" network. You can find out libvirt's networks by running `virsh net-list`. If you like you can create another one and use it for your OCP deployment. I'm using the `default`network in this example.  

```shell

VIR_NET="default"
HOST_NET=$(ip -4 a s $(virsh net-info $VIR_NET | awk '/Bridge:/{print $2}') | awk '/inet /{print $2}')
HOST_IP=$(echo $HOST_NET | cut -d '/' -f1)
# Set the dnsmasq configuration directory.
DNS_DIR="/etc/NetworkManager/dnsmasq.d"
#using a separate dnsmasq installed on the host set it to "/etc/dnsmasq.d"
BASE_DOM="your-domain"
CLUSTER_NAME="your-cluster-name"
# DNS Records that might be required: *.apps.your-cluster.your-domain, api.your-cluster.your-domain.com
```
Pick your SSH public key (file).Feel free to generate a new one
```shell
ssh-keygen -t rsa -b 4096 -N '' -C "yourmail@srv.com" -f ~/.ssh/id_rsa
```

This SSH public key will be copied over to Red Hat CoreOS and RHEL VMs for SSH access.

```shell
SSH_KEY="/root/.ssh/id_rsa.pub"
```
Download pull secret from [Red Hat OpenShift Cluster Manager](https://cloud.redhat.com/openshift/install/metal/user-provisioned).
You can also copy paste the pull secret.
The variable PULL_SEC should have your pull secret without any newlines.

If you are copy-pasting the value of pull secret, its crucial to use single quotes (not double quotes).

```shell
# If loading from downloaded file
PULL_SEC=$(cat </root/ocp4/pull-secret.txt)
# If copy-pasting, use single quotes
PULL_SEC='<paste-pull-secret>'
#Set the Red Hat Customer Portal credentials.
RHNUSER='<your-rhn-user-name>'

RHNPASS='<your-rhn-password>'
```

! OCP Ports need to be added to the firewall. Check network prerequisites on [installation doc](https://docs.openshift.com/container-platform/4.2/installing/installing_bare_metal/installing-bare-metal.html)

```shell

#public ports
firewall-cmd --add-port=6443/tcp --permanent
firewall-cmd --add-port=22623/tcp --permanent
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --add-port=443/tcp --permanent
# inter domain ports on libvirt
firewall-cmd --zone=libvirt --add-port=2379-2380/tcp --permanent
firewall-cmd --zone=libvirt --add-port=6443/tcp --permanent
firewall-cmd --zone=libvirt --add-port=22623/tcp --permanent
firewall-cmd --zone=libvirt --add-port=22643/tcp --permanent
firewall-cmd --zone=libvirt --add-port=80/tcp --permanent
firewall-cmd --zone=libvirt --add-port=443/tcp --permanent
firewall-cmd --zone=libvirt --add-port=9000-9999/tcp --permanent
firewall-cmd --zone=libvirt --add-port=10249-10259/tcp --permanent
firewall-cmd --zone=libvirt --add-port=10256/tcp --permanent
firewall-cmd --zone=libvirt --add-port=4789/udp --permanent
firewall-cmd --zone=libvirt --add-port=6081/udp --permanent
firewall-cmd --zone=libvirt --add-port=9000-9999/udp --permanent
firewall-cmd --zone=libvirt --add-port=30000-32767/udp --permanent
firewall-cmd --reload
```

Check if your kvm network enabled to  access public network.
```shell
 sysctl net.ipv4.ip_forward
 ```
 if the result is as below, you're good to go

 ```shell
  net.ipv4.ip_forward = 1
```
If its disabled, than we need to forward the outgoing traffic from libvirt to to public.
```shell
sysctl -w net.ipv4.ip_forward=1
```
To permanently set this parameter, place the line below in /etc/sysctl.conf
```shell 
net.ipv4.ip_forward = 1
```
 
Start python's webserver, serving the current directory in screen.
This will serve the ignition and image files to CoreOS installer.
```shell
# Pick a port that you want to listen on
WEB_PORT="1234"
firewall-cmd --add-port ${WEB_PORT}/tcp
firewall-cmd --add-port ${WEB_PORT}/tcp --permanent
firewall-cmd --zone=libvirt --add-port ${WEB_PORT}/tcp
firewall-cmd --zone=libvirt --add-port ${WEB_PORT}/tcp --permanent
dnf -y install screen python3
# Start your http server
screen -S ${CLUSTER_NAME} -dm \
bash -c "python3 -m http.server ${WEB_PORT}"

#check if it's running
curl http://${HOST_IP}:${WEB_PORT}
```

## Step 5: Create Openshift Manifests and Ignation Files

Create the installation directory for the OpenShift installer. Create install config and ignition files.
This directory is important and tracked by the installer.
It is important even after the installation.

```shell

mkdir install_dir

cat <<EOF > install_dir/install-config.yaml
apiVersion: v1
baseDomain: ${BASE_DOM}
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ${CLUSTER_NAME}
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '${PULL_SEC}'
sshKey: '$(cat $SSH_KEY)'
EOF
```

If you'd like install single master, edit your config file set master replicas as 1. If you go that way it will be enough to install only one master VM.

Create installation manifests than ignition files

```shell
openshift-install create manifests --dir=./install_dir
```

Optionaly you can modify the manifests/cluster-scheduler-02-config.yml Kubernetes manifest file to prevent Pods from being scheduled on the control plane machines:
* Open the manifests/cluster-scheduler-02-config.yml file.
* Locate the mastersSchedulable parameter and set its value to false.
* Save and exit the file.

Create  ignition files

```shell
openshift-install create ignition-configs --dir=./install_dir
```

## Step 6: OCP VM Nodes Setup
##### Download the RHEL guest image
Download the RHEL guest image for KVM.
To setup an external load balancer using haproxy.
Visit RHEL [download page](https://access.redhat.com/downloads/content/69/ver=/rhel---7/latest/x86_64/product-software).
Copy the download link of Red Hat Enterprise Linux KVM Guest Image.

```shell

wget "<paste your link>" -O ./${CLUSTER_NAME}-lb.qcow2
cp ./${CLUSTER_NAME}-lb.qcow2 /var/lib/libvirt/images/${CLUSTER_NAME}-lb.qcow2

```
Create the load balancer VM.
```shell
virt-customize -a /var/lib/libvirt/images/${CLUSTER_NAME}-lb.qcow2 \
  --root-password password:redhat1! --uninstall cloud-init \
  --ssh-inject root:file:$SSH_KEY --selinux-relabel \
  --sm-credentials "${RHNUSER}:password:${RHNPASS}" \
  --sm-register --sm-attach auto \
  --install [nano,vim,wget,curl,haproxy,traceroute,net-tools,lsof,bash-completion,zip,unzip,psmisc,bind-utils] 

virt-install --import --name ${CLUSTER_NAME}-lb \
--disk /var/lib/libvirt/images/${CLUSTER_NAME}-lb.qcow2 \
--memory 1024 --cpu host --vcpus 1 --debug \
--network network=${VIR_NET} --noreboot --noautoconsole

```
Start LB VM

```shell
virsh start ${CLUSTER_NAME}-lb
```

Find the IP and MAC address of the load balancer VM.
Add DHCP reservation and an entry in /etc/hosts.
We will also add the api and api-int host records as they should point to this load balancer.
```shell
LBIP=$(virsh domifaddr "${CLUSTER_NAME}-lb" | grep ipv4 | head -n1 | awk '{print $4}' | cut -d'/' -f1)
MAC=$(virsh domifaddr "${CLUSTER_NAME}-lb" | grep ipv4 | head -n1 | awk '{print $2}')
virsh net-update ${VIR_NET} add-last ip-dhcp-host --xml "<host mac='$MAC' ip='$LBIP'/>" --live --config
echo "$LBIP lb.${CLUSTER_NAME}.${BASE_DOM}" \
"api.${CLUSTER_NAME}.${BASE_DOM}" \
"api-int.${CLUSTER_NAME}.${BASE_DOM}" >> /etc/hosts
```
##### Wildcard DNS
Create the wild-card DNS record and point it to the load balancer:
```shell
echo "address=/apps.${CLUSTER_NAME}.${BASE_DOM}/${LBIP}" \
>> ${DNS_DIR}/${CLUSTER_NAME}.conf
```

!!! At this point its highly recommended, connect to your lb VM and check if you can access top public internet from it and also the phyton web server that started in above steps. 

##### Bootstrap
Install the OpenShift bootstrap node. Make sure the CoreOS installer can get the image and ignition files

```shell

virt-install --name ${CLUSTER_NAME}-bootstrap \
  --disk size=50 --ram 8192 --cpu host --vcpus 2 \
  --os-type linux --os-variant rhel7.0 \
  --network network=${VIR_NET} --noreboot --noautoconsole \
  --location rhcos-install/ \
  --debug \
  --extra-args "nomodeset rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.inst.image_url=http://${HOST_IP}:${WEB_PORT}/rhcos-4.3.0-x86_64-metal.raw.gz coreos.inst.ignition_url=http://${HOST_IP}:${WEB_PORT}/install_dir/bootstrap.ign"
```
Create master nodes

```shell

# Recommended
# --disk size=120 --ram 16384 --cpu host --vcpus 4 \
for i in {1..3}
do
virt-install --name ${CLUSTER_NAME}-master-${i} \
--disk size=100 --ram 12288 --cpu host --vcpus 3 \
--os-type linux --os-variant rhel7.0 \
--network network=${VIR_NET} --noreboot --noautoconsole \
--location rhcos-install/ \
--debug \
--extra-args "nomodeset rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.inst.image_url=http://${HOST_IP}:${WEB_PORT}/rhcos-4.3.0-x86_64-metal.raw.gz coreos.inst.ignition_url=http://${HOST_IP}:${WEB_PORT}/install_dir/master.ign"
done

```
Create worker nodes. Pay attentiton to VM configuration and allocate resources according to your host config.

```shell

  
# 120 GB 16 GB RAM 4 VCPU
# --disk size=120 --ram 16384 --cpu host --vcpus 4 \
for i in {1..3}
do
  virt-install --name ${CLUSTER_NAME}-worker-${i} \
  --disk size=100 --ram 26624 --cpu host --vcpus 12 \
  --os-type linux --os-variant rhel7.0 \
  --network network=${VIR_NET} --noreboot --noautoconsole --debug \
  --location rhcos-install/ \
  --extra-args "nomodeset rd.neednet=1 coreos.inst=yes coreos.inst.install_dev=vda coreos.inst.image_url=http://${HOST_IP}:${WEB_PORT}/rhcos-4.3.0-x86_64-metal.raw.gz coreos.inst.ignition_url=http://${HOST_IP}:${WEB_PORT}/install_dir/worker.ign"
done
```
Wait until all VM's complete the installation and shutdown.

```shell
watch "virsh list --all"
````

We will tell dnsmasq to treat our cluster domain <cluster-name>.<base-domain> as local

```bash
echo "local=/${CLUSTER_NAME}.${BASE_DOM}/" > ${DNS_DIR}/${CLUSTER_NAME}.conf
```

Start up all the VMs.

```shell

for x in  bootstrap master-1 master-2 master-3 worker-1 worker-2  worker-3
do
virsh start ${CLUSTER_NAME}-$x
done
```
Wait for the VMs to obtain IP from DHCP

```shell
for x in  bootstrap master-1 master-2 master-3 worker-1 worker-2 worker-3
do
virsh domifaddr ${CLUSTER_NAME}-$x
done
```

## Step 7: DNS Configuration

Add DHCP reservation (so the VM always get the same IP) and an entry in /etc/hosts.
##### Bootstrap Node:

```shell
IP=$(virsh domifaddr "${CLUSTER_NAME}-bootstrap" | grep ipv4 | head -n1 | awk '{print $4}' | cut -d'/' -f1)
MAC=$(virsh domifaddr "${CLUSTER_NAME}-bootstrap" | grep ipv4 | head -n1 | awk '{print $2}')
virsh net-update ${VIR_NET} add-last ip-dhcp-host --xml "<host mac='$MAC' ip='$IP'/>" --live --config
echo "$IP bootstrap.${CLUSTER_NAME}.${BASE_DOM}" >> /etc/hosts
```

##### Master Nodes

Find the IP and MAC address of the master VMs.
Add DHCP reservation and an entry in /etc/hosts.
We will also add the corresponding etcd host and SRV record:

```shell
for i in {1..3}
do
  IP=$(virsh domifaddr "${CLUSTER_NAME}-master-${i}" | grep ipv4 | head -n1 | awk '{print $4}' | cut -d'/' -f1)
  MAC=$(virsh domifaddr "${CLUSTER_NAME}-master-${i}" | grep ipv4 | head -n1 | awk '{print $2}')
  virsh net-update ${VIR_NET} add-last ip-dhcp-host --xml "<host mac='$MAC' ip='$IP'/>" --live --config
  echo "$IP master-${i}.${CLUSTER_NAME}.${BASE_DOM}" \
  "etcd-$((i-1)).${CLUSTER_NAME}.${BASE_DOM}" >> /etc/hosts
  echo "srv-host=_etcd-server-ssl._tcp.${CLUSTER_NAME}.${BASE_DOM},etcd-$((i-1)).${CLUSTER_NAME}.${BASE_DOM},2380,0,10" >> ${DNS_DIR}/${CLUSTER_NAME}.conf
done
```
##### Worker Nodes
Find the IP and MAC address of the worker VMs.
Add DHCP reservation and an entry in /etc/hosts.
```shell
for i in {1..3}
do
 IP=$(virsh domifaddr "${CLUSTER_NAME}-worker-${i}" | grep ipv4 | head -n1 | awk '{print $4}' | cut -d'/' -f1)
 MAC=$(virsh domifaddr "${CLUSTER_NAME}-worker-${i}" | grep ipv4 | head -n1 | awk '{print $2}')
 virsh net-update ${VIR_NET} add-last ip-dhcp-host --xml "<host mac='$MAC' ip='$IP'/>" --live --config
 echo "$IP worker-${i}.${CLUSTER_NAME}.${BASE_DOM}" >> /etc/hosts
done
```

Reload the NetworkManager and Libvirt for DNS entries to be loaded properly:
```shell
systemctl reload NetworkManager
systemctl restart libvirtd
```
## Step 8: HAProxy Setup
Since we already deployed our load balancer VM, we need to configure it now
##### Configure haproxy

Configure load balancing (haproxy).
We will add frontend/backend configuration in haproxy to point the required ports (6443, 22623, 80, 443) to their corresponding endpoints.

```shell

# Allow haproxy to listen on custom ports
ssh lb.${CLUSTER_NAME}.${BASE_DOM} semanage port -a -t http_port_t -p tcp 6443
ssh lb.${CLUSTER_NAME}.${BASE_DOM} semanage port -a -t http_port_t -p tcp 22623

ssh lb.${CLUSTER_NAME}.${BASE_DOM} <<EOF

echo '
global
  log 127.0.0.1 local2
  chroot /var/lib/haproxy
  pidfile /var/run/haproxy.pid
  maxconn 4000
  user haproxy
  group haproxy
  daemon
  stats socket /var/lib/haproxy/stats

defaults
  mode tcp
  log global
  option tcplog
  option dontlognull
  option redispatch
  retries 3
  timeout queue 1m
  timeout connect 10s
  timeout client 1m
  timeout server 1m
  timeout check 10s
  maxconn 3000
# 6443 points to control plan
frontend ${CLUSTER_NAME}-api *:6443
  default_backend master-api
backend master-api
  balance roundrobin
  server bootstrap bootstrap.${CLUSTER_NAME}.${BASE_DOM}:6443 check
  server master-1 master-1.${CLUSTER_NAME}.${BASE_DOM}:6443 check
  server master-2 master-2.${CLUSTER_NAME}.${BASE_DOM}:6443 check
  server master-3 master-3.${CLUSTER_NAME}.${BASE_DOM}:6443 check

# 22623 points to control plane
frontend ${CLUSTER_NAME}-mapi *:22623
  default_backend master-mapi
backend master-mapi
  balance roundrobin
  server bootstrap bootstrap.${CLUSTER_NAME}.${BASE_DOM}:22623 check
  server master-1 master-1.${CLUSTER_NAME}.${BASE_DOM}:22623 check
  server master-2 master-2.${CLUSTER_NAME}.${BASE_DOM}:22623 check
  server master-3 master-3.${CLUSTER_NAME}.${BASE_DOM}:22623 check

# 80 points to worker nodes
frontend ${CLUSTER_NAME}-http *:80
  default_backend ingress-http
backend ingress-http
  balance roundrobin
  server worker-1 worker-1.${CLUSTER_NAME}.${BASE_DOM}:80 check
  server worker-2 worker-2.${CLUSTER_NAME}.${BASE_DOM}:80 check

# 443 points to worker nodes
frontend ${CLUSTER_NAME}-https *:443
  default_backend infra-https
backend infra-https
  balance roundrobin
  server worker-1 worker-1.${CLUSTER_NAME}.${BASE_DOM}:443 check
  server worker-2 worker-2.${CLUSTER_NAME}.${BASE_DOM}:443 check

' > /etc/haproxy/haproxy.cfg

EOF

ssh lb.${CLUSTER_NAME}.${BASE_DOM} cat /etc/haproxy/haproxy.cfg
ssh lb.${CLUSTER_NAME}.${BASE_DOM}  systemctl start haproxy
ssh lb.${CLUSTER_NAME}.${BASE_DOM}  systemctl enable haproxy

# Make sure haproxy in the load balancer VM is up and running and listening on the desired ports:

ssh lb.${CLUSTER_NAME}.${BASE_DOM} systemctl status haproxy
ssh lb.${CLUSTER_NAME}.${BASE_DOM} netstat -nltupe | grep ':6443\|:22623\|:80\|:443'
```

Reload the NetworkManager and Libvirt for DNS entries to be loaded properly:
```shell
systemctl reload NetworkManager
systemctl restart libvirtd
```

## Step 9: Bootstrap OpenShift 4
Start the OpenShift installation.
Note that the installer will stop after bootrap completion:

```shell
openshift-install --dir=install_dir wait-for bootstrap-complete --log-level debug
```
The OpenShift bootstrap process has started and the control plane will now download a bunch of container images from the registry (quay.io). If internet connection is slow, this will take a long time and might timeout.
From a new terminal on the host, ssh into the bootstrap node and watch the `bootkube` service:
```shell
ssh core@bootstrap journalctl -b -f -u bootkube.service
```

Wait until the installer gives you the following message:

```
INFO API v1.13.4+8560dd6 up
INFO Waiting up to 30m0s for bootstrapping to complete...
DEBUG Bootstrap status: complete
INFO It is now safe to remove the bootstrap resources
```
At this point the bootstrap should have successfully finished and the openshift-install should have told you to that its now safe to remove the bootstrap.

You can login to your openshift cluster and make sure that all the nodes are in a ready state:

```shell
export KUBECONFIG=install_dir/auth/kubeconfig
oc get nodes
cp install_dir/auth/kubeconfig ~/.kube/config
oc get nodes
```
Alternatively you can login to the cluster using the kubeadmin user from the command-line:

```shell
KUBE_PASS=$(cat /root/ocp4/install_dir/auth/kubeadmin-password)
oc login -u kubeadmin -p $KUBE_PASS https://api.$(CLUSTER_NAME).${BASE_DOM}:6443
```
Test your console is working as openning https://console-openshift-console.apps.$(CLUSTER_NAME).${BASE_DOM}

Once you have verified that all cluster nodes are in a ready state, remove the boostrap entries from our load balancer (haproxy).

```shell
ssh lb.${CLUSTER_NAME}.${BASE_DOM} <<EOF
sed -i '/bootstrap\.${CLUSTER_NAME}\.${BASE_DOM}/d' /etc/haproxy/haproxy.cfg
systemctl restart haproxy
EOF
```
Delete the bootstrap VM

```shell
virsh destroy ${CLUSTER_NAME}-bootstrap
virsh undefine ${CLUSTER_NAME}-bootstrap --remove-all-storage
```
We can close the screen session running the python web server

```shell
screen -S ${CLUSTER_NAME} -X quit
```

You can take a look to see if any node CSRs are pending.
```shell
oc get csr
```
You can accept the CSRs by running oc adm certificate approve $CSR – conversely, you can run the following to approve them all (requires jq command).
```shell
oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve
```
Since we do not have a RWX storage type at the moment, we will patch image registry configuration to use "EmptyDir" for storage.
NOTE: This command will fail early on, as the cluster operators are being rolled out. If it fails try after a minute or so.

```shell
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```
At this point OpenShift's cluster operators are being rolled out. Wait for the cluster operators to be fully available.

```shell
watch "oc get clusterversion; echo; oc get clusteroperators"
```
 When all the cluster operators are fully available, it should look some thing like this:

```shell
oc get clusteroperators
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.3.0     True        False         False      2m9s
cloud-credential                           4.3.0     True        False         False      17m
cluster-autoscaler                         4.3.0     True        False         False      8m21s
console                                    4.3.0     True        False         False      3m49s
dns                                        4.3.0     True        False         False      12m
image-registry                             4.3.0     True        False         False      9m5s
ingress                                    4.3.0     True        False         False      8m13s
insights                                   4.3.0     True        False         False      13m
kube-apiserver                             4.3.0     True        False         False      10m
kube-controller-manager                    4.3.0     True        False         False      11m
kube-scheduler                             4.3.0     True        False         False      11m
machine-api                                4.3.0     True        False         False      12m
machine-config                             4.3.0     True        False         False      11m
marketplace                                4.3.0     True        False         False      8m23s
monitoring                                 4.3.0     True        False         False      112s
network                                    4.3.0     True        False         False      13m
node-tuning                                4.3.0     True        False         False      9m19s
openshift-apiserver                        4.3.0     True        False         False      8m45s
openshift-controller-manager               4.3.0     True        False         False      11m
openshift-samples                          4.3.0     True        False         False      7m58s
operator-lifecycle-manager                 4.3.0     True        False         False      12m
operator-lifecycle-manager-catalog         4.3.0     True        False         False      12m
operator-lifecycle-manager-packageserver   4.3.0     True        False         False      8m53s
service-ca                                 4.3.0     True        False         False      13m
service-catalog-apiserver                  4.3.0     True        False         False      9m28s
service-catalog-controller-manager         4.3.0     True        False         False      9m21s
storage                                    4.3.0     True        False         False      9m4s


oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.3.0     True        False         84s     Cluster version is 4.3.0

```

By default ingress operator deploys haproxy pods on worker ndoes. We deployed 3 worker nodes and our routers run any 2 of that 3 workers. Let's label 2 of our worker nodes and change inress router deployment target
```shell
oc label nodes worker-1.${CLUSTER_NAME}.${BASE_DOM} node-role.kubernetes.io/router=''
oc label nodes worker-2.${CLUSTER_NAME}.${BASE_DOM} node-role.kubernetes.io/router=''
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"nodePlacement": { "nodeSelector": { "matchLabels": { "node-role.kubernetes.io/router": ""} }}}'
```
Check router pods running in ``òpenshift-ingress``` project and if any of them running on worker-3 kill it. 

Congratulations! Your cluster is ready and fully functional  

## Step 10 and Beyond: Optional Steps

<details>
<summary>Optional: Set Authentication Provider (htpasswd) </summary>

## Set Authentication Provider (htpasswd)
  By default, only a kubeadmin user exists on your cluster.
  To use the HTPasswd identity provider, you must generate a flat file that contains the user names and passwords for your cluster by using htpasswd. Create or update your flat file with a user name and hashed password:

```shell
 htpasswd -c -B -b </path/to/users.htpasswd> <user_name> <password>
# Continue to add or update credentials to the file:
 htpasswd </path/to/users.htpasswd> <user_name> <password>
```

Create an OpenShift Container Platform Secret that contains the HTPasswd users file.

```shell
 oc create secret generic htpass-secret --from-file=htpasswd=</path/to/users.htpasswd> -n openshift-config
 # The secret key containing the users file must be named htpasswd. The above command includes this name.
```

The following Custom Resource (CR) shows the parameters and acceptable values for an HTPasswd identity provider.

```shell
cat > ./htpasswd_provider.yaml <<EOF
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
EOF
# Apply it
oc apply -f ./htpasswd_provider.yaml -n openshift-config
```
If a CR does not exist, oc apply creates a new CR and might trigger the following warning: Warning: oc apply should be used on resources created by either oc create --save-config or oc apply. In this case you can safely ignore this warning.
Log in to the cluster as a user from your identity provider, entering the password when prompted.

```shell
oc login -u <username>
#Confirm that the user logged in successfully, and display the user name.
oc whoami
```
After you define an identity provider and create a new cluster-admin user, you can remove the kubeadmin to improve cluster security.
```shell
# Define a user as cluster-admin (You must be logged in as an administrator)
oc adm policy add-cluster-role-to-user cluster-admin <user>
# Remove kubeadmin secrets, do not delete this before you assign a cluster admin
# You must be logged in as an administrator
oc delete secrets kubeadmin -n kube-system
```

</details>

<details>
<summary>Optional: Forward external traffic with HAProxy</summary>

## Optional: Forward external traffic with HAProxy

Configure load balancing (haproxy) on your host.
We will add frontend/backend configuration in haproxy to point the required ports (6443, 22623, 80, 443) to OCP LoadBalancer.

```shell
dnf install -y haproxy

cat <<EOF > /etc/haproxy/haproxy.cfg
global
  log 127.0.0.1 local2
  chroot /var/lib/haproxy
  pidfile /var/run/haproxy.pid
  maxconn 4000
  user haproxy
  group haproxy
  daemon
  stats socket /var/lib/haproxy/stats

defaults
  mode tcp
  log global
  option tcplog
  option dontlognull
  option redispatch
  retries 3
  timeout queue 1m
  timeout connect 10s
  timeout client 1m
  timeout server 1m
  timeout check 10s
  maxconn 3000

frontend ocp-api
  bind *:6443
  default_backend master-api
backend master-api
  balance roundrobin
  server lb lb.${CLUSTER_NAME}.${BASE_DOM}:6443
frontend ocp-mapi
  bind *:22623
  default_backend master-mapi
backend master-mapi
  balance roundrobin
  server lb lb.${CLUSTER_NAME}.${BASE_DOM}:22623
frontend ocp-http
  bind *:80
  default_backend ingress-http
backend ingress-http
  balance roundrobin
  server lb lb.${CLUSTER_NAME}.${BASE_DOM}:80
frontend ocp-https
  bind *:443
  default_backend infra-https
backend infra-https
  balance roundrobin
  server lb lb.${CLUSTER_NAME}.${BASE_DOM}:443
EOF

 cat /etc/haproxy/haproxy.cfg
systemctl start haproxy
systemctl enable haproxy
# Make sure haproxy in the load balancer VM is up and running and listening on the desired ports:
systemctl status haproxy
netstat -nltupe | grep ':6443\|:22623\|:80\|:443'
```
</details>

<details>
<summary>Optional: NFS Server</summary>

## Optional: NFS Server

 NFS Server is needed to provide storage for your persistent workloads. Providing a storage for your Image Registry is also important for an health and fast serving cluster.

### Deploy NFS server

Download the RHEL guest image for KVM to setup an NFS server. Visit RHEL [download page](https://access.redhat.com/downloads/content/69/ver=/rhel---7/latest/x86_64/product-software) page. Copy the download link of Red Hat Enterprise Linux KVM Guest Image.
Copy the download link of Red Hat Enterprise Linux KVM Guest Image.

```shell
wget "<paste your link>" -O ./${CLUSTER_NAME}-nfs.qcow2
cp ./${CLUSTER_NAME}-nfs.qcow2 /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2
# Increase the storage
qemu-img resize  /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2 +240G
# Check storage
qemu-img check -r all /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2
# Info
qemu-img info /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2
# Get a backup of the disk to expand
cp /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2 /var/lib/libvirt/images/${CLUSTER_NAME}-nfs-bck.qcow2
# Expand the storage
virt-resize --expand /dev/sda1 /var/lib/libvirt/images/${CLUSTER_NAME}-nfs-bck.qcow2 /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2
#Check FS
virt-filesystems -a /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2 --all --long -h
#Remove Backup
/var/lib/libvirt/images/${CLUSTER_NAME}-nfs-bck.qcow2
# Create the NFS  VM. Customize
virt-customize -a /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2 \
  --root-password password:redhat1! --uninstall cloud-init \
  --ssh-inject root:file:$SSH_KEY --selinux-relabel \
  --sm-credentials "${RHNUSER}:password:${RHNPASS}" \
  --sm-register --sm-attach auto --install nfs-utils,nfs-utils-lib,rpcbind,portmap,openssh-server,openssh-clients
# install the NFS VM
virt-install --import --name ${CLUSTER_NAME}-nfs \
--disk /var/lib/libvirt/images/${CLUSTER_NAME}-nfs.qcow2 \
--disk size=250 --memory 81292 --cpu host --vcpus 2 \
--network network=${VIR_NET} --noreboot --noautoconsole
# Start the NFS VM
virsh start ${CLUSTER_NAME}-nfs

```

Find the IP and MAC address of the NFS VM. Add DHCP reservation and an entry in /etc/hosts.  

```shell
NFSIP=$(virsh domifaddr "${CLUSTER_NAME}-nfs" | grep ipv4 | head -n1 | awk '{print $4}' | cut -d'/' -f1)
MAC=$(virsh domifaddr "${CLUSTER_NAME}-nfs" | grep ipv4 | head -n1 | awk '{print $2}')
virsh net-update ${VIR_NET} add-last ip-dhcp-host --xml "<host mac='$MAC' ip='$NFSIP'/>" --live --config
echo "$NFSIP nfs.${CLUSTER_NAME}.${BASE_DOM}"  >> /etc/hosts
systemctl reload NetworkManager
systemctl restart libvirtd
```

```shell
# Check the the storage
ssh nfs.${CLUSTER_NAME}.${BASE_DOM} df -H
# Configure & Enable NFS
ssh nfs.${CLUSTER_NAME}.${BASE_DOM} <<EOF
echo "Preparing NFS export directories"
mkdir -p /export/ocp/registry
mkdir -p /export/ocp/logging
mkdir -p /export/ocp/metrics
for i in {1..100}; do mkdir -p /export/ocp/pv$i; done
echo "Change mode and ownership"
chown -R nfsnobody:nfsnobody /export
chmod -R 777 /export/ocp
echo "Export /export/ocp directory"
echo "/export/ocp    *(rw,root_squash)" > /etc/exports
echo "Allow network access"
iptables -I INPUT 1 -p tcp --dport 2049 -j ACCEPT
iptables-save
echo "Enable NFS Server"
systemctl enable nfs-server.service
systemctl start nfs-server.service
exportfs -a
systemctl status nfs-server
EOF

firewall-cmd --zone=libvirt --add-port=2049/tcp --permanent
firewall-cmd --reload

```
NFS access client side testing  

```shell
dnf -y install nfs-utils
mkdir -p /tmp/mnttst
mount nfs:/export/ocp/pv99  /tmp/mnttst
# At this point you should be able to see your /tmp/mnttst mounted to an nfs export (nfs:/export/ocp/pv99)
df -H
#Also Check/Test mount output
mount
#Unmount when you done
umount -f -l /tmp/mnttst
```

If tests are ok, let's create persistent volumes.

```shell
mkdir storage
cd storage
echo 'Generating 1Gi PVs'
for i in {1..15}
do
cat > ./pv$i.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv$i
spec:
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Recycle
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs
    path: "/export/ocp/pv$i"
EOF
oc create -f ./pv$i.yaml
done
```
```shell
echo 'Generating 2Gi PVs'
for i in {16..25}
do
cat > ./pv$i.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv$i
spec:
  capacity:
    storage: 2Gi
  persistentVolumeReclaimPolicy: Recycle
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs
    path: "/export/ocp/pv$i"
EOF
oc create -f ./pv$i.yaml
done
```
```shell
echo 'Generating 5Gi PVs'
for i in {26..30}
do
cat > ./pv$i.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv$i
spec:
  capacity:
    storage: 5Gi
  persistentVolumeReclaimPolicy: Recycle
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs
    path: "/export/ocp/pv$i"
EOF
oc create -f ./pv$i.yaml
done
echo 'Generating 1Gi PVs'
for i in {31..40}
do
cat > ./pv$i.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv$i
spec:
  capacity:
    storage: 1Gi
  persistentVolumeReclaimPolicy: Recycle
  accessModes:
    - ReadWriteOnce
  nfs:
    server: nfs
    path: "/export/ocp/pv$i"
EOF
oc create -f ./pv$i.yaml
done

echo 'Generating 2Gi PVs'
for i in {41..45}
do
cat > ./pv$i.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv$i
spec:
  capacity:
    storage: 2Gi
  persistentVolumeReclaimPolicy: Recycle
  accessModes:
    - ReadWriteOnce
  nfs:
    server: nfs
    path: "/export/ocp/pv$i"
EOF
oc create -f ./pv$i.yaml
done
```

In order to provision Image Registry with persistent storage, We need a provisioned persistent volume (PV) with ReadWriteMany access mode, with "100Gi" capacity.
To configure your registry to use storage, change the spec.storage.pvc in the configs.imageregistry/cluster resource.

```shell
echo "PV for Image Registry"
cat > ./pvregistry.yaml <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pvregistry
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: nfs
    path: "/export/ocp/registry"
EOF
oc create -f ./pvregistry.yaml
oc get pv
```
Later we need to reconfigure image registry cluster operator
```shell
#Get Image Registry Backup
oc get configs.imageregistry.operator.openshift.io -o yaml --export > ./imageregistry-backup.yaml
# Patch image registry , previously patched as empty dir
oc edit configs.imageregistry.operator.openshift.io
# Edit Below Values
spec:
  managementState: Managed
  storage:
    pvc:
      claim:
#Check the config
oc get configs.imageregistry.operator.openshift.io -o yaml
#Check the cluster operator status
oc get clusteroperator image-registry
# Check pv
oc get pv pvregistry
```

</details>

<details>
<summary>Optional: Generate LetsEncrypt Certificates and Replace OCP Certficates</summary>

##  Generate LetsEncrypt Certificates and Replace OCP Certficates

Clone the acme.sh GitHub repository.

```shell
cd $HOME
git clone https://github.com/neilpang/acme.sh
cd acme.sh
```

Provide your DNS Keys. I'm using godady in this case. It uses different variables for different DNS provides. Check the project repo to find out which variables required for your case.

```shell
export GD_Key="sdfsdfsdfljlbjkljlkjsdfoiwje"
export GD_Secret="asdfsdafdsfdsfdsfdsfdsafd"
```
Make sure that you are connected to your Red Hat OpenShift Cluster.

```shell
# find the server address
oc whoami --show-server
# set the variable LE_API to the fully qualified domain name:
export LE_API=$(oc whoami --show-server | cut -f 2 -d ':' | cut -f 3 -d '/' | sed 's/-api././')
# Set the second variable LE_WILDCARD to your Wildcard Domain for example:
export LE_WILDCARD=$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}')
# run the acme.sh script
${HOME}/acme.sh/acme.sh --issue -d ${LE_API} -d *.${LE_WILDCARD} --dns dns_gd --debug
# It is usually a good idea to move the certificates from the acme.sh default path to a well known directory. So use the –install-cert option of the acme.sh script to copy the certificates to $HOME/certificates.
export CERTDIR=$HOME/certificates
mkdir -p ${CERTDIR}
${HOME}/acme.sh/acme.sh --install-cert -d ${LE_API} -d *.${LE_WILDCARD} --cert-file ${CERTDIR}/cert.pem --key-file ${CERTDIR}/key.pem --fullchain-file ${CERTDIR}/fullchain.pem --ca-file ${CERTDIR}/ca.cer
```

Installing Certificates for the Router

```shell
# create the secret – and if you have existing certificates, make sure to provide the path to your certificates instead.
oc create secret tls router-certs --cert=${CERTDIR}/fullchain.pem --key=${CERTDIR}/key.pem -n openshift-ingress
# update the Custom Resource for your router. The default custom resource is of type IngressController, is named default and is located in the openshift-ingress-operator project. Note that this project is different from where you created the secret earlier.
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec": { "defaultCertificate": { "name": "router-certs" }}}'

 oc get clusteroperator ingresscontroller
```
After you update the IngressController object the OpenShift ingress operator notices that the custom resource has changed and therefore re-deploys the router.
</details>

<details>
<summary>Optional: Single Master</summary>

## Optional: Single Master
  If you deployed only a single master in previous steps, you can remove etcd quorum guard and decrease number of some pod replicas to 1 only.  
### Remove the Etcd Quorum Guard
```shell
#Extract release info
#oc adm release extract --from quay.io/openshift-release-dev/ocp-release@sha256:782b41750f3284f3c8ee2c1f8cb896896da074e362cf8a472846356d1617752d
#find etcd files
# grep -rnw './' -e 'etcd'

oc patch clusterversion/version --type json -p "$(cat <<- EOF
- op: add
  path: /spec/overrides
  value:
  - kind: Deployment
    group: apps/v1
    name: etcd-quorum-guard
    namespace: openshift-machine-config-operator
    unmanaged: true
EOF
)"
#Scale Quorum Guard to 0
oc scale --replicas=0 deployment/etcd-quorum-guard -n openshift-machine-config-operator
```
### Downgrade pod replicas to 1

```shell
# Downscale the number of consoles, authentication, OLM and monitoring services to one:
oc scale --replicas=1 deployment.apps/console -n openshift-console
oc scale --replicas=1 deployment.apps/downloads -n openshift-console
oc scale --replicas=1 deployment.apps/oauth-openshift -n openshift-authentication
oc scale --replicas=1 deployment.apps/packageserver -n openshift-operator-lifecycle-manager
# NOTE: When enabled, the Operator will auto-scale this services back to original quantity
oc scale --replicas=1 deployment.apps/prometheus-adapter -n openshift-monitoring
oc scale --replicas=1 deployment.apps/thanos-querier -n openshift-monitoring
oc scale --replicas=1 statefulset.apps/prometheus-k8s -n openshift-monitoring
oc scale --replicas=1 statefulset.apps/alertmanager-main -n openshift-monitoring
```
### Set router deployment target and number of replicas

Below is to label single master node as deployment target, redeploy router & set num of replicas to 1

```shell
oc label nodes master-1.${CLUSTER_NAME}.${BASE_DOM} node-role.kubernetes.io/router=''
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"nodePlacement": { "nodeSelector": { "matchLabels": { "node-role.kubernetes.io/router": ""} }}}'
oc patch ingresscontroller/default --type=merge --patch '{"spec":{"replicas": 1}}'  -n openshift-ingress-operator
```
At this point you may loose your console access before router placed in master from the worker nodes
Change LB Config and restart
```shell
ssh lb.${CLUSTER_NAME}.${BASE_DOM} <<EOF

echo '
global
  log 127.0.0.1 local2
  chroot /var/lib/haproxy
  pidfile /var/run/haproxy.pid
  maxconn 4000
  user haproxy
  group haproxy
  daemon
  stats socket /var/lib/haproxy/stats

defaults
  mode tcp
  log global
  option tcplog
  option dontlognull
  option redispatch
  retries 3
  timeout queue 1m
  timeout connect 10s
  timeout client 1m
  timeout server 1m
  timeout check 10s
  maxconn 3000
# 6443 points to control plan
frontend ${CLUSTER_NAME}-api *:6443
  default_backend master-api
backend master-api
  balance roundrobin
  server master-1 master-1.${CLUSTER_NAME}.${BASE_DOM}:6443 check

# 22623 points to control plane
frontend ${CLUSTER_NAME}-mapi *:22623
  default_backend master-mapi
backend master-mapi
  balance roundrobin
  server master-1 master-1.${CLUSTER_NAME}.${BASE_DOM}:22623 check


# 80 points to worker nodes
frontend ${CLUSTER_NAME}-http *:80
  default_backend ingress-http
backend ingress-http
  balance roundrobin
  server master-1 master-1.${CLUSTER_NAME}.${BASE_DOM}:80 check

# 443 points to worker nodes
frontend ${CLUSTER_NAME}-https *:443
  default_backend infra-https
backend infra-https
  balance roundrobin
  server master-1 master-1.${CLUSTER_NAME}.${BASE_DOM}:443 check
' > /etc/haproxy/haproxy.cfg

EOF

ssh lb.${CLUSTER_NAME}.${BASE_DOM}  systemctl restart haproxy
ssh lb.${CLUSTER_NAME}.${BASE_DOM} systemctl status haproxy
ssh lb.${CLUSTER_NAME}.${BASE_DOM} netstat -nltupe | grep ':6443\|:22623\|:80\|:443'
```

</details>
