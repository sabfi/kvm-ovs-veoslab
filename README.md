# KVM OPENVSWITCH VEOSLAB

Open Virtual Switch (OVS) can be used together with KVM to switch VM and/or container traffic. Among other things it allows you to directly pass L2 protocol frames (i.e. LLDP, LACP, EAPOL, etc) between VM ports and/or container interfaces by using openflow rules as described below, so veoslab VM instances can see the other ones as directly connected by a network cable.

First, after a fresh OVS install (“apt install openvswitch-switch”, etc) create the ovs instance (ovs0 in the example):

```
# ovs-vsctl add-br ovs0
```

## Attaching VM interfaces to OVS

To attach KVM VM interfaces to the ovs0 instance the VM XML definition can be edited like in the example below. There are many ways to do that, one is virsh edit (VLAB3 is the VM name):

```
# virsh edit VLAB3
```

Now the interfaces can be defined similar to this:

```xml
  <interface type='bridge'>
      <mac address='52:54:00:9d:bd:75'/>
      <source bridge='ovs0'/>
      <vlan>
        <tag id='34'/>
      </vlan>
      <virtualport type='openvswitch'>
        <parameters interfaceid='bd9ddb67-a0af-4564-ab08-000ab909f164'/>
      </virtualport>
      <target dev='veos3_ma1'/>   *** How the HOST end of veth VM interface is named in the HOST
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:ab:16:23'/>
      <source bridge='ovs0'/>
      <vlan>
        <tag id='13'/>
      </vlan>
      <virtualport type='openvswitch'>
        <parameters interfaceid='7af2af86-c8e3-43cd-a466-fe0919060624'/>
      </virtualport>
      <target dev='veos3_e1'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x07' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:e9:15:a4'/>
      <source bridge='ovs0'/>
      <vlan>
        <tag id='13'/>
      </vlan>
      <virtualport type='openvswitch'>
        <parameters interfaceid='92f70f11-cfd6-4c17-a5ac-9da97712c8ad'/>
      </virtualport>
      <target dev='veos3_e2'/>
      <model type='e1000'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x08' function='0x0'/>
    </interface>
```

Now, once the VM has been started interfaces are seen in the hypervisor host as regular interfaces:

```
# virsh start VLAB3
# ip address show
...
8: veos3_ma1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master ovs-system state UNKNOWN group default qlen 1000
    link/ether fe:54:00:9d:bd:75 brd ff:ff:ff:ff:ff:ff
9: veos3_e1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master ovs-system state UNKNOWN group default qlen 1000
    link/ether fe:54:00:ab:16:23 brd ff:ff:ff:ff:ff:ff
10: veos3_e2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master ovs-system state UNKNOWN group default qlen 1000
    link/ether fe:54:00:e9:15:a4 brd ff:ff:ff:ff:ff:ff
11: veos3_e3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master ovs-system state UNKNOWN group default qlen 1000
    link/ether fe:54:00:8d:93:40 brd ff:ff:ff:ff:ff:ff
12: veos3_e4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master ovs-system state UNKNOWN group default qlen 1000
    link/ether fe:54:00:82:cc:6b brd ff:ff:ff:ff:ff:ff
13: veos3_e5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master ovs-system state UNKNOWN group default qlen 1000
    link/ether fe:54:00:ba:6d:ff brd ff:ff:ff:ff:ff:ff
...
```

And they are now added to the OVS0 instance (as per XML VM config) as access ports on the corresponding OVS vlans:

```
# ovs-vsctl show
b2e9be2e-0bb4-4690-8a33-075e2b31d6eb
    Bridge "ovs0"
        Port "veos3_ma1"
            tag: 34
            Interface "veos3_ma1"
        Port "veos3_e5"
            tag: 30
            Interface "veos3_e5"
        Port "veos3_e4"
            tag: 23
            Interface "veos3_e4"
        Port "veos3_e3"
            tag: 23
            Interface "veos3_e3"
        Port "veos3_e2"
            tag: 13
            Interface "veos3_e2"
        Port "veos3_e1"
            tag: 13
            Interface "veos3_e1"
        Port "ovs0"
            Interface "ovs0"
                type: internal
```

## Attaching Docker container interfaces to OVS

Docker container interfaces can be also attached to OVS. On the same host we run a docker container (image name has been called host:v2 and it is a local lab image based on alpine linux with some packages added on top like open-lldp, tcpdump, iperf and others), and like in the VM, the HOST end of the veth interface is attached to the OVS0 instance:

```
# docker run -it --hostname=host1 --name=host1 --cap-add=NET_ADMIN --net=none host:v2 /bin/sh
# ovs-docker add-port ovs0 eth0 host1 --ipaddress=10.10.34.2/24 --gateway=10.10.34.1
# ovs-vsctl show (to see what is the new interface name)
# ovs-vsctl set port 73b6c8e075ea4_l tag=23
```

Now the OVS0 instance has the container interface in OVS vlan 23 (like the veos3_e3 interface):

