The comprehensive demo built on cldemo-vagrant is here:
https://github.com/CumulusNetworks/cldemo-evpn-symmetric

# ansible-evpn-symmetric

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
