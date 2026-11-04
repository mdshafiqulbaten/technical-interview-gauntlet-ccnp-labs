# Lab 3.2: Route Reflectors and the Invisible Black

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 6142-6541). Review against the printed book before classroom use.*

Hole
Recommended time limit: 40 minutes

 Objective

 Repair a route-reflection configuration gap, then expose and fix a separate next-hop data-
 plane black hole

1. THE TOPOLOGY & OBJECTIVE
AS 65000 uses RR1 to reflect IPv4 routes between two provider-edge
routers. PE1 learns 203.0.113.1/32 from an external partner in AS 65100.
PE2 must reach that prefix through PE1. OSPF provides reachability
among the internal loopbacks but does not carry the external 172.16.1.0/30
link.

Lab topology

 External Partner
     EXT1 AS 65100
 Loopback0 203.0.113.1/32
          |
     172.16.1.1/30
          |
     172.16.1.2/30
          PE1
  Lo0 10.255.0.1/32
          |
      10.0.12.0/30
          |
          RR1
  Lo0 10.255.0.2/32
          |
      10.0.23.0/30
          |
          PE2
  Lo0 10.255.0.3/32

All internal routers: AS 65000
RR cluster ID: 10.255.0.2
PE1 and PE2: route-reflector clients
Internal loopbacks and transit links: OSPF area 0

                                    EXT1
                                   AS 65100

                                         Gi0/0 172.16.1.1/30

                                         Gi0/0 172.16.1.2/30

                                     PE1
                                 Lo0 10.255.0.1

                                         Gi0/1 10.0.12.1/30

                                         Gi0/0 10.0.12.2/30

                                     RR1
                                 Lo0 10.255.0.2

                                         Gi0/1 10.0.23.2/30

                                         Gi0/0 10.0.23.3/30

                                     PE2
                                 Lo0 10.255.0.3

     Figure: Lab 3.2 Route Reflectors and the Invisible Black Hole — Physical Topology

 Requirement

 RR1 is configured as the route reflector for PE1 and PE2

 PE1 and PE2 are configured as route-reflector-client under RR1

 PE1 learns an eBGP route from EXT1 and advertises it into iBGP

 RR1 reflects the route to PE2

 PE2's installed next hop must be reachable via the IGP

 End-to-end reachability confirmed from PE2 toward the EXT1-advertised prefix

Final working state:

    PE1 and PE2 establish iBGP only with RR1; no PE1-to-PE2 full-mesh
    session is required.

    RR1 identifies PE1 and PE2 as route-reflector clients.

    PE2 learns 203.0.113.1/32 through BGP.

   The BGP next hop on PE2 is PE1 loopback 10.255.0.1 because PE1
   uses next-hop-self toward RR1.

   OSPF resolves 10.255.0.1 through the internal network.

   PE2 can ping and trace to 203.0.113.1 from 10.255.0.3.

   No misleading static discard summary for 172.16.0.0/16 remains on
   PE2.

2. THE INITIAL GOLDEN CONFIG
EXT1 golden configuration

 enable
 configure terminal
 hostname EXT1
 no ip domain-lookup
 !
 interface Loopback0
  ip address 203.0.113.1 255.255.255.255
 !
 interface GigabitEthernet0/0
  ip address 172.16.1.1 255.255.255.252
  no shutdown
 !
 router bgp 65100
  bgp router-id 203.0.113.1
  neighbor 172.16.1.2 remote-as 65000
  address-family ipv4
   network 203.0.113.1 mask 255.255.255.255
   neighbor 172.16.1.2 activate
  exit-address-family
 end
 write memory

PE1 golden configuration

 enable
 configure terminal
 hostname PE1
 no ip domain-lookup
 !
 interface Loopback0
  ip address 10.255.0.1 255.255.255.255
  ip ospf 100 area 0
 !
 interface GigabitEthernet0/0
  description EBGP-TO-EXT1
  ip address 172.16.1.2 255.255.255.252
  no shutdown
 !
 interface GigabitEthernet0/1
  description OSPF-TO-RR1
  ip address 10.0.12.1 255.255.255.252
  ip ospf 100 area 0
  ip ospf network point-to-point
  no shutdown
 !
 router ospf 100
  router-id 10.255.0.1
  passive-interface Loopback0
 !
 router bgp 65000
  bgp router-id 10.255.0.1
  bgp log-neighbor-changes
  neighbor 172.16.1.1 remote-as 65100
  neighbor 10.255.0.2 remote-as 65000
  neighbor 10.255.0.2 update-source Loopback0
  address-family ipv4
   neighbor 172.16.1.1 activate
   neighbor 10.255.0.2 activate
   neighbor 10.255.0.2 next-hop-self
  exit-address-family
 end
 write memory

RR1 golden configuration

 enable
 configure terminal
 hostname RR1
 no ip domain-lookup
 !
 interface Loopback0
  ip address 10.255.0.2 255.255.255.255
  ip ospf 100 area 0
 !
 interface GigabitEthernet0/0
  description OSPF-TO-PE1
  ip address 10.0.12.2 255.255.255.252
  ip ospf 100 area 0
  ip ospf network point-to-point
  no shutdown
 !
 interface GigabitEthernet0/1
  description OSPF-TO-PE2
  ip address 10.0.23.2 255.255.255.252
  ip ospf 100 area 0
  ip ospf network point-to-point
  no shutdown
 !
 router ospf 100
  router-id 10.255.0.2
  passive-interface Loopback0
 !
 router bgp 65000
  bgp router-id 10.255.0.2
  bgp cluster-id 10.255.0.2
  neighbor 10.255.0.1 remote-as 65000
  neighbor 10.255.0.1 update-source Loopback0
  neighbor 10.255.0.3 remote-as 65000
  neighbor 10.255.0.3 update-source Loopback0
  address-family ipv4
   neighbor 10.255.0.1 activate
   neighbor 10.255.0.3 activate
   neighbor 10.255.0.1 route-reflector-client
   neighbor 10.255.0.3 route-reflector-client
  exit-address-family
 end
 write memory

PE2 golden configuration

 enable
 configure terminal
 hostname PE2
 no ip domain-lookup
 !
 interface Loopback0
  ip address 10.255.0.3 255.255.255.255
  ip ospf 100 area 0
 !
 interface GigabitEthernet0/0
  description OSPF-TO-RR1
  ip address 10.0.23.3 255.255.255.252
  ip ospf 100 area 0
  ip ospf network point-to-point
  no shutdown
 !
 router ospf 100
  router-id 10.255.0.3
  passive-interface Loopback0
 !
 router bgp 65000
  bgp router-id 10.255.0.3
  bgp log-neighbor-changes
  neighbor 10.255.0.2 remote-as 65000
  neighbor 10.255.0.2 update-source Loopback0
  address-family ipv4
   neighbor 10.255.0.2 activate
  exit-address-family
 end
 write memory

Golden verification:

 RR1# show ip bgp summary
 RR1# show ip bgp 203.0.113.1/32
 PE2# show ip bgp 203.0.113.1/32
 PE2# show ip route 203.0.113.1
 PE2# show ip route 10.255.0.1
 PE2# ping 203.0.113.1 source 10.255.0.3

On PE2, the BGP path should use next hop 10.255.0.1. Prefix-specific
output may display the route reflector originator and cluster-list attributes.

3. THE SABOTAGE SCRIPT
Three changes are introduced. The first suppresses route reflection. The
second preserves the external next hop. The third makes that next hop
appear reachable through a discard summary, creating a hidden forwarding
black hole after route reflection is restored.

Run on RR1

 configure terminal
 router bgp 65000
  address-family ipv4
   no neighbor 10.255.0.1 route-reflector-client
   no neighbor 10.255.0.3 route-reflector-client
  exit-address-family
 end

Run on PE1

 configure terminal
 router bgp 65000
  address-family ipv4
   no neighbor 10.255.0.2 next-hop-self
  exit-address-family
 end
 clear ip bgp 10.255.0.2 soft out

Run on PE2

 configure terminal
 ip route 172.16.0.0 255.255.0.0 Null0
 end

Initial symptom: PE2 has an Established session to RR1 but does not learn
203.0.113.1/32. After route reflection is repaired, PE2 installs the route
with next hop 172.16.1.1. The static /16 makes that next hop recursively
reachable through Null0, so the BGP route appears valid and enters the
routing table while traffic is discarded.

4. THE VERIFICATION WORKFLOW

Step 1: Verify the Internal Transport

 PE2# show ip ospf neighbor
 PE2# show ip route 10.255.0.2
 PE2# ping 10.255.0.2 source 10.255.0.3

OSPF and loopback reachability should be healthy. This confirms that the
iBGP transport to RR1 is available.

Step 2: Verify the iBGP Session

 PE2# show ip bgp summary
 RR1# show ip bgp summary

PE2 and RR1 should be Established. The missing external route is
therefore an advertisement or policy problem, not a failed TCP session.

Step 3: Locate the Route

 PE1# show ip bgp 203.0.113.1/32
 RR1# show ip bgp 203.0.113.1/32
 PE2# show ip bgp 203.0.113.1/32

Expected evidence: PE1 learns the route from EXT1. RR1 learns it from
PE1. PE2 does not have it.

Step 4: Explain the iBGP Rule

RR1 learned the route through iBGP from PE1. An ordinary iBGP speaker
cannot advertise that route to another ordinary iBGP peer. RR1 must
identify the relevant peers as route-reflector clients, or a full mesh would
be required.

Step 5: Inspect Route-Reflector Configuration
