# Lab 13.1: The Ambiguous Branch Outage

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 25046-25303). Review against the printed book before classroom use.*

Time Limit: 30 Minutes

Primary skill: Scoping, hypothesis narration, safe repair, and final
reporting.

1. THE TOPOLOGY & OBJECTIVE
A branch router reaches headquarters across a routed WAN. EIGRP
provides the branch routes. Users report that “the branch is offline,” but
the WAN interface remains up and the headquarters router can ping the
branch-facing WAN address. The candidate must define the actual failure
boundary, identify why branch LAN routes disappeared, restore the
adjacency, and explain why the physical WAN was never down.

 Branch LAN 10.40.40.0/24
         |
       BR1 Gi0/1 10.40.40.1/24
       BR1 Gi0/0 10.0.14.2/30
         |
         | Routed WAN
         |
       HQ1 Gi0/0 10.0.14.1/30
       HQ1 Lo0   10.255.1.1/32

                                                                            Lo0 10.255.1.1/32
                                                Gi0/0 10.0.14.2/30
                             Gi0/1
  Branch LAN 10.40.40.0/24            BR1                                        HQ1
                                                       Gi0/0 10.0.14.1/30
                                                     Routed WAN

              Figure: EIGRP Passive-Interface Failure Lab — Physical Topology

Business intent: HQ1 and BR1 must form an EIGRP named-mode
adjacency across Gi0/0. HQ1 must learn 10.40.40.0/24 dynamically. The
candidate should not replace the design with an emergency static route
unless explicitly directed.

2. THE INITIAL GOLDEN CONFIG
HQ1

 hostname HQ1
 !
 interface Loopback0
  ip address 10.255.1.1 255.255.255.255
 !
 interface GigabitEthernet0/0
  description WAN-TO-BR1
  ip address 10.0.14.1 255.255.255.252
  no shutdown
 !
 router eigrp ENTERPRISE
  address-family ipv4 unicast autonomous-system 100
   eigrp router-id 10.255.1.1
   network 10.0.14.0 0.0.0.3
   network 10.255.1.1 0.0.0.0
   passive-interface Loopback0
  exit-address-family

BR1

 hostname BR1
 !
 interface GigabitEthernet0/0
  description WAN-TO-HQ1
  ip address 10.0.14.2 255.255.255.252
  no shutdown
 !
 interface GigabitEthernet0/1
  description BRANCH-LAN
  ip address 10.40.40.1 255.255.255.0
  no shutdown
 !
 router eigrp ENTERPRISE
  address-family ipv4 unicast autonomous-system 100
   eigrp router-id 10.255.4.1
   network 10.0.14.0 0.0.0.3
   network 10.40.40.0 0.0.0.255
   passive-interface GigabitEthernet0/1
  exit-address-family

Golden verification:

 HQ1# show ip eigrp neighbors
 HQ1# show ip route 10.40.40.0
 BR1# show ip eigrp neighbors
 BR1# ping 10.255.1.1 source 10.40.40.1

3. THE SABOTAGE SCRIPT
Run on BR1:

 configure terminal
 router eigrp ENTERPRISE
  address-family ipv4 unicast autonomous-system 100
   passive-interface GigabitEthernet0/0
  exit-address-family
 end

The WAN interface remains up/up. Connected pings across 10.0.14.0/30
continue to work, but EIGRP hellos stop and HQ1 removes the branch
LAN route.

4. THE VERIFICATION WORKFLOW

Minute 0-3: Correct the Problem Statement

Candidate: “The phrase “branch offline” is broad. I will first determine
whether the physical WAN, the EIGRP adjacency, the branch route, or the
application path is failing. Can HQ1 still reach BR1’s WAN address?”

Interviewer Reaction Note: The candidate converts a user description
into testable layers.

On HQ1:

 show interfaces GigabitEthernet0/0
 ping 10.0.14.2
 show ip eigrp neighbors
 show ip route 10.40.40.0

Expected evidence: Gi0/0 is up/up, the connected ping succeeds, the
EIGRP neighbor is absent, and the branch LAN route is missing.

Minute 3-10: State and Test the Hypothesis

Candidate: “The connected WAN succeeds, so this is not a total branch or
physical-link outage. The failure boundary is EIGRP adjacency and
dynamic branch-route exchange. I will compare the EIGRP-enabled
interfaces and passive-interface state on both routers.”

Interviewer Reaction Note: The candidate prevents the broad user
symptom from driving a broad repair.

 show ip protocols
 show ip eigrp interfaces detail
 show running-config | section router eigrp

On BR1, look for `GigabitEthernet0/0` under passive interfaces. The WAN
interface may be included by an explicit command or by `passive-interface
default` without a corresponding `no passive-interface` exception.

Minute 10-17: Apply the Smallest Repair

Candidate: “The WAN interface is explicitly passive, which suppresses
EIGRP hellos while leaving IP connectivity intact. I will remove only that
passive setting. This should restore the neighbor and allow the existing
network statements to advertise the branch LAN.”

Interviewer Reaction Note: The candidate states mechanism, scope, and
expected outcome.

 configure terminal
 router eigrp ENTERPRISE
  address-family ipv4 unicast autonomous-system 100
   no passive-interface GigabitEthernet0/0
  exit-address-family
 end

Minute 17-25: Verify Protocol and Forwarding

 show ip eigrp neighbors
 show ip route 10.40.40.0
 show ip eigrp topology 10.40.40.0/24
 show ip cef 10.40.40.0/24
 ping 10.40.40.1 source 10.255.1.1

Verify on BR1 that the LAN interface remains passive. Removing passive
mode from the WAN should not accidentally enable neighbor formation on
the user LAN.

Minute 25-30: Deliver the Final Report

Candidate: “The branch WAN was never physically down. HQ1 could
reach BR1’s connected WAN address, but BR1 Gi0/0 was configured as an
EIGRP passive interface, so no hellos were sent and HQ1 removed
10.40.40.0/24. I removed the passive setting only from the WAN interface.
The neighbor is now established, the branch route is installed and resolved
in CEF, and sourced end-to-end testing succeeds. The branch LAN
remains passive as intended.”

Interviewer Reaction Note: The candidate closes with scope, cause,
minimal fix, proof, and preserved design.

5. THE INTERVIEW NARRATION SCRIPT
Candidate: “I will not treat “branch offline” as a protocol diagnosis. I am
separating physical WAN reachability, routing adjacency, route
installation, and business traffic.”

Interviewer Reaction Note: The candidate defines a disciplined scope.

Candidate: “Gi0/0 is up/up and the connected WAN ping succeeds. That
rejects a complete physical or Layer 3 transport failure.”

Interviewer Reaction Note: The candidate explicitly rejects a broad
hypothesis.

Candidate: “The EIGRP neighbor and branch prefix are both absent. My
next test is passive-interface state because it can suppress adjacency while
leaving connected ping operational.”

Interviewer Reaction Note: The next command follows from the
evidence.

Candidate: “BR1 Gi0/0 is passive. I will remove only that line and retain
passive mode on the branch LAN.”

Interviewer Reaction Note: The repair is narrow and preserves design
intent.

Candidate: “The neighbor is restored, HQ1 has the EIGRP route, CEF
resolves the correct next hop, and a sourced test succeeds. I have also
confirmed that no unnecessary static route was added.”

Interviewer Reaction Note: The final verification includes protocol,
route, forwarding, service, and design.
