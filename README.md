# Contrail_Ansible_TP_Integration
Contrail Ansible for Third-party OS 

### Preperation of Hosts

##### Install initial Dependencies on the servers that host the Containers (these are not required for the compute nodes)

Install the following packages on all the nodes where the containers are created:

```
yum install kernel-devel kernel-headers nfs-utils socat wget git patch ntp createrepo -y && reboot
```

##### Install latest Docker on the nodes

This can be done using Ansible which installs a previous version (yum) which is not supported by contrail. Copy and paste the lines below on all the nodes intended for the Containers:

```
echo "Installing Latest Docker Version"
sudo yum check-update
#apt-get installs older version of docker when used yum install
curl -fsSL https://get.docker.com/ | sh;
#start docker-engine service
echo "Start Docker Engine"
sudo systemctl start docker;
#check status docker-engine service
echo "Displaying the status of docker"
sudo systemctl status docker;
#enable the docker-engine service
echo "Check Docker Status and Enable the Service"
sudo systemctl enable docker;
#Flush iptables temporarily
iptables --flush;
```

### Ansible Configuration

##### Clone Ansible Repository

```
git clone https://github.com/gokulpch/Contrail_Ansible_TP_Integration.git
```

##### Create a Directory in Contrail_Ansible_TP_Integration/contrail-ansible/playbooks for Container Images

By default ansible picks up the images from this path if no Docker repo information is specified in all.yaml

```
mkdir -p Contrail_Ansible_TP_Integration/contrail-ansible/playbooks/container_images
```

#### Place all the container images in the directory created above

Following are the images that are needed for the install (sample):

```
root@hostx# tar -xzvf contrail-docker-images_4.0.1.0-32.tgz
root@hostx:~/Contrail_Ansible_TP_Integration/contrail-ansible/playbooks/container_images# ls -lrth
total 3.5G
-rw-r--r-- 1 33694  757 638M Sep 23 03:17 contrail-controller-ubuntu16.04-4.0.1.0-32.tar.gz
-rw-r--r-- 1 33694  757 247M Sep 23 03:17 contrail-analytics-ubuntu16.04-4.0.1.0-32.tar.gz
-rw-r--r-- 1 33694  757 276M Sep 23 03:17 contrail-agent-ubuntu16.04-4.0.1.0-32.tar.gz
-rw-r--r-- 1 33694  757 496M Sep 23 03:17 contrail-analyticsdb-ubuntu16.04-4.0.1.0-32.tar.gz
-rw-r--r-- 1 33694  757 109M Sep 23 03:17 contrail-lb-ubuntu16.04-4.0.1.0-32.tar.gz
```

Untar the archive with the images.

#### Following are the Configuration parameters that has to be changed as required

##### hosts (Contrail_Ansible_TP_Integration/contrail-ansible/playbooks/inventory/my-inventory/hosts)

This is the Ansible Hosts file which defines the nodes role and IP information.

```
# Enable contrail-repo when required - this will start a contrail apt or yum repo container on specified node
# This repo will be used by other nodes on installing any packages in the node
# setting up contrail-cni need this repo enabled
# NOTE: Repo is required only for mesos and nested mode kubernetes
;[contrail-repo]
;192.168.0.24

[contrail-controllers]
10.87.1.54
10.87.1.61
10.87.1.56

[contrail-analyticsdb]
10.87.1.54
10.87.1.61
10.87.1.56

[contrail-analytics]
10.87.1.54
10.87.1.61
10.87.1.56

[contrail-compute]


[contrail-lb]
10.87.1.57

##
# Only enable if you setup with openstack (when cloud_orchestrator is openstack)
##
;[openstack-controllers]
;192.168.0.23

# Kubernetes Nested Mode.
# Underlay Contrail Api Server to which kubernetes instance should
# connect to. This is to be set for ONLY for nested kubernetes installation.
;[kubernetes-contrail-controllers]
;192.168.0.24

# Kubernetes Nested Mode.
# Underlay Contrail Analytics Server to which kubernetes instance should
# connect to. This is to be set for ONLY for nested kubernetes installation.
;[kubernetes-contrail-analytics]
;192.168.0.24
```

##### all.yml (Contrail_Ansible_TP_Integration/contrail-ansible/playbooks/inventory/my-inventory/group_vars/all.yml)

