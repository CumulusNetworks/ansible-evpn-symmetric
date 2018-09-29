The comprehensive demo built on cldemo-vagrant is here:
https://github.com/CumulusNetworks/cldemo-evpn-symmetric

This demo is only mean to be run on CITC and is a subset of features from the demo above.

# citc-evpn-symmetric

This demo deploys a VXLAN architecture that provides VRF tenancy and EVPN as the control plane using flat files.

Server01 and Server03 live in VRF RED and can communicate with each other.
Server02 and Server04 live in VRF BLUE and can communicate with each other.

Hosts in VRF RED cannot communicate with hosts in VRF BLUE.


## Running the demo

Step 1: Launch a CITC instance of flavor Blank Slate:
https://cumulusnetworks.com/products/cumulus-in-the-cloud/

Step 2: Log into the oob-mgmt-server of the blank slate

Step 3: Clone this repo:
git clone https://github.com/CumulusNetworks/ansible-evpn-symmetric.git

Step 4: Run the demo:
ansible-playbook deploy.yml


## Verification of output

### Connectivity
From server01, ping to server03 should work:
```
cumulus@server01:~$ ping 10.1.1.103
PING 10.1.1.103 (10.1.1.103) 56(84) bytes of data.
64 bytes from 10.1.1.103: icmp_seq=1 ttl=64 time=8.07 ms
64 bytes from 10.1.1.103: icmp_seq=2 ttl=64 time=6.50 ms
--- 10.1.1.103 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
```

From server01, ping to server02 should fail:
```
cumulus@server01:~$ ping 10.2.2.102
PING 10.2.2.102 (10.2.2.102) 56(84) bytes of data.
--- 10.2.2.102 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1007ms
```

### Control plane
Server01 should be able to resolve ARP directly for Server03
```
cumulus@server01:~$ ip neighbor show
10.1.1.103 dev uplink lladdr 44:38:39:00:0a:01 STALE
```

Leaf01 should be able to resolve MAC and ARP information for all servers:
```
cumulus@leaf01:~$ ip neighbor show
10.1.1.101 dev vlan10 lladdr 44:38:39:00:08:01 REACHABLE
10.2.2.102 dev vlan20 lladdr 44:38:39:00:09:01 REACHABLE
10.1.1.103 dev vlan10 lladdr 44:38:39:00:0a:01 offload NOARP
10.2.2.104 dev vlan20 lladdr 44:38:39:00:0b:01 offload NOARP
```

```
cumulus@leaf01:~$ bridge fdb show
46:38:39:00:08:01 dev SERVER01 vlan 10 master bridge
44:38:39:00:09:01 dev SERVER02 vlan 20 master bridge
44:38:39:00:0a:01 dev VXLAN10 vlan 10 offload master bridge
44:38:39:00:0a:01 dev VXLAN10 dst 192.168.1.34 self offload
44:38:39:00:0b:01 dev VXLAN20 vlan 20 offload master bridge
44:38:39:00:0b:01 dev VXLAN20 dst 192.168.1.34 self offload
```

