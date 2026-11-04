# Lab 13.2: Rejecting the Attractive ACL Hypothesis

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 25304-25703). Review against the printed book before classroom use.*

Time Limit: 45 Minutes

Primary skill: Recovering from a wrong theory and pivoting to packet-size
evidence.

1. THE TOPOLOGY & OBJECTIVE
Two sites use a GRE tunnel across an IP underlay. OSPF is FULL and
small pings succeed. Users report that SSH connects, but large file
transfers and HTTPS uploads stall. An interface ACL appears near the
tunnel configuration, making ACL troubleshooting an attractive first
theory. The actual failure is an oversized tunnel MTU combined with no
TCP MSS adjustment.

 LAN-A 10.10.10.0/24 -- R1 -- GRE/IP underlay -- R2 --
 10.20.20.0/24 LAN-B

 Tunnel0: 172.16.12.1/30 <-> 172.16.12.2/30
 Underlay: 192.0.2.1/30 <-> 192.0.2.2/30
 Physical underlay MTU available for encapsulated packets:
 1476 bytes

                                                  Tunnel0 172.16.12.1/30 <-> .2/30
                        Gi0/1                                                             Gi0/1
  LAN-A 10.10.10.0/24                  R1                                            R2           LAN-B 10.20.20.0/24

                                                  Underlay 192.0.2.1/30 <-> .2/30

                                Figure: GRE Tunnel MTU/MSS Lab — Physical Topology

Business intent: OSPF must remain FULL across Tunnel0. Large TCP
applications must work without relying on IP fragmentation. The final
design must use an appropriate tunnel MTU and TCP MSS adjustment on
both tunnel endpoints.

2. THE INITIAL GOLDEN CONFIG
R1

 hostname R1
 !
 interface GigabitEthernet0/0
  ip address 192.0.2.1 255.255.255.252
  no shutdown
 !
 interface GigabitEthernet0/1
  ip address 10.10.10.1 255.255.255.0
  no shutdown
 !
 interface Tunnel0
  ip address 172.16.12.1 255.255.255.252
  ip mtu 1400
  ip tcp adjust-mss 1360
  tunnel source GigabitEthernet0/0
  tunnel destination 192.0.2.2
 !
 ip access-list extended TUNNEL-MGMT
  permit tcp 10.10.10.0 0.0.0.255 10.20.20.0 0.0.0.255
 established
  permit icmp any any
 !
 router ospf 10
  router-id 1.1.1.1
  network 10.10.10.0 0.0.0.255 area 0
  network 172.16.12.0 0.0.0.3 area 0

R2

 hostname R2
 !
 interface GigabitEthernet0/0
  ip address 192.0.2.2 255.255.255.252
  no shutdown
 !
 interface GigabitEthernet0/1
  ip address 10.20.20.1 255.255.255.0
  no shutdown
 !
 interface Tunnel0
  ip address 172.16.12.2 255.255.255.252
  ip mtu 1400
  ip tcp adjust-mss 1360
  tunnel source GigabitEthernet0/0
  tunnel destination 192.0.2.1
 !
 router ospf 10
  router-id 2.2.2.2
  network 10.20.20.0 0.0.0.255 area 0
  network 172.16.12.0 0.0.0.3 area 0

3. THE SABOTAGE SCRIPT
Run on R1 and R2:

 configure terminal
 interface Tunnel0
  no ip mtu 1400
  no ip tcp adjust-mss 1360
 end

The tunnel remains up/up. OSPF can remain FULL because ordinary
hellos and many routing packets are small. Small pings succeed. Larger
packets with the DF bit set fail, and TCP endpoints may advertise an MSS
too large for the encapsulated path.

4. THE VERIFICATION WORKFLOW

Minute 0-6: Define the Packet-Size Boundary

Candidate: “The failure is not total reachability because SSH connects
and small pings work. The symptom is application and payload-size
dependent. I will still verify the ACL because it is an obvious candidate,
but I will require counter evidence before changing it.”

