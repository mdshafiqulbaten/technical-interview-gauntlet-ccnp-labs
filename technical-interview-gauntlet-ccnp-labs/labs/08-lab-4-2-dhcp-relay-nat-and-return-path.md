# Lab 4.2: DHCP Relay, NAT, and Return-Path

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 8083-8482). Review against the printed book before classroom use.*

Failure

1. THE TOPOLOGY & OBJECTIVE
A branch client must obtain an address from a centralized DHCP server
and then reach an external test server through PAT on BR1. The lab
contains three staged faults: the helper address is placed on the wrong
interface, the centralized network lacks a return route to the client subnet,
and the NAT ACL matches the wrong internal network.

Lab 4.2 topology

 CLIENT --- 10.20.20.0/24 --- BR1 --- 172.16.12.0/30 ---
 EDGE
                                       |
 |       \
                                  DHCP relay
 10.60.60.0/24 198.51.100.0/24

 |                           |

 DHCP-SRV                    INTERNET

 10.60.60.10                 198.51.100.10

                                                                        DHCP-SRV
                                                                          10.60.60.10

                                                                              Gi0/1 10.60.60.1

                                                    Gi0/1 172.16.12.2
      CLIENT         Gi0/0             BR1                               EDGE
     10.20.20.0/24                   10.20.20.1     Gi0/0 172.16.12.1   172.16.12.1

                                                                              Gi0/2 198.51.100.1

                                                                        INTERNET
                                                                        198.51.100.10

      Figure: Lab 4.2 DHCP Relay, NAT, and Return-Path Failure — Physical Topology

 Requirement

 Client on the user VLAN receives an address via DHCP relay

 ip helper-address is on the client-facing interface

 DHCP-SRV has routed reachability to the client subnet

 Edge router has a route back to the DHCP server's subnet

 NAT ACL matches the actual client subnet

 Dynamic NAT translation appears for the client's traffic

 Return traffic routes correctly through the NAT device

 End-to-end application reachability confirmed after repair

The final working state requires a DHCP lease in 10.20.20.0/24, default
gateway 10.20.20.1, a visible server binding, a PAT translation using

172.16.12.2, and successful client reachability to 198.51.100.10.

2. THE INITIAL GOLDEN CONFIG

BR1 Golden Configuration

 hostname BR1
 service dhcp
 !
 interface GigabitEthernet0/0
  description BRANCH-USERS
  ip address 10.20.20.1 255.255.255.0
  ip helper-address 10.60.60.10
  ip nat inside
  no shutdown
 !
 interface GigabitEthernet0/1
  description TO-EDGE
  ip address 172.16.12.2 255.255.255.252
  ip nat outside
  no shutdown
 !
 ip access-list standard NAT-USERS
  10 permit 10.20.20.0 0.0.0.255
 !
 ip nat inside source list NAT-USERS interface
 GigabitEthernet0/1 overload
 ip route 0.0.0.0 0.0.0.0 172.16.12.1

EDGE Golden Configuration

 hostname EDGE
 interface GigabitEthernet0/0
  description TO-BR1
  ip address 172.16.12.1 255.255.255.252
  no shutdown
 interface GigabitEthernet0/1
  description TO-DHCP-SRV
  ip address 10.60.60.1 255.255.255.0
  no shutdown
 interface GigabitEthernet0/2
  description TO-INTERNET
  ip address 198.51.100.1 255.255.255.0
  no shutdown
 ip route 10.20.20.0 255.255.255.0 172.16.12.2

DHCP-SRV Golden Configuration

 hostname DHCP-SRV
 service dhcp
 interface GigabitEthernet0/0
  ip address 10.60.60.10 255.255.255.0
  no shutdown
 ip route 0.0.0.0 0.0.0.0 10.60.60.1
 ip dhcp excluded-address 10.20.20.1 10.20.20.20
 ip dhcp pool BRANCH-USERS
  network 10.20.20.0 255.255.255.0
  default-router 10.20.20.1
  dns-server 192.0.2.53
  lease 0 8

INTERNET Golden Configuration

 hostname INTERNET
 interface GigabitEthernet0/0
  ip address 198.51.100.10 255.255.255.0
  no shutdown
 ip route 0.0.0.0 0.0.0.0 198.51.100.1

CLIENT Golden Configuration

 hostname CLIENT
 interface GigabitEthernet0/0
  ip address dhcp
  no shutdown

Golden-State Verification

 CLIENT# show ip interface brief
 CLIENT# show dhcp lease
 DHCP-SRV# show ip dhcp binding
 BR1# show ip nat translations
 CLIENT# ping 198.51.100.10

3. THE SABOTAGE SCRIPT
Lab 4.2 sabotage

 BR1# configure terminal
 BR1(config)# interface GigabitEthernet0/0
 BR1(config-if)# no ip helper-address 10.60.60.10
 BR1(config-if)# exit
 BR1(config)# interface GigabitEthernet0/1
 BR1(config-if)# ip helper-address 10.60.60.10
 BR1(config-if)# exit
 BR1(config)# ip access-list standard NAT-USERS
 BR1(config-std-nacl)# no 10
 BR1(config-std-nacl)# 10 permit 10.20.30.0 0.0.0.255
 BR1(config-std-nacl)# end
 !
 EDGE# configure terminal
 EDGE(config)# no ip route 10.20.20.0 255.255.255.0
 172.16.12.2
 EDGE(config)# end
 !
 DHCP-SRV# clear ip dhcp binding *
 CLIENT# configure terminal
 CLIENT(config)# interface GigabitEthernet0/0
 CLIENT(config-if)# shutdown
 CLIENT(config-if)# no shutdown
 CLIENT(config-if)# end

