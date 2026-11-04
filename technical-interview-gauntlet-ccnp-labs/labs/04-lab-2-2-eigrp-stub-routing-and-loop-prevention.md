# Lab 2.2: EIGRP Stub Routing and Loop Prevention

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 3997-4396). Review against the printed book before classroom use.*

Recommended time limit: 40 minutes

 Objective

 Diagnose an EIGRP stub adjacency and query-scoping issue under time pressure

1. THE TOPOLOGY & OBJECTIVE
A remote branch connects to the enterprise hub through EIGRP named
mode. The branch owns two user subnets inside 10.50.0.0/16 and uses a
static default route toward the hub for all nonbranch destinations. The
branch should advertise one EIGRP summary and operate as a stub so the
hub does not query it for unrelated enterprise routes.

Lab topology

                                                 EIGRP AS 100

       Loopback0          Gi0/0                 Gi0/0
 Branch loopbacks
       1.1.1.1/32     HUB1 -------- 10.12.12.0/30 --------
 BR1   Lo10 10.50.10.1/24
                                       .1             .2
 Lo20 10.50.20.1/24

 BR1 default route: 0.0.0.0/0 via 10.12.12.1
 Healthy advertisement to HUB1: 10.50.0.0/16 through EIGRP
 Healthy BR1 protection: local EIGRP summary route to Null0

                                  EIGRP AS 100

                              Gi0/0                Gi0/0
             HUB1                                                BR1
                                      10.12.12.0/30

           Lo0 1.1.1.1/32                                  Lo10 10.50.10.1/24
                                                           Lo20 10.50.20.1/24

                            Figure: Lab 2.2 — Physical Topology

 Requirement

 Branch router (BR1) forms a stable EIGRP adjacency with the hub

 BR1 is configured as an EIGRP stub (connected, summary)

 Hub does not send unnecessary queries to BR1 for unrelated routes

 Loop prevention via the automatic Null0 discard route is intact

 End-to-end reachability confirmed after repair

The final working state must satisfy all of the following:

    HUB1 and BR1 maintain an EIGRP named-mode adjacency in AS
    100.

    HUB1 identifies BR1 as a stub that advertises connected and summary
    routes.

    HUB1 learns 10.50.0.0/16 through EIGRP, not through a manual static
    route.

    BR1 installs the EIGRP summary discard route for 10.50.0.0/16
    through Null0.

    HUB1 reaches both existing branch loopbacks.

    Traffic to 10.50.99.99 does not loop between HUB1 and BR1.

    BR1 retains its default route toward HUB1 for nonbranch destinations.

2. THE INITIAL GOLDEN CONFIG
HUB1 golden configuration

 enable
 configure terminal
 hostname HUB1
 no ip domain-lookup
 !
 interface Loopback0
  ip address 1.1.1.1 255.255.255.255
 !
 interface GigabitEthernet0/0
  description WAN-TO-BR1
  ip address 10.12.12.1 255.255.255.252
  no shutdown
 !
 router eigrp ENTERPRISE
  address-family ipv4 unicast autonomous-system 100
   eigrp router-id 1.1.1.1
   network 1.1.1.1 0.0.0.0
   network 10.12.12.0 0.0.0.3
   af-interface default
     passive-interface
   exit-af-interface
   af-interface GigabitEthernet0/0
     no passive-interface
   exit-af-interface
   topology base
     maximum-paths 4
   exit-af-topology
  exit-address-family
 end
 write memory

BR1 golden configuration

 enable
 configure terminal
 hostname BR1
 no ip domain-lookup
 !
 interface GigabitEthernet0/0
  description WAN-TO-HUB1
  ip address 10.12.12.2 255.255.255.252
  no shutdown
 !
 interface Loopback10
  ip address 10.50.10.1 255.255.255.0
 !
 interface Loopback20
  ip address 10.50.20.1 255.255.255.0
 !
 ip route 0.0.0.0 0.0.0.0 10.12.12.1
 !
 router eigrp ENTERPRISE
  address-family ipv4 unicast autonomous-system 100
   eigrp router-id 2.2.2.2
   network 10.12.12.0 0.0.0.3
   network 10.50.10.0 0.0.0.255
   network 10.50.20.0 0.0.0.255
   eigrp stub connected summary
   af-interface default
     passive-interface
   exit-af-interface
   af-interface GigabitEthernet0/0
     no passive-interface
     summary-address 10.50.0.0 255.255.0.0
   exit-af-interface
   topology base
   exit-af-topology
  exit-address-family
 end
 write memory

Golden-state verification

 HUB1# show eigrp address-family ipv4 neighbors detail
 HUB1# show ip route 10.50.0.0
 HUB1# ping 10.50.10.1 source 1.1.1.1
 HUB1# ping 10.50.20.1 source 1.1.1.1

 BR1# show ip route 10.50.0.0
 BR1# show eigrp address-family ipv4 interfaces detail

Expected HUB1 route:

 D 10.50.0.0/16 [90/<metric>] via 10.12.12.2,
 GigabitEthernet0/0

Expected BR1 summary discard route:

 Routing entry for 10.50.0.0/16
   Known via "eigrp 100", distance 5
   * directly connected, via Null0