Interviewer Reaction Note: The candidate acknowledges the attractive
theory without becoming committed to it.

 show ip ospf neighbor
 show interfaces Tunnel0
 show access-lists TUNNEL-MGMT
 show ip interface Tunnel0

If the ACL is not applied to Tunnel0, or its permit counters increment
without corresponding deny increments, the ACL theory is not supported.

Minute 6-14: Reject the Wrong Hypothesis

Candidate: “The ACL is not dropping this traffic. The relevant permit
counter increments and no deny counter changes during the failed transfer.
I am withdrawing the ACL hypothesis. The remaining differentiator is
packet size, so I will test the path with controlled DF-bit pings.”

Interviewer Reaction Note: The candidate converts disproof into a
sharper next test.

On R1:

 ping 10.20.20.1 source 10.10.10.1 size 1200 df-bit
 ping 10.20.20.1 source 10.10.10.1 size 1400 df-bit
 ping 10.20.20.1 source 10.10.10.1 size 1450 df-bit

The exact threshold depends on platform and encapsulation overhead. The
evidence pattern matters: smaller DF-bit packets succeed while larger
packets fail.

Minute 14-24: Compare MTU and TCP MSS

 show interfaces Tunnel0 | include MTU|line protocol
 show ip interface Tunnel0
 show running-config interface Tunnel0
 show tcp brief all

Look for a default tunnel IP MTU that exceeds the safe payload size and
the absence of `ip tcp adjust-mss`. Explain that OSPF FULL proves
routing adjacency, not application-sized transport.

Minute 24-33: Apply the Design Repair

Candidate: “The path supports smaller packets but rejects larger DF-bit
traffic after GRE overhead. I will restore an IP MTU of 1400 and clamp
TCP MSS to 1360 on both tunnel endpoints. This avoids depending on
fragmentation and keeps TCP segments within the safe path size.”

Interviewer Reaction Note: The candidate states why both settings are
needed.

Run on R1 and R2:

 configure terminal
 interface Tunnel0
  ip mtu 1400
  ip tcp adjust-mss 1360
 end

Minute 33-41: Verify the Application-Sized Path

 show interfaces Tunnel0 | include MTU|line protocol
 show ip ospf neighbor
 ping 10.20.20.1 source 10.10.10.1 size 1372 df-bit
 show running-config interface Tunnel0

Repeat the actual HTTPS upload or file-transfer test. Existing TCP
sessions may retain the old negotiated MSS, so establish a new session
after the correction.

Minute 41-45: Deliver the Final Report

Candidate: “The tunnel, OSPF adjacency, and small-packet reachability
were healthy, so this was not a general routing failure. I initially tested the
nearby ACL, but its counters rejected that hypothesis. DF-bit testing then
proved a packet-size threshold. Both tunnel endpoints had lost the intended
MTU 1400 and TCP MSS 1360 settings, allowing oversized encapsulated
traffic. I restored both values, verified OSPF remained FULL, confirmed
the larger DF-bit test within the designed limit, and completed a new
application transfer successfully.”

Interviewer Reaction Note: The candidate openly reports the rejected
hypothesis and the evidence-driven pivot.

5. THE INTERVIEW NARRATION SCRIPT
Candidate: “Small pings and SSH work, so I am not calling this a tunnel
outage. The differentiator is large application traffic.”

Interviewer Reaction Note: The candidate defines the failure boundary
before choosing a protocol.

Candidate: “The ACL is visually suspicious, but I will not edit it without
hit-counter evidence.”

Interviewer Reaction Note: The candidate resists configuration bias.

Candidate: “The permit counter increments and denies do not. That
rejects my first hypothesis. I am moving to path MTU because packet size
is the strongest remaining variable.”

Interviewer Reaction Note: A wrong theory is retired cleanly.

Candidate: “The DF-bit tests establish a threshold, and the tunnel lacks
both the designed MTU and MSS clamp. I will restore those settings on
both ends.”