The initial symptom is that CLIENT receives no DHCP address.
Correcting the helper placement exposes the missing return route.
Correcting the return route allows DHCP to work, but external traffic still
fails because the NAT ACL does not match the leased client subnet.

4. THE VERIFICATION WORKFLOW

Step 1: Establish the Client State

 CLIENT# show ip interface brief
 CLIENT# show dhcp lease
 CLIENT# show running-config interface GigabitEthernet0/0

The interface is configured for DHCP but has no usable address. Do not
start by configuring a static address, because that bypasses the service
being tested.

Step 2: Verify the Client-Facing Relay Interface

 BR1# show ip interface GigabitEthernet0/0
 BR1# show running-config interface GigabitEthernet0/0
 BR1# show running-config | include helper-address

The user-facing Gi0/0 receives the DHCP broadcast but has no helper
address. The helper is incorrectly configured on Gi0/1, which receives
routed traffic rather than the original client broadcast.

Step 3: Form the First Hypothesis

 Evidence                 Hypothesis

 No DHCP offer reaches    ip helper-address configured on the wrong (routed, not
 the client               client-facing) interface

Step 4: Correct Helper Placement

 BR1# configure terminal
 BR1(config)# interface GigabitEthernet0/1
 BR1(config-if)# no ip helper-address 10.60.60.10
 BR1(config-if)# exit
 BR1(config)# interface GigabitEthernet0/0
 BR1(config-if)# ip helper-address 10.60.60.10
 BR1(config-if)# end

Step 5: Verify Forward Reachability to the Server

 BR1# show ip route 10.60.60.10
 BR1# ping 10.60.60.10 source 10.20.20.1
 BR1# traceroute 10.60.60.10 source 10.20.20.1

BR1 should reach the server through EDGE. If a sourced ping fails,
inspect whether the server side has a route back to 10.20.20.1.

Step 6: Trigger a New DHCP Attempt

 CLIENT# configure terminal
 CLIENT(config)# interface GigabitEthernet0/0
 CLIENT(config-if)# shutdown
 CLIENT(config-if)# no shutdown
 CLIENT(config-if)# end
 CLIENT# show ip interface brief

The client may still fail to receive an address. The next step is to inspect
both the server and the return path rather than repeatedly bouncing the
client interface.

Step 7: Inspect Server Allocation State

 DHCP-SRV# show ip dhcp pool BRANCH-USERS
 DHCP-SRV# show ip dhcp binding
 DHCP-SRV# show logging | include DHCP

Depending on timing and platform, the server may show an offered or
bound address even though the client never receives the response. This is
strong evidence that the request reached the server and the failure is on the
reply path.

Step 8: Verify the Return Route

 DHCP-SRV# show ip route 10.20.20.0
 DHCP-SRV# traceroute 10.20.20.1
 EDGE# show ip route 10.20.20.0
 EDGE# show running-config | include 10.20.20.0

DHCP-SRV sends traffic toward its default gateway. EDGE lacks the
specific route to 10.20.20.0/24 and cannot forward the response to BR1.

Step 9: Form the Second Hypothesis

 Evidence                             Hypothesis

 DHCP-SRV cannot reach the client     EDGE router is missing a specific route to
 subnet with its reply                the client's subnet

Step 10: Restore the Return Route

 EDGE# configure terminal
 EDGE(config)# ip route 10.20.20.0 255.255.255.0
 172.16.12.2
 EDGE(config)# end
 EDGE# show ip route 10.20.20.0

Step 11: Verify DHCP End to End

 DHCP-SRV# clear ip dhcp binding *
 CLIENT# configure terminal
 CLIENT(config)# interface GigabitEthernet0/0
 CLIENT(config-if)# shutdown
 CLIENT(config-if)# no shutdown
 CLIENT(config-if)# end
 CLIENT# show ip interface brief
 CLIENT# show dhcp lease
 DHCP-SRV# show ip dhcp binding

The client should now receive an address in 10.20.20.0/24, a /24 mask,
default gateway 10.20.20.1, and the configured DNS server.

Step 12: Test External Reachability

 CLIENT# ping 10.20.20.1
 CLIENT# ping 198.51.100.10
 CLIENT# traceroute 198.51.100.10

The client reaches BR1 but still fails to reach the external server. Since
addressing and the default gateway are now correct, inspect BR1 routing
and NAT.

Step 13: Verify BR1 Routing and NAT Roles

 BR1# show ip route 198.51.100.10
 BR1# show ip interface GigabitEthernet0/0
 BR1# show ip interface GigabitEthernet0/1
 BR1# show ip nat statistics

BR1 has a default route. Gi0/0 is NAT inside and Gi0/1 is NAT outside.
The translation rule exists, so the match policy becomes the next focus.

Step 14: Inspect Translation and ACL Counters
