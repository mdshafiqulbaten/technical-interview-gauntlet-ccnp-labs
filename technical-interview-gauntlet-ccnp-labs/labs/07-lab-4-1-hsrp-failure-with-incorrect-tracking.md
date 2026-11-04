# Lab 4.1: HSRP Failure with Incorrect Tracking

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 7559-7958). Review against the printed book before classroom use.*

1. THE TOPOLOGY & OBJECTIVE
A campus user VLAN has redundant distribution switches. DSW1 is the
preferred HSRP Active gateway during normal operation. DSW2 must
take over when DSW1 loses usable upstream reachability, even if DSW1
physical interfaces remain up. When DSW1 service is restored, delayed
preemption should return the Active role to DSW1.

Lab 4.1 topology

                            TEST-SRV
                         Lo0 203.0.113.10
                            /          \
                       EDGE1          EDGE2
                    10.0.11.1      10.0.12.1
                         |              |
                    10.0.11.2      10.0.12.2
                       DSW1            DSW2
                         \             /
                          \ VLAN 10   /
                            ASW1
                             |
                          CLIENT
                     10.10.10.50/24
                Default gateway 10.10.10.1

                                                 TEST-SRV
                                                Lo0 203.0.113.10

                                   Gi0/0 192.0.2.2                      Gi0/1 192.0.2.6
                 Gi0/1 192.0.2.1                                                     Gi0/1 192.0.2.5

              EDGE1                                                                       EDGE2
               192.0.2.1                                                                  192.0.2.5

                    Gi0/0 10.0.11.1                                                             Gi0/0 10.0.12.1

                    Gi0/1 10.0.11.2                                                             Gi0/1 10.0.12.2

               DSW1                                                                       DSW2
              10.10.10.2                                                                  10.10.10.3

                   VLAN 10                                                     VLAN 10

                                                       ASW1
                                                       VLAN 10

                                                             Gi0/3 TO-CLIENT

                                                      CLIENT
                                                     10.10.10.50/24

         Figure: Lab 4.1 HSRP Failure with Incorrect Tracking — Physical Topology

 Requirement

 DSW1 and DSW2 form an HSRP group with DSW1 preferred Active

 IP SLA operation tracks reachability to a defined target

 Object tracking ties the IP SLA result to the HSRP priority decrement

 Preemption is enabled so DSW1 reclaims Active after recovery

 Decrement value is large enough to force a role change on failure

 Failover causes minimal packet loss

 End-to-end reachability confirmed through the virtual gateway throughout the test

The required final behavior is: DSW1 Active when its IP SLA succeeds;
DSW2 Active when DSW1 cannot reach the test endpoint; DSW1
effective priority below 100 during failure; and automatic return to DSW1
after service recovery and the preemption delay.

2. THE INITIAL GOLDEN CONFIG

ASW1 Golden Configuration

 hostname ASW1
 spanning-tree mode rapid-pvst
 vlan 10
  name USERS
 !
 interface GigabitEthernet0/1
  description TO-DSW1
  switchport mode access
  switchport access vlan 10
  no shutdown
 !
 interface GigabitEthernet0/2
  description TO-DSW2
  switchport mode access
  switchport access vlan 10
  no shutdown
 !
 interface GigabitEthernet0/3
  description TO-CLIENT
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast
  no shutdown

DSW1 Golden Configuration

 hostname DSW1
 ip routing
 !
 interface Vlan10
  description USER-GATEWAY
  ip address 10.10.10.2 255.255.255.0
  standby version 2
  standby 10 ip 10.10.10.1
  standby 10 priority 110
  standby 10 preempt delay minimum 30
  standby 10 track 10 decrement 30
  no shutdown
 !
 interface GigabitEthernet0/1
  description TO-EDGE1
  no switchport
  ip address 10.0.11.2 255.255.255.252
  no shutdown
 !
 ip route 0.0.0.0 0.0.0.0 10.0.11.1
 !
 ip sla 10
  icmp-echo 203.0.113.10 source-interface
 GigabitEthernet0/1
  timeout 1000
  frequency 5
 ip sla schedule 10 life forever start-time now
 track 10 ip sla 10 reachability

DSW2 Golden Configuration

 hostname DSW2
 ip routing
 !
 interface Vlan10
  description USER-GATEWAY
  ip address 10.10.10.3 255.255.255.0
  standby version 2
  standby 10 ip 10.10.10.1
  standby 10 priority 100
  standby 10 preempt delay minimum 30
  no shutdown
 !
 interface GigabitEthernet0/1
  description TO-EDGE2
  no switchport
  ip address 10.0.12.2 255.255.255.252
  no shutdown
 !
 ip route 0.0.0.0 0.0.0.0 10.0.12.1

EDGE1 Golden Configuration

 hostname EDGE1
 interface GigabitEthernet0/0
  description TO-DSW1
  ip address 10.0.11.1 255.255.255.252
  no shutdown
 interface GigabitEthernet0/1
  description TO-TEST-SRV
  ip address 192.0.2.1 255.255.255.252
  no shutdown
 ip route 203.0.113.10 255.255.255.255 192.0.2.2
 ip route 10.10.10.0 255.255.255.0 10.0.11.2

