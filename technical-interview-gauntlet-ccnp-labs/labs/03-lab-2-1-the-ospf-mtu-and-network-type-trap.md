# Lab 2.1: The OSPF MTU and Network-Type Trap

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 3459-3858). Review against the printed book before classroom use.*

Recommended time limit: 35 minutes

 Objective

 Identify an OSPF network-type mismatch without assuming a cabling fault

 Diagnose a secondary MTU mismatch hidden behind the first fault

1. THE TOPOLOGY & OBJECTIVE
A three-router enterprise uses Area 0 between the core and distribution
router and Area 10 between the distribution and branch router. A recent
circuit migration changed interface settings on the R1-to-R2 link. R3 can
still reach R2, but R1 no longer learns the branch loopback and the
monitoring system reports that the R1-R2 OSPF adjacency is down.

Lab topology

                                                     AREA 0
 AREA 10

  Loopback0              Gi0/0          Gi0/0
 Gi0/1           Gi0/0        Loopback0
  1.1.1.1/32       R1 --------- 10.12.12.0/30 --------- R2
 --------- 10.23.23.0/30 --------- R3   3.3.3.3/32
                       .1                          .2
 .2                            .3

 Intended OSPF network type on both routed links: point-to-
 point
 Intended MTU on R1-R2: 1500 bytes
 OSPF process: 100

                                   AREA 0                                        AREA 10

                         Gi0/0               Gi0/0                      Gi0/1               Gi0/0
             R1                                             R2                                           R3
                                 10.12.12.0/30                                  10.23.23.0/30

        Lo0 1.1.1.1/32                                 Lo0 2.2.2.2/32                               Lo0 3.3.3.3/32

                                   Figure: Lab 2.1 — Physical Topology

 Requirement

 R1-R2 link forms a stable OSPF adjacency

 R2-R3 link remains Full throughout the fault

 Network type matches on both ends of each link

 IP MTU matches on both ends of each link

 Database Description exchange completes without stalling

 Full end-to-end reachability after repair

The final working state must satisfy all of the following:

    R1 and R2 are Full OSPF neighbors on Gi0/0.

    R2 and R3 are Full OSPF neighbors on the Area 10 link.

   R1 learns 3.3.3.3/32 as an interarea route.

   R3 learns 1.1.1.1/32 as an interarea route.

   The R1-R2 link uses OSPF point-to-point network type on both ends.

   The R1-R2 IP MTU is consistent and supports ordinary 1500-byte
   Ethernet forwarding.

   No `ip ospf mtu-ignore` workaround remains in the final
   configuration.

2. THE INITIAL GOLDEN CONFIG
R1 golden configuration

 enable
 configure terminal
 hostname R1
 no ip domain-lookup
 !
 interface Loopback0
  ip address 1.1.1.1 255.255.255.255
  ip ospf 100 area 0
 !
 interface GigabitEthernet0/0
  description AREA-0-LINK-TO-R2
  ip address 10.12.12.1 255.255.255.252
  ip ospf 100 area 0
  ip ospf network point-to-point
  no shutdown
 !
 router ospf 100
  router-id 1.1.1.1
  passive-interface Loopback0
  log-adjacency-changes detail
 end
 write memory

R2 golden configuration

 enable
 configure terminal
 hostname R2
 no ip domain-lookup
 !
 interface Loopback0
  ip address 2.2.2.2 255.255.255.255
  ip ospf 100 area 0
 !
 interface GigabitEthernet0/0
  description AREA-0-LINK-TO-R1
  ip address 10.12.12.2 255.255.255.252
  ip ospf 100 area 0
  ip ospf network point-to-point
  no shutdown
 !
 interface GigabitEthernet0/1
  description AREA-10-LINK-TO-R3
  ip address 10.23.23.2 255.255.255.252
  ip ospf 100 area 10
  ip ospf network point-to-point
  no shutdown
 !
 router ospf 100
  router-id 2.2.2.2
  passive-interface Loopback0
  log-adjacency-changes detail
 end
 write memory

R3 golden configuration

 enable
 configure terminal
 hostname R3
 no ip domain-lookup
 !
 interface Loopback0
  ip address 3.3.3.3 255.255.255.255
  ip ospf 100 area 10
 !
 interface GigabitEthernet0/0
  description AREA-10-LINK-TO-R2
  ip address 10.23.23.3 255.255.255.252
  ip ospf 100 area 10
  ip ospf network point-to-point
  no shutdown
 !
 router ospf 100
  router-id 3.3.3.3
  passive-interface Loopback0
  log-adjacency-changes detail
 end
 write memory

Golden-state verification on R1

 show ip ospf neighbor
 show ip ospf interface brief
 show ip route ospf
 ping 3.3.3.3 source 1.1.1.1

Expected R1 route:

 O IA 3.3.3.3/32 [110/2] via 10.12.12.2, GigabitEthernet0/0

