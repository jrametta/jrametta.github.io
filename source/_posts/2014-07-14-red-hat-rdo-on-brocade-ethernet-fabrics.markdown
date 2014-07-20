---
layout: post
title: "Red Hat RDO on Brocade Ethernet Fabrics"
date: 2014-07-14 22:46:57 -0700
comments: true
categories: [openstack,brocade,rdo]
---

This short writup describes how to setup Red Hat's OpenStack RDO Havana release with the Brocade VCS plugin for Neutron networking.

<!--more-->

## INTRODUCTION
RDO can easily be deployed using Packstack and the Linux Bridge plugin.  Once complete, a few steps are
required to reconfigure Neutron to use the Brocade VCS plugin for managing both virtual and physical network
infrastructure through OpenStack API.

This guide has been tested using CentOS 6.4, but should be applicable to other Red Hat based systems such as
RHEL or Fedora.

With the Havana release of OpenStack, we are still using the monolithic network plugin. Ice House release will
introduce the use of the Modular Layer2 (ML2) plugin to simultaneously utilize the variety of layer 2
networking technologies.

## BROCADE VCS CONFIGURATION
Brocade VDX switches should be running NOS 4.0 or above with logical chassis mode enabled for configuration
distribution across all fabric nodes.  See the Brocade NOS administrators guide for additional information.
Any ports connected to OpenStack compute/network nodes should be configured as port-profile ports.

```
VDX_84_186# show running-config interface GigabitEthernet 2/0/28
interface GigabitEthernet 2/0/28
  port-profile-port
 no shutdown
```

## RDO INSTALLATION USING PACKSTACK
This guide will walk through the process of deploying OpenStack on a single server with the option of adding
one or more additional compute nodes.
Begin by deploying OpenStack as documented in the RDO Neutron Quickstart guide at
http://openstack.redhat.com/Neutron-Quickstart.  Install the software repos, reboot the system, and install
PackStack.

```
# yum install –y -y http://rdo.fedorapeople.org/openstack-grizzly/rdo-release-grizzly.rpm
# yum -y update
# reboot
# sudo yum install -y openstack-PackStackack python-netaddr
```

Generate a PackStack answer file and edit the following variables to enable the linuxbridge plugin.
Additional compute nodes can be specified here as well.

```
# packstack --gen-answer-file=linuxbridge.answers
# vi linuxbridge.answers
```
```
CONFIG_QUANTUM_L2_PLUGIN=linuxbridge
CONFIG_QUANTUM_LB_VLAN_RANGES=physnet1:2:1000
CONFIG_QUANTUM_LB_INTERFACE_MAPPINGS=physnet1:eth1
CONFIG_NOVA_COMPUTE_HOSTS=10.17.95.6,10.17.88.130
CONFIG_NOVA_NETWORK_PRIVIF=eth1
CONFIG_NOVA_COMPUTE_PRIVIF=eth1
```

Edit the configuration for  the physical interface connected to the VCS fabric for tenant networks.  It should
be configured up with no IP address and in promiscuous mode.  All nodes should have a similar configuration.

```
DEVICE="eth1"
BOOTPROTO=static
ONBOOT="yes"
TYPE="Ethernet"
```

One method to configure the interface for promiscuous mode at boot, is to create /sbin/ifup-local with the
following content

```
#!/bin/bash
if [[ "$1" == "eth1" ]]
then
  /sbin/ifconfig $1 promisc
  RC=$?
fi
```

Set executable bit.  This script will run during boot right after network interfaces are brought online.

```
# chmod +x /sbin/ifup-local
# /etc/init.d/network restart
```

Run packstack on the controller node using the previously created answer file

```
# packstack --answer-file=linuxbridge.answers
```

Once complete, edit /etc/ImageMagick//quantum/l3_agent.ini on the controller and remove br-ex from the
following line (it should be set to nothing)
```
external_network_bridge =
```

Download and install a test image into glance
```
# wget -c https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
# glance image-create --name=cirros-0.3.0-x86_64 --disk-format=qcow2  \
   --container-format=bare < cirros-0.3.0-x86_64-disk.img
```

## INSTALL BROCADE NEUTRON PLUGIN
To simplify installation, all existing nova instances, networks, and routers should be deleted. Install the
Brocade plugin from the repository on the controller/network node.
```
# yum install openstack-quantum-brocade`
```

Edit /etc/quantum/plugins/brocade/brocade.ini file on the controller/network node.  The database user and
password can be copied from the packstack answer file or preexisting linuxbridge.ini.  Ensure the database
user has proper credentials.
```
[SWITCH]
username = admin
password = password
address  = 10.254.8.5
ostype   = NOS

[DATABASE]
sql_connection = mysql://root:75ddb50b165b4ed0@localhost/brcd_quantum?charset=utf8

[PHYSICAL_INTERFACE]
physical_interface = physnet1

[VLANS]
network_vlan_ranges = physnet1:1000:2999

[AGENT]
root_helper = sudo /usr/bin/quantum-rootwrap /etc/quantum/rootwrap.conf

[LINUX_BRIDGE]
physical_interface_mappings = physnet1:eth0
```

On the controller, create brcd_quantum database and grant permissions (change IP address and passwords as
needed).
```
# mysql << EOF
CREATE DATABASE brcd_quantum;
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON quantum.* TO 'quantum'@'10.17.88.130' \
IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EOF
```

Set the core plugin to Brocade in quantum.conf on all nodes.
core_plugin = quantum.plugins.brocade.QuantumPlugin.BrocadePluginV2


On the controller,QuantumPlugin install the netconf client needed to communicate with the VCS fabric
```
# git clone https://code.grnet.gr/git/ncclient
# cd ncclient && python setup.py install
```

On the controller, update the symlink in /etc/quantum to point to the brocade.ini file
```
# rm plugin.ini && ln -s  /etc/quantum/plugins/brocade/brocade.ini plugin.ini
```

Disable or remove the OVS agent if it is installed.  Reboot the system and verify working status.

## TESTING THINGS OUT
Source the keystone rc file that was installed into root’s home directory to obtain credentials and verify
status of OpenStack services.  
```
# . ~/ keystonerc_admin
# openstack-status
# quantum agent-list
```
Then use the CLI or Horizon to create networks, and launch new virtual machine instances.  Within the VCS
fabric, check that new port-profiles are created for every tenant network that is created.  
```
VDX_84_186# show port-profile
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


As new instances are launched, they should be tied to the port-profile corresponding to the network they
belong to.  Any instances on the same network should be able to communicate with each other.
```
VDX_84_186# show port-profile status
Port-Profile                        PPID        Activated        Associated MAC        Interface
openstack-profile-2                 1           Yes              fa16.3e1b.95d0        None
                                                                 fa16.3e64.fce8        Gi 2/0/28
                                                                 fa16.3e85.5b2f        Gi 2/0/28
                                                                 fa16.3ea6.3741        Gi 2/0/5
                                                                 fa16.3ecd.bfc1        Gi 2/0/5
                                                                 fa16.3eeb.87f7        Gi 2/0/28
openstack-profile-3                 2           Yes              fa16.3e2c.0baf        None


```