```
# ovs-vsctl show
b2e9be2e-0bb4-4690-8a33-075e2b31d6eb
    Bridge "ovs0"
        Port "73b6c8e075ea4_l"
            tag: 23
            Interface "73b6c8e075ea4_l"
        Port "veos3_ma1"
            tag: 34
            Interface "veos3_ma1"
        Port "veos3_e5"
            tag: 30
            Interface "veos3_e5"
        Port "veos3_e4"
            tag: 23
            Interface "veos3_e4"
        Port "veos3_e3"
            tag: 23
            Interface "veos3_e3"
        Port "veos3_e2"
            tag: 13
            Interface "veos3_e2"
        Port "veos3_e1"
            tag: 13
            Interface "veos3_e1"
        Port "ovs0"
            Interface "ovs0"
                type: internal
```

As expected, inside the container the other end of the 73b6c8e075ea4_l veth is seen as a regular eth0 interface.
Inside the host1 docker container (i.e. docker attach host1) LLDP can be enabled similar to this (grabbed from [Docker Topo](github.com/networkop/docker-topo)

```
lldpad -d
for i in `ls /sys/class/net/ | grep 'eth\|ens\|eno'`
do
    lldptool set-lldp -i $i adminStatus=rxtx
    lldptool -T -i $i -V sysName enableTx=yes
    lldptool -T -i $i -V portDesc enableTx=yes
    lldptool -T -i $i -V sysDesc enableTx=yes
done
```

## Adding OVS openflow rules

At this point, even if LLDP has been enabled in the host1 container *no* LLDP neighbors are seen in the veos_lab VM yet:

```
VLAB3(config)#show lldp neighbors
Last table change time   : never
Number of table inserts  : 0
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port       Neighbor Device ID               Neighbor Port ID           TTL
VLAB3(config)#
```

Note that at this point *no* openflow rules are in place:

```
# /usr/bin/ovs-ofctl dump-flows ovs0
 cookie=0x0, duration=1005.107s, table=0, n_packets=1545, n_bytes=156079, priority=0 actions=NORMAL
```

Adding the following openflow rules sets up “direct forwarding”:
From veos3_e3 to 73b6c8e075ea4_l (container interface)
From 73b6c8e075ea4_l (container interface) to veos3_e3

```
# cat connect.sh
#!/usr/bin/bash
/usr/bin/ovs-ofctl add-flow ovs0 in_port=$1,actions=output:$2
/usr/bin/ovs-ofctl add-flow ovs0 in_port=$2,actions=output:$1

# ./connect.sh 73b6c8e075ea4_l veos3_e3
```

Now the openflow rules are in place:

```
# /usr/bin/ovs-ofctl dump-flows ovs0
 cookie=0x0, duration=24.209s, table=0, n_packets=4, n_bytes=604, in_port="73b6c8e075ea4_l" actions=output:"veos3_e3"
 cookie=0x0, duration=24.207s, table=0, n_packets=14, n_bytes=1640, in_port="veos3_e3" actions=output:"73b6c8e075ea4_l"
 cookie=0x0, duration=1387.076s, table=0, n_packets=1890, n_bytes=198461, priority=0 actions=NORMAL
```

And now host1 container is seen as an LLDP neighbor in e3 interface as expected:

```
VLAB3(config)#show lldp neighbors
Last table change time   : 0:00:30 ago
Number of table inserts  : 1
Number of table deletes  : 0
Number of table drops    : 0
Number of table age-outs : 0

Port       Neighbor Device ID               Neighbor Port ID           TTL
Et3        host1                            626f.4ccf.42c9             120
```

## A word on mac addresses

### Veoslab System Mac Address

When setting up lab topologies sometimes is handy to manually define the System Mac Address of your veoslab swicthes. To do so you can edit ```/mnt/flash/veos-config``` to define the SYSTEMMACADDR variable.

```
# cat /mnt/flash/veos-config
SYSTEMMACADDR=5054.0000.0101
```

After the change is done and the veoslab switch rebooted you can see the new System Mac Address.

```
vlab01#show version
 vEOS
Hardware version:
Serial number:
System MAC address:  5054.0000.0101

Software image version: 4.22.3M
Architecture:           i686
Internal build version: 4.22.3M-14418192.4223M
Internal build ID:      413991dd-4451-4406-a16c-f1c6ac19d1f3

Uptime:                 0 weeks, 0 days, 0 hours and 10 minutes
Total memory:           1498424 kB
Free memory:            696436 kB
```

### Mlag

By default libvirt assigns mac addresses to VM interfaces beggining with 52:54:xx:xx:xx:xx. These macs are defined in the VM XML definition file like in the previous example in "Attaching VM interfaces to OVS" section. If you are using mlag and see your mlag state always in "connecting" then try to assign mac addresses to your veoslab VM interfaces with the locally administered bit cleared out. That is the second least significant bit of the first mac address byte. For instance, assign mac addresses beggining with 5*0*:54:xx:xx:xx:xx instead.