The exact metric may differ by platform reference bandwidth, but the route
must be interarea and use R2 as the next hop.

3. THE SABOTAGE SCRIPT
Run both faults on R2. Do not reveal them to the candidate.

R2 sabotage commands

 configure terminal
 interface GigabitEthernet0/0
  ip ospf network non-broadcast
  ip mtu 1400
 end

Fault 1 changes R2 to the nonbroadcast network type. This changes
neighbor discovery expectations and common timer defaults. No manual
OSPF neighbor statement is configured.

Fault 2 reduces the IP MTU on R2 Gi0/0 to 1400 while R1 remains at
1500. After the network type is repaired, the routers can exchange Hellos
but fail during Database Description negotiation.

 Fault     Change

 Fault 1   Network type changed on one side of the R1-R2 link

 Fault 2   IP MTU reduced to 1400 on R2 Gi0/0 while R1 remains at 1500

4. THE VERIFICATION WORKFLOW

Step 1: Establish Layer 3 Reachability

 R1# show ip interface brief
 R1# show interfaces GigabitEthernet0/0
 R1# ping 10.12.12.2

The interface should be up/up and a normal small ping should succeed.
This proves basic unicast IP reachability, but it does not prove that OSPF
parameters match.

Step 2: Read the Neighbor State Before Debugging

 R1# show ip ospf neighbor
 R1# show ip ospf interface brief

R1 may show no neighbor on Gi0/0. R2-to-R3 should remain Full, which
limits the failure domain to the R1-R2 link rather than the complete OSPF
process.

 Link         Expected Neighbor State

 R1-R2        No neighbor, or stuck below Full

 R2-R3        Full (confirms the failure domain is limited to R1-R2)

Step 3: Compare Interface Parameters on Both Ends

 R1# show ip ospf interface GigabitEthernet0/0
 R2# show ip ospf interface GigabitEthernet0/0

On R1, hunt for lines resembling:

 Process ID 100, Router ID 1.1.1.1, Network Type
 POINT_TO_POINT
 Timer intervals configured, Hello 10, Dead 40

On R2, hunt for:

 Process ID 100, Router ID 2.2.2.2, Network Type
 NON_BROADCAST
 Timer intervals configured, Hello 30, Dead 120

The exact wording varies, but the network types and timer expectations are
visibly different.

Step 4: State the First Hypothesis

 Evidence                                          Hypothesis

 Visible network-type and timer mismatch in        OSPF network type differs between
 neighbor output                                   R1 and R2

Step 5: Correct Only the Network Type

 R2# configure terminal
 R2(config)# interface GigabitEthernet0/0
 R2(config-if)# ip ospf network point-to-point
 R2(config-if)# end

Do not change the MTU yet. The purpose of a disciplined lab is to observe
what the first correction reveals.

Step 6: Observe the New Failure State

 R1# show ip ospf neighbor
 R2# show ip ospf neighbor

The neighbor should now appear but remain in ExStart or Exchange.
Representative output:

 Neighbor ID        Pri     State              Dead Time     Address
 Interface
 2.2.2.2              0   EXSTART/-            00:00:36
 10.12.12.2      GigabitEthernet0/0

The network type repair was successful because bidirectional Hellos now
progress beyond the discovery stage. Database synchronization is failing
next.

Step 7: Compare MTU Values

 R1# show interfaces GigabitEthernet0/0 | include MTU
 R1# show ip ospf interface GigabitEthernet0/0 | include
 MTU
 R2# show interfaces GigabitEthernet0/0 | include MTU
 R2# show ip interface GigabitEthernet0/0 | include MTU

Depending on the image, `show interfaces` may display the physical
interface MTU while `show ip interface` exposes the configured IP MTU.
The candidate must compare the value OSPF places in Database
Description packets, not rely on one command label.

Expected evidence

 R1 IP MTU: 1500
 R2 IP MTU: 1400

Step 8: Use a Targeted Debug

In an isolated lab, enable a narrow debug. In production, explain the risk
and use conditional debugging where supported.

 R1# terminal monitor
 R1# debug ip ospf adj

 R1# undebug all

Look for repeated DBD negotiation, sequence restarts, or an MTU
mismatch indication. Stop debugging as soon as sufficient evidence is
collected.

Step 9: Prove the Data-Plane MTU Difference

 R1# ping 10.12.12.2 repeat 5 size 1372 df-bit
 R1# ping 10.12.12.2 repeat 5 size 1472 df-bit

A 1372-byte ICMP payload plus IP and ICMP headers fits inside a 1400-
byte IP MTU. A 1472-byte payload requires a 1500-byte IP packet.
Command syntax varies, so use an extended ping if the one-line form is
unavailable.

Step 10: State the Second Hypothesis

 Test                                           Result If MTU Is the Fault