Interviewer Reaction Note: The repair follows converging evidence.

Candidate: “The final proof is a new TCP session and the actual
application transfer, not only another small ping.”

Interviewer Reaction Note: Verification matches the business symptom.

13.19 Interview Phrasebook
Situation                    Phrase to Use

Starting the investigation   "Before I run commands, I want to confirm the intended
                             design against what I'm observing."

Stating a hypothesis         "My current hypothesis is that... If that's correct, the next
                             command should show..."

Rejecting a hypothesis       "That evidence does not support my theory. I'm withdrawing
                             it and moving to..."

Before a disruptive          "Here is the intent, expected impact, scope, and rollback
change                       before I proceed."

Recovering from a            "This platform uses different syntax. Let me confirm the
syntax error                 supported form."

Responding to an             "That's a useful point. Let me fold it into the current
interviewer hint             hypothesis."

Managing time                "I'm at the halfway checkpoint. Here's what's proven and
                             what remains."

Closing the lab              "Here is the business impact, root cause, fix, proof, and
                             prevention recommendation."

13.20 Seven-Day Communication Practice Plan
 Day    Focus

 Day    Practice separating observation from interpretation on recorded lab transcripts

 Day    Drill the four-part think-aloud update (fact, interpretation, next test, expected
 2      result)

 Day    Practice forming and stating falsifiable hypotheses before running commands

 Day    Rehearse risk-communication statements for disruptive changes (intent, impact,
 4      scope, rollback)

 Day    Run a timed lab focused solely on checkpoint discipline and the two-minute
 5      dead-end rule

 Day    Practice recovering aloud from a deliberately wrong hypothesis or syntax
 6      mistake

 Day    Deliver full final root-cause reports from memory for three different past labs

Chapter 13 Review Questions
1. Why is a successful ping insufficient as final verification?

Answer: It proves only one packet exchange. It may not test the
application port, return path, MTU, policy, intended routing source, or
failover behavior.

2. What makes a hypothesis testable?

Answer: It identifies a mechanism and predicts observable evidence that
can confirm or reject it.

3. How should a candidate respond when evidence disproves a theory?

Answer: State the rejecting evidence, withdraw the theory, preserve useful
facts, redefine the boundary, and choose the next highest-value test.

4. What should be stated before a potentially disruptive change?

Answer: Intent, expected impact, scope, rollback, and verification method.

5. Why should command purpose be narrated before the command?

Answer: It shows that the command is a deliberate test rather than a
remembered ritual and prevents post-hoc interpretation.

6. What is the difference between operational state and intended state?

Answer: Operational state describes what the network is doing now.
Intended state describes what the approved design requires.

7. How can a candidate avoid excessive thinking aloud?

Answer: Use brief updates containing fact, interpretation, next test, and
expected result, then run the command.

8. What is the two-minute dead-end rule?

Answer: When an investigation path produces no useful evidence for
roughly two minutes, stop, restate the problem, and choose a higher-value
test.

9. How should a candidate handle unfamiliar platform syntax?

Answer: Identify the conceptual question, use portable commands and
context-sensitive help, and avoid guessing hardware-specific syntax.

10. What are the six components of a strong final report?

Answer: Business impact, technical symptom, confirmed root cause,
corrective action, verification evidence, and prevention recommendation.

11. Why can the nearest suspicious configuration be misleading?

Answer: Visual proximity does not prove packet matching or causation.
Operational counters and packet-path evidence are required.

12. What does professional composure look like after a mistake?

Answer: Acknowledge the mistake, assess impact, correct it, verify
recovery, and continue without concealment or defensiveness.

Chapter 13 Closing Principle
A senior network engineer is not the person who never becomes uncertain.
It is the person who makes uncertainty visible, testable, and progressively
smaller without exposing the business to unnecessary risk.

 State the intended state.
 Name the fact.
 Separate the interpretation.
 Form one testable hypothesis.
 Predict the evidence.
