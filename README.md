# Installing OpenStack Single VM PoC using RDO

## Create Base Image:
1. Create VM in VirtualBox or Fusion and enable Nested Virtualization
2. Install Latest CentOS-7 x86_64 Minimal from ISO
3. yum -y update
4. yum -y open-vm-tools
5. Shut Down and Save Image as CentOS7 Base

## Create RDO AIO Controller
1. Create Linked-Clone system with at least 2 CPUs and 12GB of RAM
2. Edit __/etc/sysconfig/network-scripts/ifcfg-ens##__ and set __onboot=yes__ _(Optional: set Static IP settings)_
3. ```yum -y install centos-release-openstack-queens```
4. ```yum -y install python-setuptools```
5. Disable selinux enforcement: ```setenforce 0```
6. Make Selinux Change Permanent:  ```sed -i s/SELINUX=enforcing/SELINUX=permissive/g /etc/selinux/config```
7. ```systemctl disable NetworkManager```
8. ```systemctl stop NetworkManager```
9. ```systemctl enable network```
10. ```systemctl disable firewalld```
11. ```systemctl stop firewalld```
12. ```hostnamectl set-hostname controller```
13. ```yum install -y openstack-packstack```
14. ```packstack --allinone```

# Configuring Barbican Keystone Hooks

```
source keystonerc_admin
openstack user create --domain default --password-prompt barbican
openstack role add --project services --user barbican admin
openstack role create creator
openstack role add --project services --user barbican creator
openstack service create --name barbican --description "key manager" key-manager
openstack endpoint create --region RegionOne key-manager public http://barbican:9311
openstack endpoint create --region RegionOne key-manager internal http://barbican:9311
openstack endpoint create --region RegionOne key-manager admin http://barbican:9311
```
* Remember your password you'll use it in the barbican.conf

# Installing Barbican API on CentOS
(Use the base image you created earlier or just a plain and updated CentOS7 VM)

## Configure Networking:
1. Edit __/etc/sysconfig/network-scripts/ifcfg-ens##__ and set __onboot=yes__ _(Optional: set Static IP settings)_
2. ifup ens##
3. Add ens## IP address to /etc/hosts as ```<ip.goes.here>    barbican barbican.localdomain```
4. Set Hostname: ```hostnamectl set-hostname barbican```

## Install Barbican
1. ```yum -y install centos-release-openstack-queens```
2. ```yum install openstack-barbican-api -y```

## Configure Barbican
barbican.conf.simplified:
```
[DEFAULT]
host_href = http://barbican:9311
sql_connection = mysql+pymysql://barbican:barbican-db-pass@barbican/barbican
debug = false
transport_url = rabbit://guest:guest@controller

[keystone_authtoken]
auth_uri = http://controller:5000/v3
auth_url = http://controller:35357/v3
auth_version = v3
insecure = true
region_name = RegionOne
memcached_servers = localhost:11211
auth_type = password
project_name = services
user_domain_id = default
project_domain_id = default
username = barbican
password = barbicanpassword
```
