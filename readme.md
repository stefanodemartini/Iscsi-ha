#### Installation of a redundant Iscsi SAN based upon an Ubuntu 22.04 LTS minimized.

The system consists in a 2 nodes cluster which have same hardware components currently supported natively since the Ubuntu setup. 
The operating system is installed on a reserved hard disk (USB)  and other hard disk, even raid volumes, block devices, files etc. are available trough  SCST which is a collection of Linux kernel drivers that implement SCSI target functionality. Aim of the project is to have a fully redundant ISCSI san with automatic fail-over and offering multipath (MPIO) connection to iscsi initiators. All components are included as installable packages directly from standard Ubuntu repository except for SCST which we are going to compile and install. 
Thanks to `ALUA` (Asymetric Logical Unit Access) the back-end iSCSI storage can work in `Active/Active` setup thus providing faster fail-over since in this case the resources are not being moved around. The initiator has the paths to the same target on both back-end servers available, can detect when the current active path has failed and quickly switch to the spare one.

Core components are:

- **DRBD** Distributed Replicated Block Device (*drbd-utils*)
- **LVM2** Linux Logical Volume Manager V2
- **LVM** Locking daemon (*lvm2-lockd*)
- **DLM** Distributed Lock Manager control daemon (*dlm-controld*)
- **Corosync** - or *Cluster Membership Layer*, provides reliable messaging, membership and quorum information about the cluster. Currently, Pacemaker supports Corosync as this layer.
- **Pacemaker** - or *Cluster Resource Manager*, provides the brain that processes and reacts to events that occur in the cluster. Events might be: nodes joining or leaving the cluster, resource events caused by failures, maintenance, or scheduled activities. To achieve the desired availability, Pacemaker may start and stop resources and fence nodes.
- **Resource Agents** - Scripts or operating system components that start, stop or monitor resources, given a set of resource parameters. These provide a uniform interface between pacemaker and the managed services.
- **Fence Agents** - Scripts that execute node fencing actions, given a target and fence device parameters.
- **crmsh** - Advanced command-line interface for High-Availability cluster management in GNU/Linux.
- **SCST** The generic SCSI target subsystem for Linux 



##### Configure time zone

`timedatectl set-timezone Europe/Rome`

##### Configure networking 

*****

`sudo rm /etc/netplan/00-installer-config.yaml`

Iscsinode0 -- Replace with proper ip addresses

```
cat << EOF > /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: no
    eno2:
      dhcp4: no
    enp2s0f0:
      dhcp4: no
      ignore-carrier: true
      addresses:
        - 172.16.15.1/8
      mtu: 9000
    enp2s0f1:
      dhcp4: no
      ignore-carrier: true
      optional: true
      addresses:
        - 172.16.20.1/8
      mtu: 9000
    enp6s0f0:
      dhcp4: no
      ignore-carrier: true
      addresses:
        - 172.16.25.1/8
      mtu: 9000
    enp6s0f1:
      dhcp4: no
      ignore-carrier: true
      addresses:
        - 172.16.30.1/8
      mtu: 9000
  bonds:
    bond0:
      interfaces:
        - eno1
        - eno2
      addresses:
        - 10.10.0.1/8
      parameters:
        mode: active-backup
        transmit-hash-policy: layer3+4
        mii-monitor-interval: null
        primary: eno1
      routes:
        - to: default
          via: 10.0.30.254
      nameservers:
        search:
          - ge.vevy.com
        addresses:
          - 10.0.30.121
          - 10.0.30.122
  version: 2
EOF
```

 Iscsinode1 -- Replace with proper ip addresses

```
cat << EOF > /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: no
    eno2:
      dhcp4: no
    enp2s0f0:
      dhcp4: no
      ignore-carrier: true
      addresses:
        - 172.16.15.2/8
      mtu: 9000
    enp2s0f1:
      dhcp4: no
      ignore-carrier: true
      optional: true
      addresses:
        - 172.16.20.2/8
      mtu: 9000
    enp6s0f0:
      dhcp4: no
      ignore-carrier: true
      addresses:
        - 172.16.25.2/8
      mtu: 9000
    enp6s0f1:
      dhcp4: no
      ignore-carrier: true
      addresses:
        - 172.16.30.2/8
      mtu: 9000
  bonds:
    bond0:
      interfaces:
        - eno1
        - eno2
      addresses:
        - 10.10.0.2/8
      parameters:
        mode: active-backup
        transmit-hash-policy: layer3+4
        mii-monitor-interval: null
        primary: eno1
      routes:
        - to: default
          via: 10.0.30.254
      nameservers:
        search:
          - ge.vevy.com
        addresses:
          - 10.0.30.121
          - 10.0.30.122
  version: 2
EOF
```

`sudo netplan apply`

Reconnect and install required packages on both nodes

`sudo apt-get update && apt-get upgrade && apt-get install nano build-essential git corosync pacemaker drbd-utils resource-agents lvm2-lockd dlm-controld psmisc ipmitool -y`

If necessary prevent floppy module from loading. Optional but remove annoying entry on system log

`sudo rmmod floppy`
`echo "blacklist floppy" | sudo tee /etc/modprobe.d/blacklist-floppy.conf`
`sudo dpkg-reconfigure initramfs-tools`