Leaf01 should be able to resolve EVPN routes and MAC information:
```
cumulus@leaf01:~$ net show bgp l2vpn evpn route
BGP table version is 16, local router ID is 10.255.255.11
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete
EVPN type-2 prefix: [2]:[ESI]:[EthTag]:[MAClen]:[MAC]:[IPlen]:[IP]
EVPN type-3 prefix: [3]:[EthTag]:[IPlen]:[OrigIP]
EVPN type-5 prefix: [5]:[ESI]:[EthTag]:[IPlen]:[IP]

   Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 10.255.255.11:2
*> [2]:[0]:[0]:[48]:[44:38:39:00:08:01]
                    192.168.1.12                       32768 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:08:01]:[32]:[10.1.1.101]
                    192.168.1.12                       32768 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:08:01]:[128]:[fe80::4638:39ff:fe00:801]
                    192.168.1.12                       32768 i
*> [2]:[0]:[0]:[48]:[46:38:39:00:08:01]
                    192.168.1.12                       32768 i
*> [2]:[0]:[0]:[48]:[46:38:39:00:08:02]
                    192.168.1.12                       32768 i
*> [3]:[0]:[32]:[192.168.1.12]
                    192.168.1.12                       32768 i
Route Distinguisher: 10.255.255.11:5
*> [2]:[0]:[0]:[48]:[44:38:39:00:09:01]
                    192.168.1.12                       32768 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:09:01]:[32]:[10.2.2.102]
                    192.168.1.12                       32768 i
*> [3]:[0]:[32]:[192.168.1.12]
                    192.168.1.12                       32768 i
Route Distinguisher: 10.255.255.13:2
*  [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]
                    192.168.1.34                           0 65202 65103 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]
                    192.168.1.34                           0 65201 65103 i
*  [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]:[32]:[10.1.1.103]
                    192.168.1.34                           0 65202 65103 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]:[32]:[10.1.1.103]
                    192.168.1.34                           0 65201 65103 i
*  [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]:[128]:[fe80::4638:39ff:fe00:a01]
                    192.168.1.34                           0 65202 65103 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]:[128]:[fe80::4638:39ff:fe00:a01]
                    192.168.1.34                           0 65201 65103 i
*  [3]:[0]:[32]:[192.168.1.34]
                    192.168.1.34                           0 65201 65103 i
*> [3]:[0]:[32]:[192.168.1.34]
                    192.168.1.34                           0 65202 65103 i
Route Distinguisher: 10.255.255.13:4
*> [2]:[0]:[0]:[48]:[44:38:39:00:0b:01]
                    192.168.1.34                           0 65201 65103 i
*  [2]:[0]:[0]:[48]:[44:38:39:00:0b:01]
                    192.168.1.34                           0 65202 65103 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:0b:01]:[32]:[10.2.2.104]
                    192.168.1.34                           0 65201 65103 i
*  [2]:[0]:[0]:[48]:[44:38:39:00:0b:01]:[32]:[10.2.2.104]
                    192.168.1.34                           0 65202 65103 i
*  [3]:[0]:[32]:[192.168.1.34]
                    192.168.1.34                           0 65202 65103 i
*> [3]:[0]:[32]:[192.168.1.34]
                    192.168.1.34                           0 65201 65103 i
Route Distinguisher: 10.255.255.14:3
*  [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]
                    192.168.1.34                           0 65202 65104 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]
                    192.168.1.34                           0 65201 65104 i
*  [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]:[32]:[10.1.1.103]
                    192.168.1.34                           0 65202 65104 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]:[32]:[10.1.1.103]
                    192.168.1.34                           0 65201 65104 i
*  [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]:[128]:[fe80::4638:39ff:fe00:a01]
                    192.168.1.34                           0 65202 65104 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:0a:01]:[128]:[fe80::4638:39ff:fe00:a01]
                    192.168.1.34                           0 65201 65104 i
*  [3]:[0]:[32]:[192.168.1.34]
                    192.168.1.34                           0 65202 65104 i
*> [3]:[0]:[32]:[192.168.1.34]
                    192.168.1.34                           0 65201 65104 i
Route Distinguisher: 10.255.255.14:4
*> [2]:[0]:[0]:[48]:[44:38:39:00:0b:01]
                    192.168.1.34                           0 65201 65104 i
*  [2]:[0]:[0]:[48]:[44:38:39:00:0b:01]
                    192.168.1.34                           0 65202 65104 i
*  [2]:[0]:[0]:[48]:[44:38:39:00:0b:01]:[32]:[10.2.2.104]
                    192.168.1.34                           0 65202 65104 i
*> [2]:[0]:[0]:[48]:[44:38:39:00:0b:01]:[32]:[10.2.2.104]
                    192.168.1.34                           0 65201 65104 i
*  [3]:[0]:[32]:[192.168.1.34]
                    192.168.1.34                           0 65202 65104 i
*> [3]:[0]:[32]:[192.168.1.34]
                    192.168.1.34                           0 65201 65104 i

Displayed 23 prefixes (37 paths)
```

## Optional - Route leaking between the BLUE and RED VPN's.

Step by step instructions below, or you can run the following command to have Ansible deploy the changes for you.

cumulus@oob-mgmt-server:~$ ansible-playbook deploy-routeleaks.yml

### Server config changes:
Run "ip route" - note the default route points out the management eth0, for instance. If you want to ping 10.2.2.0 devices, you don’t have a route into the EVPN cloud. You’ll want to add a route to each server.

Server01 and server03 would get the following entry at the bottom of /etc/network/interfaces file: 

post-up ip route add 10.2.2.0/24 via 10.1.1.2

Server02 and server04 would have the following route:

post-up ip route add 10.1.1.0/24 via 10.2.2.2
```
cumulus@server01:~$ sudo systemctl restart networking
```
If you run “ip route” command again, you should now have a route to that remote subnet.

```
cumulus@server01:~$ ip route 
default via 192.168.0.254 dev eth0 
10.1.1.0/24 dev uplink  proto kernel  scope link  src 10.1.1.101 
10.2.2.0/24 via 10.1.1.2 dev uplink 
192.168.0.0/16 dev eth0  proto kernel  scope link  src 192.168.0.31 
cumulus@server01:~$ 
```

