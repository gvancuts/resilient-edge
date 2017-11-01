# Resilient IoT Edge Device Gateway Demo


## Abstract

This tutorial covers installation and configuration of our Resilient IoT Edge Device to Cloud Kubernetes demo. In the process of bringing up the demo, we will work through the following stages. 

Index
OS installation & configuration for each device
Kubernetes (K8) Cluster installation
Creation of local Docker registry
Deployment of demo pods & services in the K8 cluster
Demo configuration 
How to show the demo
Troubleshooting
Addendum


## OS Installation
This tutorial was written for Ubuntu Server 16.04 LTS. We therefore recommend you use the same operating system when you set up your own demo environment. All devices will be connected to the same network and use DHCP. In this demo the following Class-C network is being used: 192.168.2.0/24. In the instructions we assume  router with at least 4 Ethernet drops is available and can be configured appropriately for the demo to work. We use the router’s DHCP server to assign static IP addresses to each device. This is required for the Ansible playbooks that deploy and configure the K8 cluster and to work. The demo is comprised of the following devices:


### 1 x Skull Canyon NUC

This device will be our K8 master and server for the local Docker registry, as well as an NTP time server. 
When you install the OS, please select the package groups openssh server and hypervisor. 
During setup phase, create a “demouser” account and disable the auto-update feature.
After the installation is complete install the following packages manually into the system.

sudo apt update
sudo apt upgrade
sudo apt install python-pip
sudo pip install --upgrade pip
sudo pip install docker.py
sudo pip install ansible 

Use the MAC address of the NUC Ethernet interface to set the following static IP lease in your router’s DHCP configuration: 192.168.2.117 

Replace the /etc/hosts file with the following content: cat /etc/hosts
127.0.0.1	localhost 192.168.2.117 
127.0.1.1	loadbalancer
192.168.2.124	gw1
192.168.2.126	gw2
192.168.2.128	gw3

The following lines are desirable for IPv6 capable hosts
::1     localhost 192.168.2.117 ip6-localhost 192.168.2.117 ip6-loopback
f:f02::1 ip6-allnodes
ff02::2 ip6-allrouters

Copy the example NTP server example config file in the addendum of this document to /etc/ntp.conf and enable the service at boot time.
systemctl enable ntpd.service

Generate an SSH key-pair. The private key must have no password set and the public key must be deployed to all MinnowBoard devices in order to grant password-less authentication. How this is done is explained here.  
Allow the demouser account to run sudo without requiring a password by adding the following line at the end of the /etc/sudoers file (use sudo visudo for that) 
demouser ALL=(ALL) NOPASSWD: ALL 

NOTE: On the Skull Canyon NUC it’s advisable to upgrade the kernel to latest version (4.14 or higher.) This is required as the kernel version Ubuntu server 16.04 ships with does result in the system losing its HDMI signal at boot time. The only way to work around that is to unplug and replug the HDMI cable. With newer kernel versions this doesn’t happen.

### 3 x MinnowBoard Turbot B

The MinnowBoard devices will act as edge device gateways. They will also be kubelets (K8 workers). Please install Ubuntu Server 16.04 LTS on all three devices.
When you install the OS, please select the package groups openssh server and hypervisor. 
During setup phase, create a “demouser” account and disable the auto-update feature.
The hostname for each MinnowBoard should be set to gw[1-3] respectively.
Use the MAC address of each MinnowBoard Ethernet interface to set the following static IP leases in your router’s DHCP configuration:	
	Gw1: 192.168.2.124
	Gw2: 192.168.2.126
		Gw3: 192.168.2.128
After the installation is complete install the following packages manually into the system.
Copy the example NTP client config example file in the addendum of this document to /etc/ntp.conf and enable the service at boot time.
systemctl enable ntpd.service
Allow the demouser account to run sudo without requiring a password by adding the following line at th end of the /etc/sudoers file. 
demouser ALL=(ALL) NOPASSWD: ALL 
Use the public SSH key you generated on the Skull Canyon NUC and create an ~/.ssh/authorized_keys file. Instructions how this is done can be found here. 

### 3 x Intel Edison

The Edison will act as  our OCF sensor devices and connect to the network wirelessly. Each device has an RGB led and a motion sensor connected to it. 