This is the Central configuration file for various contrail related parameters. The cloned repo already has a sample file, following are the parametes that are to be changed as required.

  *Set the Compute Mode to Baremetal (default)
  
  ```
  contrail_compute_mode: bare_metal
  ```
  
  *Provide the OS release information of the Containers
  
  ```
  os_release: ubuntu16.04
  ```
  
  *Provide the Release information of Contrail. This should match the image_version tag
  
  ```
  contrail_version: 4.0.1.0-32
  ```
  
  *vRouter Physical interface. This is the interface that will be inherited by vRouter
  
  ```
  vrouter_physical_interface: eth0
  ```
  
  *Mention the LB IP as the Analytics, AnalyticsDB and Controller IP in global configuration
  
  ```
  global_config: { analytics_ip: 10.87.1.57, controller_ip: 10.87.1.57, config_ip: 10.87.1.57 }
  ```
  
  *Provide the admin credentails and Keystone information
  
  ```
  keystone_config: {'ip': '50.51.23.14', 'admin_password': 'P0rterple1', 'admin_user': 'admin@cos.net', 'admin_port': '443',     'insecure': 'False', 'identity_uri': 'https://juniper.cos.net/keystone_admin', 'auth_url':               'https://juniper.cos.net:443/keystone/v2.0'}
  ```

### Run Ansible

Path:  Contrail_Ansible_TP_Integration/contrail-ansible/playbooks/

```
ansible-playbook  -i inventory/my-inventory site.yml -vv
```

### Custom Configuration Changes

*Change parameters of vnc_api_lib.ini for contrail-alarm-gen service to be active. Once the installation is complete the contrail-alarm-gen service on the analytics containers will be in inactive state the following are the changes that are needed to activate the service.

```
[global]
WEB_SERVER = 10.87.1.43
WEB_PORT = 8082

BASE_URL = /
;BASE_URL = /tenants/infra ; common-prefix for all URLs

; Authentication settings (optional)
[auth]
AUTHN_TYPE = keystone
AUTHN_PROTOCOL = https
AUTHN_SERVER = juniper.cosnet.net
AUTHN_PORT = 443
AUTHN_URL = /keystone/v2.0
AUTHN_TOKEN_URL = https://juniper.cosnet.net/keystone/v2.0/tokens
;AUTHN_TOKEN_URL = http://127.0.0.1:35357/v2.0/tokens
```

### vRouter Installation

#### Prepare vRouter/Compute Nodes

Get the vRouter packages to all the compute nodes. Run the below snippet as a shell script adding it in a file (example: setup_vrouter.sh | execute: ./setup_vrouter.sh)


```

######
#Get Contrail-vRouter packages
######
#Install kernel specific dependencies
yum install glibc-devel wget git -y;
######
#Set Contrail Packages
######
mkdir -p /tmp/contrail-vrouter/repo;
tar -xzf /root/contrail-vrouter-packages_4.0.1.0-32.tgz -C /tmp/contrail-vrouter/repo/;
cd /tmp/contrail-vrouter/repo/contrail-vrouter-packages_4.0.1.0-32 && mv * /tmp/contrail-vrouter/repo/.;
cd;
######
#Removing empty directory
######
rm -rf /tmp/contrail-vrouter/repo/contrail-vrouter-packages_4.0.1.0-32;
######
#Create a Repo
######
yum install createrepo -y;
createrepo /tmp/contrail-vrouter/repo/;
######
#when using local repo in the target node
######
cat << __EOT__ > /etc/yum.repos.d/contrail-install.repo
[contrail_install_repo]
name=contrail_install_repo
baseurl=file:///tmp/contrail-vrouter/repo/
enabled=1
priority=1
gpgcheck=0
__EOT__
sleep 10;
yum clean all;
##
sleep 5;
yum install yum-plugin-priorities -y;
######
#Install contrail-utilities
######
sleep 5;
yum install contrail-fabric-utils contrail-setup -y;
######
#Install contrail-vrouter
######
sleep 5;
yum install contrail-vrouter-common contrail-vrouter contrail-vrouter-init -y;
######

```

This will install all required packages on the Compute node required for vRouter installation.

#### Provision and Register the vRouter to the Controller

* Make sure to have proper hostname on the nodes, if using the name.domain_name then the out put of "hostname -s"should be same as the hostname.domain_name

Use the below utility to provision the vRouter:

```
contrail-compute-setup --self_ip 10.87.1.57 --hypervisor libvirt --cfgm_ip 10.87.1.57 --collectors 10.87.1.54 10.87.1.61 10.87.1.56 --control-nodes 10.87.1.54 10.87.1.61 10.87.1.56 --keystone_ip 50.51.23.14  --keystone_auth_protocol https  --keystone_auth_port 443  --keystone_admin_user admin@cos9.net  --keystone_admin_password bye761Uq --keystone_admin_tenant_name admin --register && cd /etc/contrail && ./contrail_reboot
```

This will provision the vRouter followed by Registration (hostname mismatches may occur if the hostname doesn't match the hostname.domain_name) and will reboot the node (contrail_reboot)

```
*--self_ip    #IP of the Compute_Node
*--cfgm_ip    #IP of LB_COntainer
*--collectors   #List of all three Analytics_Nodes
*--control-nodes  #List of all three Controller_Nodes
*--keystone_ip  #Openstack Kestone_IP
*--keystone_admin_user  #Exact user name as specified in Openstack
```
