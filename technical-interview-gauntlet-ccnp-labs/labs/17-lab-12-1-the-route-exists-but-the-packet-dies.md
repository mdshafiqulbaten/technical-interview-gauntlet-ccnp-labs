# Lab 12.1: The Route Exists but the Packet Dies

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 23639-23967). Review against the printed book before classroom use.*

Time Limit: 20 Minutes

1. THE TOPOLOGY & OBJECTIVE
A headquarters router has a static route to a server network through a
directly connected transit next hop. The route appears in the routing table,
but no packet reaches the server router. The candidate must prove the exact
failure point without deleting the route or replacing it with an interface-
only static route.

 HQ-RTR Loopback10 10.10.10.1/24
         Gi0/0 192.0.2.1/24
                |
                | Ethernet transit 192.0.2.0/24
                |
         Gi0/0 192.0.2.2/24
 SERVER-RTR Gi0/1 10.30.30.1/24
 Server target: 10.30.30.10/24

       Lo10 10.10.10.1/24
                            Gi0/0 192.0.2.1/24

          HQ-RTR                                                      SERVER-RTR
                                Ethernet transit 192.0.2.0/24

                                                 Gi0/0 192.0.2.2/24           Gi0/1

                                                                       Server LAN 10.30.30.0/24

            Figure: Static Route Next-Hop Resolution Lab — Physical Topology

Business intent:

    HQ-RTR must reach 10.30.30.0/24 through next hop 192.0.2.2.

    SERVER-RTR must return 10.10.10.0/24 through 192.0.2.1.

    The immediate Ethernet next-hop adjacency must resolve normally.

    The final state must use the original static-route design.

2. THE INITIAL GOLDEN CONFIG
HQ-RTR:

 enable
 configure terminal
 hostname HQ-RTR
 no ip domain-lookup
 !
 interface Loopback10
  ip address 10.10.10.1 255.255.255.0
 !
 interface GigabitEthernet0/0
  description TRANSIT-TO-SERVER-RTR
  ip address 192.0.2.1 255.255.255.0
  no shutdown
 !
 ip route 10.30.30.0 255.255.255.0 192.0.2.2
 end

SERVER-RTR:

 enable
 configure terminal
 hostname SERVER-RTR
 no ip domain-lookup
 !
 interface GigabitEthernet0/0
  description TRANSIT-TO-HQ-RTR
  ip address 192.0.2.2 255.255.255.0
  no shutdown
 !
 interface GigabitEthernet0/1
  description SERVER-LAN
  ip address 10.30.30.1 255.255.255.0
  no shutdown
 !
 ip route 10.10.10.0 255.255.255.0 192.0.2.1
 end

Golden verification:

 HQ-RTR# show ip route 10.30.30.0
 HQ-RTR# show ip arp 192.0.2.2
 HQ-RTR# show ip cef exact-route 10.10.10.1 10.30.30.10
 HQ-RTR# ping 10.30.30.1 source Loopback10

3. THE SABOTAGE SCRIPT
Run on SERVER-RTR:

 configure terminal
 interface GigabitEthernet0/0
  no ip address 192.0.2.2 255.255.255.0
  ip address 192.0.2.22 255.255.255.0
 end

The physical link remains up. HQ-RTR keeps its static route because
192.0.2.2 is still considered reachable through the connected 192.0.2.0/24
network. However, no device answers ARP for 192.0.2.2.

4. THE VERIFICATION WORKFLOW
Step 1: Confirm the symptom with the application source.

 HQ-RTR# ping 10.30.30.1 source Loopback10
 .....
 Success rate is 0 percent

Step 2: Confirm that the route really is installed.

 HQ-RTR# show ip route 10.30.30.0
 Routing entry for 10.30.30.0/24
   Known via "static", distance 1, metric 0
   Routing Descriptor Blocks:
   * 192.0.2.2

The route is present, so deleting and recreating it would be an evidence-
free change. Continue into CEF and adjacency resolution.

Step 3: Ask CEF what it will do for the actual source and destination.

 HQ-RTR# show ip cef exact-route 10.10.10.1 10.30.30.10
 10.10.10.1 -> 10.30.30.10 : GigabitEthernet0/0
 (incomplete)

The exact wording varies, but the critical clue is an unresolved or
incomplete adjacency toward the intended egress interface.

Step 4: Inspect ARP and the immediate next hop.

 HQ-RTR# show ip arp 192.0.2.2
 Protocol Address       Age (min) Hardware Addr Type
 Interface
 Internet 192.0.2.2            0 Incomplete    ARPA
 Gi0/0

 HQ-RTR# ping 192.0.2.2
 .....

