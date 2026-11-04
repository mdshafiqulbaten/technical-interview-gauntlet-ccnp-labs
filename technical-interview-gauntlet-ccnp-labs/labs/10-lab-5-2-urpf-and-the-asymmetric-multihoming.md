# Lab 5.2: uRPF and the Asymmetric Multihoming

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 10172-10571). Review against the printed book before classroom use.*

Trap
Suggested interview time limit: 35 minutes

1. THE TOPOLOGY & OBJECTIVE
EDGE is connected to two providers. BGP prefers ISP-A for the route to
the remote cloud source prefix. External policy, however, causes the cloud
client to enter the enterprise through ISP-B. The application design

intentionally permits asymmetric routing. The packet must be accepted on
ISP-B, forwarded to the inside application, and returned through ISP-A.

Lab 5.2 topology

                                   203.0.113.0/24
                                        CLOUD-CLIENT
                                       203.0.113.100
                                        /           \
                                       /             \
                              ISP-A AS65001       ISP-B AS65002
                            192.0.2.1/30          198.51.100.1/30
                                   |                    |
                            Gi0/0 |                     | Gi0/1
                                 192.0.2.2 EDGE 198.51.100.2
                                        |
                                 Gi0/2 10.100.0.1/24
                                        |
                                 APP-SRV 10.100.0.10

                                          CLOUD-CLIENT

                                            203.0.113.100

               ISP-A                                                           ISP-B
              AS65001                                                          AS65002

                        Gi0/0                                         Gi0/1
                           192.0.2.0/30                      198.51.100.0/30

                                              EDGE

                                                    Gi0/2

                                            APP-SRV

    Figure: Lab 5.2: uRPF and the Asymmetric Multihoming Trap — Physical Topology

 Device            Interface         Address                Role

 EDGE              Gi0/0             192.0.2.2/30           eBGP to ISP-A (AS65001)

 EDGE              Gi0/1             198.51.100.2/30        eBGP to ISP-B (AS65002)

 EDGE              Gi0/2             10.100.0.1/24          Application LAN

 CLOUD-            via ISP-          203.0.113.100          External source entering through
 CLIENT            B                                        ISP-B

2. THE INITIAL GOLDEN CONFIG

EDGE Golden Configuration

 enable
 configure terminal
 hostname EDGE
 no ip domain-lookup
 !
 interface GigabitEthernet0/0
  description EBGP-TO-ISP-A
  ip address 192.0.2.2 255.255.255.252
  ip verify unicast source reachable-via any
  no shutdown
 !
 interface GigabitEthernet0/1
  description EBGP-TO-ISP-B
  ip address 198.51.100.2 255.255.255.252
  ip verify unicast source reachable-via any
  no shutdown
 !
 interface GigabitEthernet0/2
  description APPLICATION-LAN
  ip address 10.100.0.1 255.255.255.0
  no shutdown
 !
 router bgp 65000
  bgp log-neighbor-changes
  neighbor 192.0.2.1 remote-as 65001
  neighbor 198.51.100.1 remote-as 65002
  address-family ipv4
   network 10.100.0.0 mask 255.255.255.0
   neighbor 192.0.2.1 activate
   neighbor 198.51.100.1 activate
   neighbor 192.0.2.1 route-map PREFER-A-IN in
  exit-address-family
 !
 route-map PREFER-A-IN permit 10
  set local-preference 200
 !
 end
 write memory

ISP-A Golden Configuration

 hostname ISP-A
 !
 interface GigabitEthernet0/0
  ip address 192.0.2.1 255.255.255.252
  no shutdown
 !
 interface Loopback100
  ip address 203.0.113.1 255.255.255.0
 !
 router bgp 65001
  neighbor 192.0.2.2 remote-as 65000
  address-family ipv4
   network 203.0.113.0 mask 255.255.255.0
   neighbor 192.0.2.2 activate
  exit-address-family

ISP-B Golden Configuration

 hostname ISP-B
 !
 interface GigabitEthernet0/0
  ip address 198.51.100.1 255.255.255.252
  no shutdown
 !
 interface Loopback100
  ip address 203.0.113.2 255.255.255.0
 !
 router bgp 65002
  neighbor 198.51.100.2 remote-as 65000
  address-family ipv4
   network 203.0.113.0 mask 255.255.255.0
   neighbor 198.51.100.2 activate
  exit-address-family

In a real Internet design, two independent providers would not normally
originate the same remote customer prefix in this simplified manner. The
lab uses the duplicate advertisement only to create a deterministic FIB
choice and demonstrate the uRPF decision. The source host can be
emulated behind ISP-B while EDGE prefers ISP-A for the reverse route.

APP-SRV Configuration

 hostname APP-SRV
 interface GigabitEthernet0/0
  ip address 10.100.0.10 255.255.255.0
  no shutdown
 ip route 0.0.0.0 0.0.0.0 10.100.0.1

CLOUD-CLIENT Test Method
Use a Linux or IOS test node behind ISP-B with source address
203.0.113.100. Ensure the request reaches EDGE through Gi0/1. The
return path from EDGE to 203.0.113.0/24 must resolve through Gi0/0
because of Local Preference 200 on the ISP-A route.

Golden-state verification

 EDGE# show ip bgp 203.0.113.0
 EDGE# show ip route 203.0.113.100
 EDGE# show ip cef 203.0.113.100 detail
 EDGE# show ip interface GigabitEthernet0/1 | include
 verify|unicast
 CLOUD-CLIENT$ ping 10.100.0.10