3. THE SABOTAGE SCRIPT
The first fault suppresses all branch advertisements. The second and third
commands imitate an emergency workaround that restores current branch
reachability but removes loop protection.

Sabotage commands

 BR1# configure terminal
 BR1(config)# router eigrp ENTERPRISE
 BR1(config-router)# address-family ipv4 unicast
 autonomous-system 100
 BR1(config-router-af)# eigrp stub receive-only
 BR1(config-router-af)# af-interface GigabitEthernet0/0
 BR1(config-router-af-interface)# no summary-address
 10.50.0.0 255.255.0.0
 BR1(config-router-af-interface)# end

 HUB1# configure terminal
 HUB1(config)# ip route 10.50.0.0 255.255.0.0 10.12.12.2
 HUB1(config)# end

The adjacency remains up because the autonomous system and neighbor
parameters still match. HUB1 can reach 10.50.10.1 and 10.50.20.1 through
the manual static route. A quick ping test therefore appears to confirm that
the emergency repair worked.

However, BR1 still has a default route back to HUB1. Without the local
EIGRP summary discard route, a packet for an unused address such as
10.50.99.99 follows this path:

 HUB1 static 10.50.0.0/16 -> BR1
 BR1 has no 10.50.99.0/24 route -> default route to HUB1
 HUB1 static 10.50.0.0/16 -> BR1
 BR1 default route -> HUB1
 ...until TTL expires

4. THE VERIFICATION WORKFLOW

Step 1: Verify the Adjacency Without Assuming Routes Exist

 HUB1# show eigrp address-family ipv4 neighbors
 HUB1# show eigrp address-family ipv4 neighbors detail

The neighbor should be up. In the detailed output, find the stub capability.
Depending on the release, the output may state that the peer is a receive-
only stub or show its advertised stub attributes.

 Check                         Expected

 Neighbor state                Up

 Stub capability in detailed   Peer advertises itself as a stub (e.g. receive-only or its
 output                        configured stub attributes)

Step 2: Identify the Route Source on HUB1

 HUB1# show ip route 10.50.0.0
 HUB1# show ip route 10.50.10.1
 HUB1# show ip route eigrp
 HUB1# show running-config | include ^ip route

The route to 10.50.0.0/16 should be marked static rather than EIGRP:

 S 10.50.0.0/16 [1/0] via 10.12.12.2

The static administrative distance of 1 also masks any future EIGRP
summary with administrative distance 90. Leaving it in place after
repairing EIGRP would prevent the candidate from proving that the
dynamic route is being used.

Step 3: Confirm That Current Branch Prefixes Appear Reachable

 HUB1# ping 10.50.10.1 source 1.1.1.1
 HUB1# ping 10.50.20.1 source 1.1.1.1

Both tests may succeed. This is deliberately misleading. The existing
loopbacks are directly connected to BR1, so the branch can deliver them
even though the hub is using an unsafe summary.

Step 4: Test an Unused Address Inside the Summary

 HUB1# traceroute 10.50.99.99 source 1.1.1.1

    1 10.12.12.2
    2 10.12.12.1
    3 10.12.12.2
    4 10.12.12.1
    ...

The repeated next hops expose the loop. This test is more valuable than
repeatedly pinging only known working hosts.

Step 5: Inspect BR1 Routing

 BR1# show ip route 10.50.99.99
 BR1# show ip route 0.0.0.0
 BR1# show ip route 10.50.0.0

Expected evidence:

    No specific route exists for 10.50.99.99.

    The default route points to 10.12.12.1 on HUB1.

    No EIGRP summary route to Null0 exists for 10.50.0.0/16.

Step 6: Inspect the Branch Stub and Summary Configuration

 BR1# show running-config | section router eigrp
 BR1# show eigrp address-family ipv4 interfaces detail
 BR1# show ip eigrp topology

Look for:

 eigrp stub receive-only

 No summary-address 10.50.0.0 255.255.0.0 under Gi0/0

Step 7: State the Root-Cause Hypothesis Before Repair

 Look For

 Missing stub keyword(s) on BR1's EIGRP configuration

 Hub still sending queries for routes BR1 cannot provide

Step 8: Remove the Static Workaround First

 HUB1# configure terminal
 HUB1(config)# no ip route 10.50.0.0 255.255.0.0 10.12.12.2
 HUB1(config)# end

Removing the static route creates a brief loss of branch reachability until
the dynamic summary is restored. In production, prepare the complete
change and execute it within an approved window. In the lab, removing the
route prevents the static administrative distance from hiding the repaired
EIGRP route.

Step 9: Restore the Correct Stub Policy

 BR1# configure terminal
 BR1(config)# router eigrp ENTERPRISE
 BR1(config-router)# address-family ipv4 unicast
 autonomous-system 100
 BR1(config-router-af)# eigrp stub connected summary

The explicit keywords document the intended advertisement policy. The
default `eigrp stub` command also advertises connected and summary
information, but explicit syntax is easier to audit in an interview.

Step 10: Restore the EIGRP Summary on the Correct Interface