Step 5: Discover the connected neighbor without assuming its configured
address.

 HQ-RTR# show cdp neighbors detail
 Device ID: SERVER-RTR
   IP address: 192.0.2.22
   Interface: GigabitEthernet0/0

Step 6: Verify the remote interface configuration.

 SERVER-RTR# show ip interface brief
 GigabitEthernet0/0 192.0.2.22 YES manual            up   up

 SERVER-RTR# show running-config interface
 GigabitEthernet0/0

Step 7: State the root cause before changing it.

 Symptom                                 Recommended First Commands

 Ping works, application fails           show ip access-lists, show policy-map control-plane

 Route is present, traffic still fails   show ip cef exact-route, show ip arp, show
                                         adjacency detail

 One VLAN fails, others on the           show interfaces trunk, show vlan brief, show
 same trunk work                         spanning-tree vlan

 Routing neighbor will not form          show ip ospf/eigrp interface, show ip ospf/eigrp
                                         neighbors; compare timers/area/AS

 Static route always wins over a         show ip route detail, compare administrative
 dynamic route                           distance

 OSPF stuck below Full                   show ip ospf neighbor; compare MTU, area type,
                                         and authentication

 BGP prefix missing from a peer          show ip bgp neighbors advertised-routes/routes,
                                         show route-map

 Tunnel up but application               ping with df-bit at increasing size, show interfaces |
 unreliable                              include MTU

 Voice quality degrades under            show policy-map interface, show interfaces | include
 load                                    drop

 Automation request fails silently       show netconf-yang sessions, show logging | include
                                         NETCONF

 EtherChannel forms but                  show etherchannel detail, show lacp neighbor; check
 intermittently drops                    load-balancing mismatch

 Security control appears to block       show ip access-lists counters, show policy-map
 legitimate traffic                      control-plane

Step 8: Restore the intended address.

 SERVER-RTR# configure terminal
 SERVER-RTR(config)# interface GigabitEthernet0/0
 SERVER-RTR(config-if)# no ip address 192.0.2.22
 255.255.255.0
 SERVER-RTR(config-if)# ip address 192.0.2.2 255.255.255.0
 SERVER-RTR(config-if)# end

Step 9: Verify route, adjacency, and end-to-end traffic.

 HQ-RTR# show ip arp 192.0.2.2
 HQ-RTR# show ip cef exact-route 10.10.10.1 10.30.30.10
 HQ-RTR# ping 192.0.2.2
 HQ-RTR# ping 10.30.30.1 source Loopback10
 HQ-RTR# traceroute 10.30.30.1 source Loopback10 numeric

The final evidence must show a complete ARP entry, a resolved CEF
adjacency, and a successful sourced test. The original static route remains
unchanged.

5. THE INTERVIEW NARRATION SCRIPT
Candidate: “The route is installed, so I do not yet have evidence that the
static route itself is wrong. I will follow the forwarding decision from the
RIB into CEF and Layer 2 adjacency resolution.”

Interviewer Reaction Note: The candidate does not confuse route
presence with packet delivery.

Candidate: “The exact-route lookup selects GigabitEthernet0/0 but
reports an incomplete adjacency. I will check ARP for the configured next
hop and test that immediate neighbor directly.”

Interviewer Reaction Note: The investigation has narrowed from remote
routing to local next-hop resolution.

Candidate: “ARP for 192.0.2.2 is incomplete. CDP identifies SERVER-
RTR on the link with management address 192.0.2.22, so I will confirm
the transit-interface address on the remote router.”

Interviewer Reaction Note: The candidate uses discovery as evidence but
still verifies the actual interface configuration.

Candidate: “SERVER-RTR is configured with 192.0.2.22 instead of
192.0.2.2. The static route remains installed because the intended next hop
is inside the connected /24, but no device answers ARP for that address. I
will restore the golden address and then verify ARP, CEF, and a sourced
end-to-end ping.”

Interviewer Reaction Note: The root cause explains why the route looked
healthy while forwarding failed.

Candidate: “The repair is verified. ARP now resolves to the remote MAC,
CEF shows a complete adjacency through Gi0/0, and traffic sourced from
Loopback10 reaches the server LAN. I did not replace or weaken the
original routing design.”

Lab 12.1 Completion Gate
     The route remains installed through the intended next-hop IP.

     The ARP entry for 192.0.2.2 contains a real hardware address.

     CEF no longer reports an incomplete adjacency.

     The sourced ping and traceroute succeed in both directions.

     The candidate can explain why the RIB remained healthy during the
     failure.

 Sentence    Content

 1           What business symptom you are addressing

 2           What evidence you have collected so far

 3           What you currently believe is happening, stated as a hypothesis

 4           What the next command or change will test
