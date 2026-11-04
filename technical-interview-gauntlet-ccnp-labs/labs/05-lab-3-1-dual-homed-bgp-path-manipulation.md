# Lab 3.1: Dual-Homed BGP Path Manipulation

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 5508-5907). Review against the printed book before classroom use.*

Recommended time limit: 40 minutes

 Objective

 Diagnose a BGP policy-direction error affecting outbound and inbound traffic
 independently

1. THE TOPOLOGY & OBJECTIVE
A company in AS 65000 advertises 198.51.100.0/24 through two service
providers. ISP1 is the contracted primary path. ISP2 is a backup path with

a lower-cost circuit but limited capacity. The company also receives a
simulated Internet prefix, 203.0.113.0/24, through both providers.

Lab topology

                                                     203.0.113.0/24
                                                        INTERNET
                                                         AS 65300
                                                    Gi0/0        Gi0/1
                                              10.0.31.2        10.0.30.2
                                                    |                |
                                              10.0.31.1        10.0.30.1
                                                ISP1               ISP2
                                              AS 65100          AS 65200
                                              Gi0/0               Gi0/0
                                            10.0.11.1          10.0.12.1
                                                    \              /
                                              10.0.11.2        10.0.12.2
                                                        CUST-EDGE
                                                         AS 65000
                                                    198.51.100.0/24

 Business intent:
 Outbound traffic to 203.0.113.0/24 must prefer ISP1.
 Inbound traffic to 198.51.100.0/24 should prefer ISP1.
 ISP2 must remain a valid automatic backup.

                                               INTERNET
                                                  AS 65300

                                        Gi0/0 10.0.31.2      Gi0/1 10.0.30.2

                          Gi0/1 10.0.31.1                                 Gi0/1 10.0.30.1

                 ISP1                                                                ISP2
               AS 65100                                                            AS 65200

             Gi0/0 10.0.11.1                                                     Gi0/0 10.0.12.1

                                    Gi0/0 10.0.11.2            Gi0/1 10.0.12.2

                                              CUST-EDGE
                                                  AS 65000

       Figure: Lab 3.1 Dual-Homed BGP Path Manipulation — Physical Topology

 Requirement

 CUST-EDGE maintains eBGP sessions to both ISP1 and ISP2

 ISP1 is the intended primary outbound path

 ISP2 is the intended backup outbound path

 Inbound traffic from ISP2 should prefer ISP1 via AS-path prepending

 Route maps FROM-ISP1 and PREPEND-TO-ISP2 exist and are syntactically correct

 Local Preference is set on routes learned from ISP1

 Failover to ISP2 succeeds if ISP1 fails

 Primary path is restored automatically when ISP1 recovers

The provider lab intentionally makes the ISP1 path to the Internet prefix
one AS hop longer when sent to the customer. Correct Local Preference
must therefore override the shorter ISP2 path. On the Internet router, ISP2
has the lower BGP router ID, so equal unmodified customer paths favor
ISP2. Correct AS-path prepending on the ISP2 advertisement must make
ISP1 the preferred inbound path.

2. THE INITIAL GOLDEN CONFIG
CUST-EDGE golden configuration

enable
configure terminal
hostname CUST-EDGE
no ip domain-lookup
!
interface Loopback100
 ip address 198.51.100.1 255.255.255.0
!
interface GigabitEthernet0/0
 description EBGP-TO-ISP1-PRIMARY
 ip address 10.0.11.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description EBGP-TO-ISP2-BACKUP
 ip address 10.0.12.2 255.255.255.252
 no shutdown
!
ip prefix-list ALL-RECEIVED seq 10 permit 0.0.0.0/0 le 32
ip prefix-list OUR-PREFIX seq 10 permit 198.51.100.0/24
!
route-map FROM-ISP1 permit 10
 match ip address prefix-list ALL-RECEIVED
 set local-preference 200
!
route-map PREPEND-TO-ISP2 permit 10
 match ip address prefix-list OUR-PREFIX
 set as-path prepend 65000 65000 65000
route-map PREPEND-TO-ISP2 permit 100
!
router bgp 65000
 bgp router-id 10.255.0.10
 bgp log-neighbor-changes
 neighbor 10.0.11.1 remote-as 65100
 neighbor 10.0.11.1 description ISP1-PRIMARY
 neighbor 10.0.12.1 remote-as 65200
 neighbor 10.0.12.1 description ISP2-BACKUP
 address-family ipv4
  network 198.51.100.0 mask 255.255.255.0
  neighbor 10.0.11.1 activate
  neighbor 10.0.12.1 activate
  neighbor 10.0.11.1 route-map FROM-ISP1 in
  neighbor 10.0.12.1 route-map PREPEND-TO-ISP2 out
 exit-address-family
end
write memory

ISP1 golden configuration

 enable
 configure terminal
 hostname ISP1
 no ip domain-lookup
 !
 interface GigabitEthernet0/0
  ip address 10.0.11.1 255.255.255.252
  no shutdown
 interface GigabitEthernet0/1
  ip address 10.0.31.1 255.255.255.252
  no shutdown
 !
 ip prefix-list INTERNET-PREFIX seq 10 permit
 203.0.113.0/24
 route-map LONGER-TO-CUSTOMER permit 10
  match ip address prefix-list INTERNET-PREFIX
  set as-path prepend 65100
 route-map LONGER-TO-CUSTOMER permit 100
 !
 router bgp 65100
  bgp router-id 3.3.3.3
  neighbor 10.0.11.2 remote-as 65000
  neighbor 10.0.31.2 remote-as 65300
  address-family ipv4
   neighbor 10.0.11.2 activate
   neighbor 10.0.31.2 activate
   neighbor 10.0.11.2 route-map LONGER-TO-CUSTOMER out
  exit-address-family
 end
 write memory

