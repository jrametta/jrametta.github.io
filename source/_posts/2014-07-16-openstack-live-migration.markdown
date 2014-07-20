---
layout: post
title: "OpenStack Live Migration"
date: 2014-07-16 21:52:20 -0700
comments: true
categories: [OpenStack,Brocade]
---

Live migration in the cloud can be useful at times as it can minimize downtime during maintenence and move
instances from overloaded compute nodes.  A little while back I setup a devstack cluster with a shared NFS 
filesystem to perform live migration of OpenStack instances from one compute node to another using KVM
hypervisors.

<!--more-->

When using the Brocade VCS plugin for OpenStack Neutron, the tenant network VLAN configuration is 
automatically updated in the physical network when a new instance is created and also when it is moved to 
another compute node.  This enables live migration without needing to make any changes to the network.

In this writeup I describe the process to reconfigure an existing 2-node Icehouse devstack deployment to
support shared storage-based live migration on OpenStack instances using an NFS server.  If you don't already 
have a working devstack setup,
take a look at [this post](http://jrametta.github.io/the-fabric/blog/2014/07/15/devstack-icehouse-with-brocade-ml2-plugin) using Ubuntu.  


##NFS Server Configuration
I built a simple NFS server using Ubuntu. Install the software package and prepare a directory to
export.

```
$ sudo apt-get install nfs-kernel-server
$ sudo mkdir -p /srv/demo-stack/instances
$ sudo chmod o+x /srv/demo-stack/instances
$ sudo chown stack:stack /srv/demo-stack/instances
```
Add an entry like the one below into `/etc/exports` and then export the directory via `sudo exportfs -ra`
```
/srv/demo-stack/instances 10.254.0.0/20(rw,fsid=0,insecure,no_subtree_check,async,no_root_squash)
```

##OpenStack Node Configuration
Each of the devstack nodes will be NFS clients.  Setup a directory and mount the remote filesystem.
Optionally add the mount point to your `/etc/fstab` so it persists after reboot.

```
mkdir -p /opt/stack/data/instances
sudo apt-get install rpcbind nfs-common
sudo mount 10.254.15.12:/srv/demo-stack/instances /opt/stack/data/instances
```

Several changes to libvirt were made to enable migration.  Edit `/etc/libvirt/libvirtd.conf` to include the
following

&nbsp;&nbsp;   - listen_tls = 0  
&nbsp;&nbsp;   - listen_tcp = 1  
&nbsp;&nbsp;   - auth_tcp = "none"  

Edit libvirtd options in `/etc/default/libvirt-bin` to listen over tcp

&nbsp;&nbsp;   - libvirtd_opts = " -d -l"  


Restart libvirt
```
sudo service libvirt-bin  restart
```

The final configuration is to make sure the VNC server listens on all interfaces and the path to hold the nova
images is set to the mounted NFS directory.  This can be done by adding the following lines to devstack's
local.conf

```
# set this for live migration
VNCSERVER_LISTEN=0.0.0.0
NOVA_INSTANCES_PATH=/opt/stack/data/instances
```
That should be it- run stack.sh and make sure everything is up and running properly.

##Testing things out
Launch an instance in the cloud using Horizon or the CLI.  You can check which compute node the instance lives
on using nova commands (currently it resides on icehouse1)

```
stack@icehouse1:~/devstack$ nova-manage vm list   | awk '{print $1,$2,$4,$5}' | column -t
instance   node       state   launched
instance1  icehouse1  active  2014-07-17
```

If you take a look at the VCS fabric, you can find the physical port in the network for the compute node
node hosting this instance based on it's mac address (in my case port Gi 5/0/7).

```
openstack-rb5# show port-profile status
Port-Profile                        PPID        Activated        Associated MAC        Interface
openstack-profile-901               1           Yes              fa16.3e0e.265d        None
                                                                 fa16.3e98.c835        Gi 5/0/7
                                                                 fa16.3ee7.91bb        None
openstack-profile-902               2           Yes              fa16.3eee.5fe1        None
```
You'll notice it belongs to openstack-profile-901.  If you examine the configuration for this port-profile,
you can see the VLAN association.  Any instances in this particular tenant network will be carried on VLAN 901
as the traffic traverses the VCS fabric.

```
openstack-rb5# show port-profile name openstack-profile-901
port-profile openstack-profile-901
ppid 1
 vlan-profile
  switchport
  switchport mode trunk
  switchport trunk allowed vlan add 901
```

Run the nova live-migration command to move the VM to another compute node.  I ran a continuous ping from the
instance to another server to see if any packets were dropped during the migration.

```
stack@icehouse1:~/devstack$ nova live-migration instance1 icehouse2
stack@icehouse1:~/devstack$
stack@icehouse1:~/devstack$ nova list
+--------------------------------------+-----------+-----------+------------+-------------+---------------------+
| ID                                   | Name      | Status    | Task State | Power State | Networks            |
+--------------------------------------+-----------+-----------+------------+-------------+---------------------+
| 77df10d1-90da-4f88-91ed-dc360ce7733c | instance1 | MIGRATING | migrating  | Running     | private=192.168.0.2 |
+--------------------------------------+-----------+-----------+------------+-------------+---------------------+
```

Looging at the VCS fabric again after a few moments, you should see that the instance has moved to another
OpenStack compute node (it's now on port Gi 6/0/7)

```
openstack-rb5# show port-p status
Port-Profile                        PPID        Activated        Associated MAC        Interface
openstack-profile-901               1           Yes              fa16.3e0e.265d        None
                                                                 fa16.3e98.c835        Gi 6/0/7
                                                                 fa16.3ee7.91bb        None
openstack-profile-902               2           Yes              fa16.3eee.5fe1        None

```
Running nova show confirms the migration has taken place.  If you check the instance, you should see that no
pings were lost during the move.

```
stack@icehouse1:~/devstack$ nova-manage vm list   | awk '{print $1,$2,$4,$5}' | column -t
instance   node       state   launched
instance1  icehouse2  active  2014-07-17
```

Congratulations, you have successfully performed a live migration of an OpenStack instance with zero downtime
;)

##References
- http://docs.openstack.org/trunk/config-reference/content/section_configuring-compute-migrations.html

