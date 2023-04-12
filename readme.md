#### Installation of a redundant Iscsi SAN based upon an Ubuntu 22.04 LTS minimized.

Thanks to `ALUA` (Asymetric Logical Unit Access) the back-end iSCSI storage can work in `Active/Active` setup thus providing faster fail-over since in this case the resources are not being moved around. The initiator has the paths to the same target on both back-end servers available, can detect when the current active path has failed and quickly switch to the spare one.







Prevent floppy module from loading. Optional but remove annoying entry on system log

`sudo rmmod floppy`
`echo "blacklist floppy" | sudo tee /etc/modprobe.d/blacklist-floppy.conf`
`sudo dpkg-reconfigure initramfs-tools`

##### Configure networking 

`sudo rm /etc/netplan/00-installer-config.yaml`

Node0 -- Replace with proper ip addresses

```
cat << EOF > /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    eth0:
      addresses:
      - 10.0.9.1/8
      nameservers:
        addresses: [10.0.30.121, 10.0.30.122]
      routes:
        - to: default
          via: 10.0.30.254
    eth1:
      addresses:
      - 192.168.10.1/24
    eth2:
      addresses:
      - 172.16.15.1/16
    eth3:
      addresses:
      - 172.16.20.1/16   
  version: 2
EOF
```

Node1  -- Replace with proper ip addresses

```
cat << EOF > /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    eth0:
      addresses:
      - 10.0.9.2/8
      nameservers:
        addresses: [10.0.30.121, 10.0.30.122]
      routes:
        - to: default
          via: 10.0.30.254
    eth1:
      addresses:
      - 192.168.10.2/24
    eth2:
      addresses:
      - 172.16.15.2/16
    eth3:
      addresses:
      - 172.16.20.2/16   
  version: 2
EOF
```

`sudo netplan apply`

Reconnect and install required packages on both nodes

`sudo apt-get update && apt-get upgrade && apt-get install nano build-essential git corosync pacemaker drbd-utils resource-agents lvm2-lockd dlm-controld psmisc -y`