ISP2 golden configuration

 enable
 configure terminal
 hostname ISP2
 no ip domain-lookup
 !
 interface GigabitEthernet0/0
  ip address 10.0.12.1 255.255.255.252
  no shutdown
 interface GigabitEthernet0/1
  ip address 10.0.30.1 255.255.255.252
  no shutdown
 !
 router bgp 65200
  bgp router-id 2.2.2.2
  neighbor 10.0.12.2 remote-as 65000
  neighbor 10.0.30.2 remote-as 65300
  address-family ipv4
   neighbor 10.0.12.2 activate
   neighbor 10.0.30.2 activate
  exit-address-family
 end
 write memory

INTERNET golden configuration

 enable
 configure terminal
 hostname INTERNET
 no ip domain-lookup
 !
 interface Loopback0
  ip address 203.0.113.1 255.255.255.0
 !
 interface GigabitEthernet0/0
  ip address 10.0.31.2 255.255.255.252
  no shutdown
 interface GigabitEthernet0/1
  ip address 10.0.30.2 255.255.255.252
  no shutdown
 !
 router bgp 65300
  bgp router-id 4.4.4.4
  neighbor 10.0.31.1 remote-as 65100
  neighbor 10.0.30.1 remote-as 65200
  address-family ipv4
   network 203.0.113.0 mask 255.255.255.0
   neighbor 10.0.31.1 activate
   neighbor 10.0.30.1 activate
  exit-address-family
 end
 write memory

Golden-state verification on CUST-EDGE:

 show ip bgp summary
 show ip bgp 203.0.113.0/24
 show ip route 203.0.113.0
 show ip cef 203.0.113.1

The best path should be through ISP1 because Local Preference 200 is
evaluated before the shorter ISP2 AS path.

Golden-state verification on INTERNET:

 show ip bgp 198.51.100.0/24
 show ip route 198.51.100.0
 traceroute 198.51.100.1 source 203.0.113.1

The path through ISP1 should be best because the customer advertisement
toward ISP2 contains additional copies of AS 65000.

3. THE SABOTAGE SCRIPT
The interviewer reverses the policy direction on both customer neighbors.
The route maps remain present and syntactically valid, which makes the
failure easy to overlook.

Run on CUST-EDGE

 configure terminal
 router bgp 65000
  address-family ipv4
   no neighbor 10.0.11.1 route-map FROM-ISP1 in
   no neighbor 10.0.12.1 route-map PREPEND-TO-ISP2 out
   neighbor 10.0.11.1 route-map FROM-ISP1 out
   neighbor 10.0.12.1 route-map PREPEND-TO-ISP2 in
  exit-address-family
 end
 !
 clear ip bgp 10.0.11.1 soft out
 clear ip bgp 10.0.12.1 soft in

Resulting behavior:

    Both BGP sessions remain Established.

    CUST-EDGE chooses ISP2 for 203.0.113.0/24 because its path has
    fewer AS entries.

    The Local Preference route map is attached outbound to ISP1 and
    therefore does not set Local Preference on received Internet routes.

    The prepend route map is attached inbound from ISP2 and does not
    alter the customer prefix advertisement.

    INTERNET receives equal-length customer paths and prefers ISP2
    because of the deterministic tie-break conditions in the lab.

    Failover still exists, so the network appears functional while violating
    cost and capacity policy.

4. THE VERIFICATION WORKFLOW

Step 1: Confirm the Business Requirement

State the required behavior before reading attributes: ISP1 is primary for
outbound and inbound traffic; ISP2 is backup. Without that statement, a
technically valid ISP2 path could be mistaken for success.

Step 2: Verify Both Sessions

 CUST-EDGE# show ip bgp summary

Look for both neighbors in Established state with nonzero prefix counts.
This eliminates a basic session outage but does not validate policy.

Step 3: Inspect the Outbound Destination

 CUST-EDGE# show ip bgp 203.0.113.0/24

Representative path comparison:

 Paths: (2 available, best #2)
   65100 65100 65300
     10.0.11.1 from 10.0.11.1
   65200 65300
     10.0.12.1 from 10.0.12.1
     best

The ISP2 path wins because the Local Preference values are equal and the
AS path is shorter. In the golden state, the ISP1 path should display Local
Preference 200 and win before AS-path length is considered.

Step 4: Confirm the Installed Exit

 CUST-EDGE# show ip route 203.0.113.0
 CUST-EDGE# show ip cef 203.0.113.1
 CUST-EDGE# traceroute 203.0.113.1 source 198.51.100.1

The route and trace should point toward 10.0.12.1, proving the actual
outbound policy violation.

Step 5: Read the Neighbor Policy Direction

 CUST-EDGE# show ip bgp neighbors 10.0.11.1 | include
 route-map|Route map
 CUST-EDGE# show ip bgp neighbors 10.0.12.1 | include
 route-map|Route map
 CUST-EDGE# show running-config | section router bgp

Expected evidence:
