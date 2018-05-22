# Installing OpenStack Single VM PoC using RDO

## Create Base Image:
1. Create VM in VirtualBox or Fusion and enable Nested Virtualization
2. Install Latest CentOS-7 x86_64 Minimal from ISO
3. yum -y update
4. yum -y open-vm-tools
5. Shut Down and Save Image as CentOS7 Base

## Create RDO All-In-One OpenStack Controller
1. Create Linked-Clone system with at least 2 CPUs and 12GB of RAM
2. Edit __/etc/sysconfig/network-scripts/ifcfg-ens##__ and set __onboot=yes__ _(Optional: set Static IP settings)_
3. ```yum -y install centos-release-openstack-queens```
4. ```yum -y install python-setuptools```
5. Disable selinux enforcement: ```setenforce 0```
6. Make Selinux Change Permanent:  ```sed -i s/SELINUX=enforcing/SELINUX=permissive/g /etc/selinux/config```
7. ```systemctl disable NetworkManager```
8. ```systemctl enable network```
9. ```systemctl disable firewalld```
10. ```hostnamectl set-hostname controller```
11. ```yum install -y openstack-packstack```
12. Reboot
13. ```packstack --allinone``` _(Optional: <a href=https://www.rdoproject.org/networking/neutron-with-existing-external-network/>Configure External Networking</a>)_
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
3. Set Hostname: ```hostnamectl set-hostname barbican```
4. Update /etc/hosts on both controller and barbican (use your own IPs, example below)
```
[root@controller ~(keystone_admin)]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

172.16.9.164 controller controller.localdomain
172.16.9.178 barbican barbican.localdomain
```
5. Disable Firewall _(Optional: Configure firewall, but this is a PoC)_
```
systemctl disable firewalld
systemctl stop firewalld
```

## Install Barbican
1. ```yum -y install centos-release-openstack-queens```
2. ```yum install openstack-barbican-api -y```

## Configure Barbican

### Install Barbican's Local Database
In this PoC we are going to use a separate database for Barbican.  Barbican's database should be protected in production and should not be shared with other applications, even though we're already running SQL for the controller.  

1. ```yum -y install mariadb mariadb-server```
2. ```systemctl enable mariadb```
3. ```systemctl start mariadb```

### Configure Barbican's Database and DB User
1. ```mysql -u root```
2. ```CREATE DATABASE barbican;```
3. Add barbican user and give it full permissions to the barbican DB:
```
GRANT ALL PRIVILEGES ON barbican.* TO 'barbican'@'localhost' \
  IDENTIFIED BY 'barbican-db-pass';
```
4. Exit MySQL Client ```\q```

### Install Memcached
1. ```yum -y install memcached```
2. ```systemctl enable memcached```
3. ```systemctl start memcached```

### Configure Barbican Service
1. Copy the following configuration to /etc/barbican/barbican.conf:  _(Rename the original for safe keeping)_
```
[DEFAULT]
host_href = http://barbican:9311
sql_connection = mysql+pymysql://barbican:barbican-db-pass@localhost/barbican
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
password = barbican
```
2. Enable Barbican API on Boot ```systemctl enable openstack-barbican-api```
3. Start the Barbican API ```systemctl start openstack-barbican-api```
4. _Optionally: Follow barbican logs with ```journalctl -fu openstack-barbican-api```

### Fix Bug with Barbican RDO Package
There's currently a bug that prevents the RDO barbican package from creating /run/barbican after a reboot.
1. Create /etc/tmpfiles.d/barbican.conf with the following contents:
```
d /var/run/barbican 0755 barbican barbican -
```

# Verify Barbican is working from the Controller
1. ```source keystonerc_admin```
2. ```openstack secret store```
3. New generic secret HREF should be returned, see example output:
```
[root@controller ~(keystone_admin)]# openstack secret store
+---------------+----------------------------------------------------------------------+
| Field         | Value                                                                |
+---------------+----------------------------------------------------------------------+
| Secret href   | http://barbican:9311/v1/secrets/95bb66b5-d78c-44bc-bfd4-34cf53df6947 |
| Name          | None                                                                 |
| Created       | 2018-04-04T22:32:47+00:00                                            |
| Status        | ACTIVE                                                               |
| Content types | None                                                                 |
| Algorithm     | aes                                                                  |
| Bit length    | 256                                                                  |
| Secret type   | opaque                                                               |
| Mode          | cbc                                                                  |
| Expiration    | None                                                                 |
+---------------+----------------------------------------------------------------------+
```

# Configure OpenStack Volume Encryption Support
(see https://docs.openstack.org/cinder/latest/configuration/block-storage/volume-encryption.html)
## Modify cinder.conf for Cinder-API
1. Search for auth_endpoint under the [barbican] section and add
```auth_endpoint = http://localhost:35357/v3```
2. Search for [key_manager] section and add
```backend = barbican```
3. Restart cinder-api: ```systemctl restart openstack-cinder-api```
4. Install cryptsetup tools: ```yum -y install cryptsetup```

## Modify the nova.conf for Nova-Compute
1. Search for [key_manager] section and add
```backend = barbican```
2. Comment out the backend=nova.keymgr.conf_key_mgr.ConfKeyManager:
```
backend=barbican
#backend=nova.keymgr.conf_key_mgr.ConfKeyManager
```
3. Restart nova-compute: ```systemctl restart openstack-nova-compute```

## Create an Encrypted Volume Type
1. ```source keystonerc_admin```
2. Create encrypted volume-type called "LUKS":
```
openstack volume type create --encryption-provider luks \
  --encryption-cipher aes-xts-plain64 --encryption-key-size 256 --encryption-control-location front-end LUKS
```

## Create Unencrypted Volume
```openstack volume create --size 1 'unencrypted volume'```

## Create Encrypted Volume
```openstack volume create --size 1 --type LUKS 'encrypted volume'```

# Test Volume Encryption
## Create a Test Instance
```openstack server create --flavor 1 --image cirros test-instance```

## Mount Unencrypted Volume and Write Data
1. ```openstack server add volume test-instance 'unencrypted volume'```
2. Open VNC Console to test-instance
3. Log into Cirros with user: cirros / password = cubswin:)
4. Switch user to root: ```sudo su -```
5. Write some clear text to volume: ```echo "test data" > /dev/vdb```
6. Verify you can read data within OS: ```head -n1 /dev/vdb```

## Mount Encrypted Volume and Write Data
1. ```openstack server add volume test-instance 'encrypted volume'```
2. Write some clear text to volume: ```echo "test data" > /dev/vdc```
3. Verify you can read data within OS: ```head -n1 /dev/vdc```

## Check Volumes for Data
On the controller:
1. ```openstack volume list``` (Make note of volume IDs)
2. ```head -n1 /dev/mapper/cinder--volumes-volume--<unencrypted volume id>```
3. You should see the string: *test data*
4. ```head -n1 /dev/mapper/cinder--volumes-volume--<encrypted volume id>```
5. You should see the LUKS encrypted volume header