EDGE2 Golden Configuration

 hostname EDGE2
 interface GigabitEthernet0/0
  description TO-DSW2
  ip address 10.0.12.1 255.255.255.252
  no shutdown
 interface GigabitEthernet0/1
  description TO-TEST-SRV
  ip address 192.0.2.5 255.255.255.252
  no shutdown
 ip route 203.0.113.10 255.255.255.255 192.0.2.6
 ip route 10.10.10.0 255.255.255.0 10.0.12.2

TEST-SRV Golden Configuration

 hostname TEST-SRV
 interface Loopback0
  ip address 203.0.113.10 255.255.255.255
 interface GigabitEthernet0/0
  description TO-EDGE1
  ip address 192.0.2.2 255.255.255.252
  no shutdown
 interface GigabitEthernet0/1
  description TO-EDGE2
  ip address 192.0.2.6 255.255.255.252
  no shutdown
 ip route 10.0.11.0 255.255.255.252 192.0.2.1
 ip route 10.0.12.0 255.255.255.252 192.0.2.5
 ip route 10.10.10.0 255.255.255.0 192.0.2.1 10
 ip route 10.10.10.0 255.255.255.0 192.0.2.5 20

CLIENT Golden Configuration

 hostname CLIENT
 interface GigabitEthernet0/0
  ip address 10.10.10.50 255.255.255.0
  no shutdown
 ip route 0.0.0.0 0.0.0.0 10.10.10.1

Golden-State Verification

 DSW1# show standby brief
 DSW1# show track 10
 DSW1# show ip sla statistics 10
 DSW2# show standby brief
 CLIENT# ping 203.0.113.10

Expected normal state: DSW1 is Active with effective priority 110, DSW2
is Standby with priority 100, track 10 is Up, the IP SLA succeeds, and the
client reaches 203.0.113.10 through the virtual gateway.

3. THE SABOTAGE SCRIPT
The interviewer injects three related conditions. The Ethernet uplink
remains up, but the default route disappears. The tracking object is
changed from IP SLA reachability to interface line protocol. The HSRP
decrement is reduced so that even a later tracking failure is not large
enough to move the Active role.

Lab 4.1 sabotage

 DSW1# configure terminal
 DSW1(config)# no track 10
 DSW1(config)# track 10 interface GigabitEthernet0/1 line-
 protocol
 DSW1(config)# interface Vlan10
 DSW1(config-if)# no standby 10 track 10 decrement 30
 DSW1(config-if)# standby 10 track 10 decrement 5
 DSW1(config-if)# exit
 DSW1(config)# no ip route 0.0.0.0 0.0.0.0 10.0.11.1
 DSW1(config)# end

The intended symptom is that DSW1 remains HSRP Active even though it
has no default route. The client sends traffic to the virtual MAC owned by
DSW1, and DSW1 drops destinations for which it has no route.

4. THE VERIFICATION WORKFLOW

Step 1: Confirm the Client Symptom

 CLIENT# show ip route
 CLIENT# ping 10.10.10.1
 CLIENT# ping 10.10.10.2
 CLIENT# ping 203.0.113.10
 CLIENT# traceroute 203.0.113.10

The client should reach the virtual gateway and DSW1 physical SVI but
fail to reach the remote test address. This separates first-hop LAN
reachability from upstream forwarding.

Step 2: Identify the HSRP Owner

 DSW1# show standby brief
 DSW1# show standby vlan 10
 DSW2# show standby brief

Look for the Active and Standby roles, local and peer priorities, virtual IP,
preemption state, tracked objects, and last state change. DSW1 remains
Active with effective priority 110 because its line-protocol object is still
Up.

Step 3: Verify the Active Router Forwarding Path

 DSW1# show ip route 0.0.0.0
 DSW1# show ip route 203.0.113.10
 DSW1# ping 203.0.113.10 source GigabitEthernet0/1
 DSW1# traceroute 203.0.113.10 source GigabitEthernet0/1

The default route is missing. The interface can remain up/up because loss
of routing information is not the same as loss of Ethernet carrier.

Step 4: Inspect the Tracked Object

 DSW1# show track 10
 DSW1# show running-config | section track
 DSW1# show running-config interface Vlan10
 DSW1# show ip sla statistics 10

The IP SLA operation may show failure because it cannot reach the target,
but HSRP is not tracking it. Track 10 has been redefined as interface line
protocol, which remains Up.

Step 5: Form the First Hypothesis

 Evidence                   Hypothesis

 IP SLA shows failure but   Track object is bound to the wrong target (interface line
 HSRP role unchanged        protocol instead of the IP SLA result)

Step 6: Restore Service-Aware Tracking

 DSW1# configure terminal
 DSW1(config)# no track 10
 DSW1(config)# track 10 ip sla 10 reachability
 DSW1(config)# end

The IP SLA is already scheduled. With the default route absent, the
operation should fail and track 10 should transition Down.

Step 7: Observe the Remaining Priority Problem

 DSW1# show track 10
 DSW1# show standby vlan 10
 DSW2# show standby brief

Track 10 is now Down, but DSW1 effective priority is 105 because the
configured decrement is only 5. DSW2 remains priority 100, so DSW1 is
still more preferred.

Step 8: Form the Second Hypothesis

 Evidence                  Hypothesis

 Track fires but DSW1      Configured decrement is too small to make DSW1 less
 remains Active            preferred than DSW2