The operating system for this device was built using Yocto and is based on the latest Reference OS for IoT GitHub repository available here. Please follow the instructions in the repository README file to build an image for Edison. The default image configuration will result in a build that contains all the bits that are needed to power and run OCF servers using the Edison platform. 
TODO: put the OCF sensor files that include the kiosk[1-3] href changes in a GitHub demo repository and reference them here.
The OCF server code for each board needs to be updated to extend the HREF resource URI for the sensor by the demo kiosk number the device is located at. I.e. /a/rgbled-kiosk1 and a/pir-kiosk1 for the pair of sensors connected to one of the Edison boards.git clone
Kubernetes (K8) Cluster installation
Once all devices are up and running and have been assigned an IP in our 192.168.2.0/24 network, it’s a good time to perform a couple of pre-flight checks.
Can you ping all devices (3 MinnowBoard IP addresses from the Skull Canyon NUC? 
Can you log on to each MinnowBoard as demouser using SSH without being prompted for a password?
Can you sudo to root as demouser on each MinnowBoard without being prompted for a password?

If the answer to all three questions is yes, it’s time to proceed to the next stage of the demo setup, which is the installation of the K8 cluster. 

### Skull Canyon NUC

Log on to the Skull Canyon NUC as demouser and create a git directory into which you clone the Ansible playbook repository
git clone https://github.com/uhofemei/resilient-edge.git
In the repository please update the inventory file to reflect the IPs that were assigned to the Skull Canyon NUC and MinnowBoard devices. For example:

cat ~/demouser/git/resilient-edge/inventory
192.168.2.117 ansible_connection=local <-(this must be the Skull Canyon IP)

[kube-workers] <- (here you list the MinnowBoard IPs)
192.168.2.124
192.168.2.126
192.168.2.128
After you saved the inventory file changed it’s a good idea to conduct an Ansible sanity test to make sure that it can access all devices K8 will be deployed on. Please run the following command to perform the test.
demouser@loadbalancer:~/git/uhofemei/resilient-edge/$ ansible -i inventory kube-workers -m ping

The expected output looks like this. 

192.168.2.128 | SUCCESS => {
    "changed": false, 
    "failed": false, 
    "ping": "pong"
}
192.168.2.126 | SUCCESS => {
    "changed": false, 
    "failed": false, 
    "ping": "pong"
}
192.168.2.124 | SUCCESS => {
    "changed": false, 
    "failed": false, 
    "ping": "pong"
}

Now it’s time to deploy K8. To do this we run the first ansible playbook using the following command.
ansible-playbook -i inventory deploy-k8s.yml
 
## Creation of local Docker registry
In our demo we use a local Docker registry, which allows us to operate and run the demo offline. It will also speed up the download and deployment of Docker images. To achieve these performance improvements, we need to replace  the Docker service file on the Skull Canyon NUC and all MinnowBoards with the following version.

demouser@loadbalancer:~$ cat /etc/systemd/system/docker.service.d/kolla.conf 
[Service]
MountFlags=shared
ExecStart=
ExecStart=/usr/bin/dockerd  --storage-driver=overlay2 --insecure-registry 192.168.2.117:5000 <-(this is the IP of the Skull Canyon NUC where the registry will be created)

After replacing the file please make sure you restart the Docker service by running the following command on all devices. 

service docker restart

Next add ‘demouser’ to the the docker system group on each device by running the following command: 

sudo usermod -aG docker $USER

Now it’s time to pull the DockerHub images, tag them, and push the images to the local Docker registry. We do this by running the following command: 


ansible-playbook -i inventory load-docker-images.yml


The output of the command should look like this:

## Deployment of demo pods & services in the K8 cluster

Finally run the last playbook, which will generate the needed K8 pods and services to run the demo. 

	ansible-playbook -i inventory deploy-resilient-edge-demo.yml

The output of the command should look like this:


## Demo configuration
Once the demo has been deployed the REST API URL in the admin dashboard needs to be updated. To identify the cluster and service IPs for the demo run the following command:


	kubectl -n iot2cloud get services

You should see the admin dashboard running on port 4000 (use the normal IP of the loadbalancer, not the one internal to Kubernetes). Log in with the credentials user: admin, password: admin and change the REST API URL to http://gateway.iot2cloud:8000/
 
## Troubleshooting
This tutorial is based on a blog written by Theresa Shan. Her blog explains how to deploy K8 services manually and contains many useful K8 administrative commands to debug the cluster in case there are issues.  You can find her blog here: https://ichbinblau.github.io/

## Addendum

In the NTP server configuration file it’s required to update the restrict IP address range to reflect the network range chosen in the demo setup. 

### NTP server config: 
```
# For more information about this file, see the man pages
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).

driftfile /var/lib/ntp/drift
logfile /var/log/ntp.log

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
#restrict 127.0.0.1 
#restrict ::1

# Hosts on local network are less restricted.
restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst

server  127.127.1.0 # local clock
fudge   127.127.1.0 stratum 8

#broadcast 192.168.1.255 autokey	# broadcast server
#broadcastclient			# broadcast client
#broadcast 224.0.1.1 autokey		# multicast server
#multicastclient 224.0.1.1		# multicast client
#manycastserver 239.255.254.254		# manycast server
#manycastclient 239.255.254.254 autokey # manycast client

# Enable public key cryptography.
#crypto

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography. 
keys /etc/ntp/keys

# Specify the key identifiers which are trusted.
#trustedkey 4 8 42

# Specify the key identifier to use with the ntpdc utility.
#requestkey 8

# Specify the key identifier to use with the ntpq utility.
#controlkey 8

# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats

# Disable the monitoring facility to prevent amplification attacks using ntpdc
# monlist command when default restrict does not include the noquery flag. See
# CVE-2013-5211 for more details.
# Note: Monitoring will not be disabled with the limited restriction flag.
disable monitor
```
In the client configuration file it’s required to update the server IP to point to the IP assigned to the loadbalancer. 

### NTP client config:
```
# For more information about this file, see the man pages
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).

driftfile /var/lib/ntp/drift
 
# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1 
restrict ::1

# Hosts on local network are less restricted.
#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).

server 192.168.1.102 iburst

#broadcast 192.168.1.255 autokey	# broadcast server
#broadcastclient			# broadcast client
#broadcast 224.0.1.1 autokey		# multicast server
#multicastclient 224.0.1.1		# multicast client
#manycastserver 239.255.254.254		# manycast server
#manycastclient 239.255.254.254 autokey # manycast client

# Enable public key cryptography.
#crypto

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography. 
keys /etc/ntp/keys

# Specify the key identifiers which are trusted.
#trustedkey 4 8 42

# Specify the key identifier to use with the ntpdc utility.
#requestkey 8

# Specify the key identifier to use with the ntpq utility.
#controlkey 8

# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats

# Disable the monitoring facility to prevent amplification attacks using ntpdc
# monlist command when default restrict does not include the noquery flag. See
# CVE-2013-5211 for more details.
# Note: Monitoring will not be disabled with the limited restriction flag.
disable monitor
```