The cloud request should be accepted on ISP-B because loose uRPF
requires the source to be reachable through any interface. EDGE then
returns traffic through ISP-A according to its FIB.

3. THE SABOTAGE SCRIPT
Replace loose uRPF with strict uRPF on ISP-B

 EDGE# configure terminal
 EDGE(config)# interface GigabitEthernet0/1
 EDGE(config-if)# no ip verify unicast source reachable-via
 any
 EDGE(config-if)# ip verify unicast source reachable-via rx
 EDGE(config-if)# end

No routing policy is changed. BGP remains established and the route to
203.0.113.0/24 remains present through ISP-A. The legitimate packet now
fails the strict reverse-path test when it arrives through ISP-B.

4. THE VERIFICATION WORKFLOW

Step 1: Establish Whether the Application Server Is Locally
Reachable

 EDGE# ping 10.100.0.10 source 10.100.0.1
 EDGE# show arp 10.100.0.10
 EDGE# show ip route 10.100.0.10

Local application-LAN reachability proves the inside interface and server
gateway path. It does not test the provider ingress security policy.

Step 2: Verify Both BGP Sessions

 EDGE# show ip bgp summary
 EDGE# show ip bgp neighbors 192.0.2.1
 EDGE# show ip bgp neighbors 198.51.100.1

Both sessions should be Established. The failure is therefore not caused by
a missing provider adjacency.

Step 3: Inspect the Route to the Packet Source

 EDGE# show ip bgp 203.0.113.0
 EDGE# show ip route 203.0.113.100
 EDGE# show ip cef 203.0.113.100 detail

The best route and CEF adjacency point through Gi0/0 toward ISP-A
because the inbound route map set Local Preference 200 on that path.

Step 4: Confirm the Actual Ingress Interface

Use the lab wiring, provider counters, packet capture, or interface counters
to prove the cloud packet enters Gi0/1 from ISP-B.

 EDGE# show interfaces GigabitEthernet0/1 | include packets
 input|rate
 EDGE# show access-lists
 EDGE# show logging

The key comparison is now explicit: packet ingress is Gi0/1, but the
reverse FIB path to the source is Gi0/0.

Step 5: Inspect uRPF Mode

 EDGE# show ip interface GigabitEthernet0/1
 EDGE# show running-config interface GigabitEthernet0/1

Look for strict source validation, typically displayed as source reachable
via RX or the corresponding configured command.

Step 6: Inspect Platform Drop Evidence

 EDGE# show platform hardware qfp active statistics drop |
 include RPF|Unicast
 EDGE# show ip traffic | include RPF
 EDGE# show cef interface GigabitEthernet0/1

Use whichever command the platform supports. A virtual image may
expose limited drop statistics. The FIB and interface comparison is still
sufficient to explain the strict-mode decision.

Step 7: Form the Hypothesis

 Hypothesis                 Supporting Evidence            Test That Would Disprove
                                                           It

 Strict uRPF on Gi0/1 is    BGP is healthy, the route      If loose mode is already
 rejecting a legitimately   exists, but traffic entering   configured on Gi0/1, this
 asymmetric flow            ISP-B is dropped               hypothesis is wrong

Step 8: Consider the Valid Repair Options

 Option                     Effect                        Recommended Here?

 Restore loose uRPF on      Keeps source validation       Yes — smallest change
 Gi0/1                      while permitting the          consistent with the business
                            approved asymmetric path      requirement

 Disable uRPF entirely on   Removes source validation     No — weakens security
 Gi0/1                      completely                    beyond what the
                                                          requirement calls for

 Add a static route to      Could make strict mode        No — changes routing
 force symmetry             work, but overrides BGP's     design to work around a
                            real path selection           security setting

 Change BGP local           Could restore symmetry, but   No — not requested and
 preference to force        may conflict with provider    has broader routing impact
 symmetric routing          traffic-engineering intent

 Use a platform-specific    Can work on some platforms    No — loose mode already
 uRPF exception/ACL         but adds complexity for a     solves this cleanly
 override                   single legitimate flow

The business requirement explicitly permits asymmetry. Restoring loose
mode is therefore the smallest correct change for this lab.

Step 9: Restore Loose uRPF

 EDGE# configure terminal
 EDGE(config)# interface GigabitEthernet0/1
 EDGE(config-if)# no ip verify unicast source reachable-via
 rx
 EDGE(config-if)# ip verify unicast source reachable-via
 any
 EDGE(config-if)# end

Step 10: Verify the Source Still Exists in the FIB

 EDGE# show ip route 203.0.113.100
 EDGE# show ip cef 203.0.113.100 detail
 EDGE# show ip interface GigabitEthernet0/1

Loose mode is not “permit anything.” The source prefix must still be
reachable through some FIB path, subject to platform options such as
allow-default.

Step 11: Test the Asymmetric Flow

 CLOUD-CLIENT$ ping 10.100.0.10
 CLOUD-CLIENT$ traceroute 10.100.0.10
 EDGE# show interfaces GigabitEthernet0/1 | include packets
 input
 EDGE# show interfaces GigabitEthernet0/0 | include packets
 output

The request enters Gi0/1 and the return traffic exits Gi0/0. That asymmetry
is expected and approved in this scenario.

Step 12: Test a Clearly Invalid Source

Where the authorized lab platform permits source-address testing, send a
packet from an unrouted documentation prefix and verify that loose uRPF
