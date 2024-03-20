# Configure Libvirt to Use Bridged Networking

### Notes
- Network is `10.10.10.0/23`.
- Server has two NICs that will be teamed as `bond0`.
- NICs are connected to a managed switch (capable of Link Aggregation)
- Bridge will use the `bond0` interface to allow VMs access to the network.
- Default libvirt NAT network will be replaced with bridge.
- VMs get their IP from a dedicated DHCP server on the network.



### 1. Create Link Aggregation Group (using LACP) named `bond0`
```bash
nmcli con add type bond con-name bond0 ifname bond0 mode 802.3ad miimon 100
```

### 2. Add the two NICs to the LAG
```bash
nmcli con add type bond-slave ifname enp4s0f0 master bond0
nmcli con add type bond-slave ifname enp4s0f1 master bond0
```
### 3. Temporarily disable `bond0`` to prevent loops
```bash
nmcli con down bond0
```

### 4. On the switch, create a LAG with the ports connected to each NIC


### 5. Bring `bond0`` up back up
```bash
nmcli con down bond0
```

### 6. Verify LACP is working
```bash
cat /proc/net/bonding/bond0

Ethernet Channel Bonding Driver: v5.14.0-362.18.1.el9_3.0.1.x86_64

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer2 (0)
MII Status: up
(...)

Slave Interface: enp4s0f0
(...)

Slave Interface: enp4s0f1
(...)
```

### 7. Disable IPv6 and Enable IPv4 port-forwarding by adding the following lines to `/etc/sysctl.conf`
```bash
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv4.ip_forward = 1
```

### 8. Reload the configuration.
```bash
systl -p
```

### 9. Create a new bridge named `br0` and configure it to use the `bond0` interface
```bash
nmcli con add type bridge con-name br0 ifname br0
nmcli con mod bond0 master br0
```

### 10. Assign a static IP to the new bridge and bring it up

```bash
nmcli con mod br0 ipv4.addresses '10.10.11.1/23' ipv4.gateway '10.10.10.1' ipv4.dns '10.10.10.2' ipv4.dns-search 'example.com' ipv4.method manual
nmcli con up br0
```

### 11. Undefine (delete) the default libvirt NAT network and restart the libvirtd service

```bash
virsh net-undefine default
systemctl restart libvirtd
```

### 12. Define a new default network by adding the following lines to a new file `br0-network.xml`

```xml
<network>
  <name>default</name>
  <forward mode='bridge'/>
  <bridge name='br0'/>
</network>
```

### 13. Define (load) the new default network, activate it, and set it to autostart

```bash
virsh net-define br0-network.xml
virsh net-start default
virsh net-autostart default
```

### 14. Verify the network is running before use

```bash
virsh net-list

 Name  	State	Autostart   Persistent
--------------------------------------------
 default   active   yes     	yes
```

