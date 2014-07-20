---
layout: post
title: "Devstack Icehouse with Brocade ML2 Plugin"
date: 2014-07-15 00:21:31 -0700
comments: true
categories: [devstack,brocade]
---

[Devstack](http://devstack.org) is a scripted installation of OpenStack that can be used for development or
demo purposes.  This writeup covers a simple two node devstack installation using the Brocade VCS plugin for
OpenStack networking (aka Neutron). 

<!--more-->

The VCS ML2 plugin supports both Open vSwitch and Linux Bridge agents and realizes tenant networks as
port-profiles in the physical network infrastructure.  A port-profile in a Brocade Ethernet fabric is like a
network policy for VMs or OpenStack instances and can contain information like VLAN assignment, QoS
information, and ACLs.  Because tenant networks are provisioned end-to-end, no additional networking setup is
required anywhere in the network.

##Deployment Topology

My hardware environment is pretty simple.  I have two servers on which to run OpenStack - one will be a
controller/compute node, the other will just be a compute node.  

{% img /images/os_topo.png %}

##Server Configuration

I used Ubuntu Precise as the OS platform.  The network interfaces were configured as below on the controller.
Compute node is similar.

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
address 10.17.88.129
netmask 255.255.240.0
gateway 10.17.80.1
dns-nameservers 10.17.80.21

# Private tenant network interface (connected to VCS fabric)
auto eth1
iface eth1 inet manual
up ifconfig eth1 up promisc
```

The VCS plugin currently uses NETCONF for communication with the Ethernet fabric and the ncclient python library
is required on the controller node.

```
# git clone https://code.grnet.gr/git/ncclient
# cd ncclient && python ./setup.py install
```
OpenStack runs as a non-root user that has sudo privileges. I usually have a user already setup, but devstack
will create a new user if you try to run stack.sh as root.

```
# adduser stack
# echo "stack ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

As stack user, install this devstack repo in the home directory
```
# git clone https://github.com/openstack-dev/devstack.git
# cd devstack
```

##Configuring Devstack

Controller node local.conf

```
[[local|localrc]]
# MISC
# RECLONE=yes
# OFFLINE=False

# Branch/Repos
NOVA_BRANCH=stable/icehouse
GLANCE_BRANCH=stable/icehouse
HORIZON_BRANCH=stable/icehouse
KEYSTONE_BRANCH=stable/icehouse
QUANTUM_BRANCH=stable/icehouse
NEUTRON_BRANCH=stable/icehouse
CEILOMETER_BRANCH=stable/icehouse
HEAT_BRANCH=stable/icehouse

disable_service n-net
enable_service g-api g-reg key n-crt n-obj n-cpu n-cond n-sch horizon

# Neutron services (enable)
enable_service neutron q-svc q-agt q-dhcp q-meta q-l3
Q_PLUGIN_CLASS=neutron.plugins.ml2.plugin.Ml2Plugin
Q_PLUGIN_EXTRA_CONF_FILES=ml2_conf_brocade.ini
Q_PLUGIN_EXTRA_CONF_PATH=/home/stack/devstack
Q_ML2_PLUGIN_MECHANISM_DRIVERS=openvswitch,brocade
Q_ML2_PLUGIN_TYPE_DRIVERS=${Q_ML2_PLUGIN_TYPE_DRIVERS:-local,flat,vlan,gre,vxlan}
ENABLE_TENANT_VLANS="True"
PHYSICAL_NETWORK="phys1"
TENANT_VLAN_RANGE=901:950
OVS_PHYSICAL_BRIDGE=br-eth1

ADMIN_PASSWORD=openstack
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=$ADMIN_PASSWORD
MYSQL_PASSWORD=$ADMIN_PASSWORD

DEST=/opt/stack
LOG=True
LOGFILE=stack.sh.log
LOGDAYS=1
HOST_NAME=$(hostname)
SERVICE_HOST=10.17.88.129
HOST_IP=10.17.88.129
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
KEYSTONE_AUTH_HOST=$SERVICE_HOST
KEYSTONE_SERVICE_HOST=$SERVICE_HOST
SCHEDULER=nova.scheduler.filter_scheduler.FilterScheduler
SCREEN_HARDSTATUS="%{= Kw}%-w%{BW}%n %t%{-}%+w"

```

The controller node should also contain an ML2 configuration file ml2_conf_brocade.ini identifying
the authentication credentials
and management virtual IP for the VCS fabric.  The location of this file (usually somewhere in
/etc/neutron/plugins), but should be specified via the `Q_PLUGIN_EXTRA_CONF_PATH` parameter in local.conf above.
I happened to just place it in stack's home directory.

```
[ml2_brocade]
username = admin
password = password
address  = 172.27.116.32
ostype   = NOS
physical_networks = phys1
```

Compute node local.conf
```
[[local|localrc]]

# MISC
# RECLONE=yes
# OFFLINE=True

# Branch/Repos
NOVA_BRANCH=stable/icehouse
GLANCE_BRANCH=stable/icehouse
HORIZON_BRANCH=stable/icehouse
KEYSTONE_BRANCH=stable/icehouse
QUANTUM_BRANCH=stable/icehouse
NEUTRON_BRANCH=stable/icehouse

disable_service n-net
ENABLED_SERVICES=n-cpu,rabbit,quantum,q-agt,n-novnc

Q_PLUGIN=ml2
Q_AGENT=openvswitch
ENABLE_TENANT_VLANS="True"
PHYSICAL_NETWORK="phys1"
TENANT_VLAN_RANGE=901:950
OVS_PHYSICAL_BRIDGE=br-eth1

ADMIN_PASSWORD=openstack
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=$ADMIN_PASSWORD
MYSQL_PASSWORD=$ADMIN_PASSWORD

DEST=/opt/stack
LOG=True
LOGFILE=stack.sh.log
LOGDAYS=1
HOST_NAME=$(hostname)
SERVICE_HOST=10.17.88.129
HOST_IP=10.17.88.130
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
KEYSTONE_AUTH_HOST=$SERVICE_HOST
KEYSTONE_SERVICE_HOST=$SERVICE_HOST
SCHEDULER=nova.scheduler.filter_scheduler.FilterScheduler

VNCSERVER_LISTEN=$HOST_IP
VNCSERVER_PROXYCLIENT_ADDRESS=$HOST_IP

SCREEN_HARDSTATUS="%{= Kw}%-w%{BW}%n %t%{-}%+w"
```

## Testing Things Out
Source the openrc in the devstack directory to obtain credentials and use the CLI to have a look around,
create networks, and launch new virtual machine instances. Alternatively, login to the Horizon dashboard at
http://controlerNodeIP and use the GUI (user: admin or demo, password: openstack).

Within the VCS fabric, check that new port-profiles are created for every tenant network that is created. Two
new port-profiles should exist after running stack.sh for the first time.  These corrospond to the initial
pubic and private networks that the devstack script creates.

```
DX1# show port-profile
port-profile default
ppid 0
vlan-profile
  switchport
  switchport mode trunk
  switchport trunk allowed vlan all
  switchport trunk native-vlan 1
port-profile openstack-profile-2
ppid 1
vlan-profile
  switchport
  switchport mode trunk
  switchport trunk allowed vlan add 2
port-profile openstack-profile-3
ppid 2
vlan-profile
  switchport
  switchport mode trunk
  switchport trunk allowed vlan add 3
```

As new instances are launched, they should be tied to the port-profile corresponding the network they belong
to. Any instances on the same network should be able to communicate with each other through the VCS fabric.

```
VDX1# show port-profile status
Port-Profile                        PPID        Activated        Associated MAC Interface
openstack-profile-2                 1           Yes              fa16.3e1b.95d0 None
                                                                 fa16.3e64.fce8        Gi 2/0/28
                                                                 fa16.3e85.5b2f        Gi 2/0/28
                                                                 fa16.3ea6.3741        Gi 2/0/5
                                                                 fa16.3ecd.bfc1        Gi 2/0/5
                                                                 fa16.3eeb.87f7        Gi 2/0/28
openstack-profile-3                 2           Yes              fa16.3e2c.0baf        None
```

## References
- Brocade OpenStack Networking Plugin Wiki https://wiki.openstack.org/wiki/Brocade-quantum-plugin
- OpenStack Documentation http://docs.openstack.org
- DevStack http://devstack.org