### Leaf Config changes:
Add VRF static route leaking per documentation:
https://docs.cumulusnetworks.com/display/DOCS/Virtual+Routing+and+Forwarding+-+VRF#VirtualRoutingandForwarding-VRF-EVPN_static_route_leakConfiguringStaticRouteLeakingwithEVPN

```
cumulus@leaf02:~$ net add routing route 10.1.1.0/24 RED vrf BLUE nexthop-vrf RED
cumulus@leaf02:~$ net add routing route 10.2.2.0/24 BLUE vrf RED nexthop-vrf BLUE
cumulus@leaf02:~$ net pending
cumulus@leaf02:~$ net commit
```

Repeat this process for each leaf.

### Run show commands, you should now see a static route between VRF's
...
cumulus@leaf01:~$ net show route vrf RED

show ip route vrf RED
======================
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route


VRF RED:
K * 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:05:57
C * 10.1.1.0/24 is directly connected, vlan10-v0, 00:05:05
C>* 10.1.1.0/24 is directly connected, vlan10, 00:05:05
S>* 10.2.2.0/24 [1/0] is directly connected, BLUE(vrf BLUE), 00:05:45



show ipv6 route vrf RED
========================
Codes: K - kernel route, C - connected, S - static, R - RIPng,
       O - OSPFv3, I - IS-IS, B - BGP, N - NHRP, T - Table,
       v - VNC, V - VNC-Direct, A - Babel, D - SHARP, F - PBR,
       > - selected route, * - FIB route


VRF RED:
K * ::/0 [255/8192] unreachable (ICMP unreachable)(vrf Default-IP-Routing-Table), 00:05:58
C * fe80::/64 is directly connected, vlan10-v0, 00:05:06
C * fe80::/64 is directly connected, vlan4001, 00:05:06
C>* fe80::/64 is directly connected, vlan10, 00:05:06
K>* ff00::/8 [0/256] is directly connected, vlan10-v0, 00:05:58

cumulus@leaf01:~$
```

```
cumulus@leaf01:~$ net show route vrf BLUE

show ip route vrf BLUE
=======================
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route


VRF BLUE:
K * 0.0.0.0/0 [255/8192] unreachable (ICMP unreachable), 00:06:34
S>* 10.1.1.0/24 [1/0] is directly connected, RED(vrf RED), 00:06:22
C * 10.2.2.0/24 is directly connected, vlan20-v0, 00:05:42
C>* 10.2.2.0/24 is directly connected, vlan20, 00:05:42



show ipv6 route vrf BLUE
=========================
Codes: K - kernel route, C - connected, S - static, R - RIPng,
       O - OSPFv3, I - IS-IS, B - BGP, N - NHRP, T - Table,
       v - VNC, V - VNC-Direct, A - Babel, D - SHARP, F - PBR,
       > - selected route, * - FIB route


VRF BLUE:
K * ::/0 [255/8192] unreachable (ICMP unreachable)(vrf Default-IP-Routing-Table), 00:06:35
C * fe80::/64 is directly connected, vlan20-v0, 00:05:43
C * fe80::/64 is directly connected, vlan4002, 00:05:43
C>* fe80::/64 is directly connected, vlan20, 00:05:43
K>* ff00::/8 [0/256] is directly connected, vlan20-v0, 00:06:35

cumulus@leaf01:~$
```

## Verification Test:
```
cumulus@server01:~$ ifconfig
uplink    Link encap:Ethernet  HWaddr 44:38:39:00:08:01
          inet addr:10.1.1.101  Bcast:10.1.1.255  Mask:255.255.255.0
          inet6 addr: fe80::4638:39ff:fe00:801/64 Scope:Link
          UP BROADCAST RUNNING MASTER MULTICAST  MTU:9000  Metric:1
          RX packets:18954 errors:0 dropped:4039 overruns:0 frame:0
          TX packets:14581 errors:0 dropped:1 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2112156 (2.1 MB)  TX bytes:1793866 (1.7 MB)
```

```
cumulus@server01:~$ ping 10.2.2.102
PING 10.2.2.102 (10.2.2.102) 56(84) bytes of data.
64 bytes from 10.2.2.102: icmp_seq=1 ttl=63 time=3.48 ms
64 bytes from 10.2.2.102: icmp_seq=2 ttl=63 time=3.12 ms

--- 10.2.2.102 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 3.128/3.304/3.481/0.185 ms
cumulus@server01:~$
```

```
cumulus@server01:~$ ping 10.2.2.104
PING 10.2.2.104 (10.2.2.104) 56(84) bytes of data.
64 bytes from 10.2.2.104: icmp_seq=1 ttl=62 time=12.6 ms
64 bytes from 10.2.2.104: icmp_seq=2 ttl=62 time=8.51 ms

--- 10.2.2.104 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 8.512/10.568/12.625/2.059 ms
cumulus@server01:~$
```


