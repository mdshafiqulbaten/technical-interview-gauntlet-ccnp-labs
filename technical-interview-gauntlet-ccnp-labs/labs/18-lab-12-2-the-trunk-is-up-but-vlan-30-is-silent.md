# Lab 12.2: The Trunk Is Up but VLAN 30 Is Silent

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 23968-24367). Review against the printed book before classroom use.*

Time Limit: 20 Minutes

1. THE TOPOLOGY & OBJECTIVE
Two access switches are connected by an operational 802.1Q trunk. VLAN
10 works across the link, but VLAN 30 endpoints cannot communicate.
The candidate must prove whether the failure is VLAN creation, trunk
carriage, spanning-tree forwarding, or MAC learning.

 VLAN 30 test SVI                         VLAN 30 test SVI
 SW1 Vlan30 10.30.30.1/24                 SW2 Vlan30
 10.30.30.2/24
        |                                         |
        +-- Gi0/1 ===== 802.1Q trunk ===== Gi0/1--+
                   Native VLAN 999
                   Allowed 10,30,999

       Vlan30 10.30.30.1/24                                       Vlan30 10.30.30.2/24

                              Gi0/1                      Gi0/1
             SW1                                                         SW2
                                      802.1Q trunk, native VLAN 999, allowed 10,30,999

               Figure: VLAN Carriage Across a Trunk Lab — Physical Topology

Final working state:

    VLANs 10, 30, and 999 exist on both switches.

    Gi0/1 is trunking on both switches with native VLAN 999.

    VLAN 30 is present in the allowed, active, and forwarding lists.

    Spanning tree forwards VLAN 30 across the trunk.

    The two VLAN 30 SVIs can ping each other.

2. THE INITIAL GOLDEN CONFIG
SW1:

 enable
 configure terminal
 hostname SW1
 spanning-tree mode rapid-pvst
 !
 vlan 10
  name USERS
 vlan 30
  name SERVERS
 vlan 999
  name NATIVE-BLACKHOLE
 !
 interface GigabitEthernet0/1
  description TRUNK-TO-SW2
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,30,999
  no shutdown
 !
 interface Vlan30
  ip address 10.30.30.1 255.255.255.0
  no shutdown
 end

SW2:

 enable
 configure terminal
 hostname SW2
 spanning-tree mode rapid-pvst
 !
 vlan 10
  name USERS
 vlan 30
  name SERVERS
 vlan 999
  name NATIVE-BLACKHOLE
 !
 interface GigabitEthernet0/1
  description TRUNK-TO-SW1
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,30,999
  no shutdown
 !
 interface Vlan30
  ip address 10.30.30.2 255.255.255.0
  no shutdown
 end

Golden verification:

 SW1# show interfaces trunk
 SW1# show spanning-tree vlan 30
 SW1# show mac address-table dynamic vlan 30
 SW1# ping 10.30.30.2 source Vlan30

3. THE SABOTAGE SCRIPT
Run on SW2:

 configure terminal
 interface GigabitEthernet0/1
  switchport trunk allowed vlan 10,999
 end

The trunk remains operational, CDP still works, and VLAN 10 still crosses
the link. Only VLAN 30 has been removed from the allowed list on one

side.

4. THE VERIFICATION WORKFLOW
Step 1: Confirm the narrow symptom.

 SW1# ping 10.30.30.2 source Vlan30
 .....
 SW1# show ip interface brief | include Vlan30
 Vlan30 10.30.30.1 YES manual up up

The local SVI is up/up. That proves VLAN 30 has at least one active Layer
2 port locally, but it does not prove the trunk carries the VLAN.

Step 2: Verify that VLAN 30 exists on both switches.

 SW1# show vlan brief
 SW2# show vlan brief

If VLAN 30 exists and is active on both devices, move to the trunk state
rather than recreating the VLAN.

Step 3: Compare trunk output on both ends.

 SW1# show interfaces trunk
 Port Vlans allowed on trunk
 Gi0/1 10,30,999

 SW2# show interfaces trunk
 Port Vlans allowed on trunk
 Gi0/1 10,999

The trunk is up, but the operational allowed list differs. This is sufficient to
form a strong hypothesis.

Step 4: Confirm the interface switchport configuration.

 SW2# show interfaces GigabitEthernet0/1 switchport
 SW2# show running-config interface GigabitEthernet0/1

Step 5: Use spanning-tree and MAC learning to validate the consequence.

 SW1# show spanning-tree vlan 30
 SW2# show spanning-tree vlan 30
 SW1# show mac address-table dynamic vlan 30
 SW2# show mac address-table dynamic vlan 30

SW1 may show Gi0/1 participating for VLAN 30, while SW2 does not
forward VLAN 30 on the trunk. Remote VLAN 30 MAC addresses are not
learned across the link.

Step 6: State the root cause.

 Evidence Collected                           Conclusion

 SW1 trunk shows VLAN 30 allowed; SW2         The allowed-VLAN lists on the two ends
 trunk does not                               of the trunk are asymmetric

 VLAN 30 exists and is active on both         The fault is not VLAN creation
 switches

 Spanning-tree and MAC learning show no       Confirms the allowed-list mismatch, not a
 VLAN 30 activity on SW2's trunk port         spanning-tree block, is the cause

Step 7: Add the missing VLAN without overwriting the existing list.

 SW2# configure terminal
 SW2(config)# interface GigabitEthernet0/1
 SW2(config-if)# switchport trunk allowed vlan add 30
 SW2(config-if)# end

Using `add` is safer than entering a replacement list from memory because
it preserves the currently approved VLANs.

Step 8: Verify carriage, forwarding, learning, and traffic.

 SW1# show interfaces trunk
 SW2# show interfaces trunk
 SW1# show spanning-tree vlan 30
 SW2# show spanning-tree vlan 30
 SW1# ping 10.30.30.2 source Vlan30
 SW1# show mac address-table dynamic vlan 30

5. THE INTERVIEW NARRATION SCRIPT
Candidate: “Because VLAN 10 works across the same physical link, I
will treat a complete physical or trunk failure as less likely. I will verify
VLAN 30 creation, trunk carriage, spanning-tree participation, and MAC
learning in that order.”

Interviewer Reaction Note: The candidate uses the narrow symptom to
reduce the failure domain.

Candidate: “VLAN 30 exists on both switches, and both local VLAN 30
SVIs are up. The trunk is operational, but the allowed VLAN list differs:
SW1 includes VLAN 30 and SW2 does not.”

Interviewer Reaction Note: The candidate found a direct operational
mismatch without changing configuration prematurely.

Candidate: “I will confirm the SW2 switchport configuration and then
add VLAN 30 to the existing list. I will use the `add` keyword so I do not
accidentally remove VLANs 10 or 999.”

Interviewer Reaction Note: The candidate communicates the safety reason
for the exact command syntax.

Candidate: “After the change, both switches show VLAN 30 allowed and
forwarding on Gi0/1. The VLAN 30 SVIs can ping each other, and remote
MAC learning is present. The root cause was an asymmetric trunk allowed
list, not a physical or spanning-tree failure.”

12.15 Rapid Interview Drills
Drill   Scenario Prompt                            Target Skill

1       "Ping works but the application            ACL shadow-match recognition
        times out"

2       "The route is there but the packet         RIB vs. CEF vs. ARP reasoning
        never arrives"

3       "One VLAN is silent on an                  Allowed-list and spanning-tree
        otherwise healthy trunk"                   verification

4       "A routing neighbor won't come up"         Interface-parameter comparison before
                                                   process-level changes

5       "A static route keeps overriding a         Administrative-distance comparison
        healthy dynamic route"

6       "A GRE-based neighbor is stuck             MTU/DBD-level reasoning
        mid-negotiation"

7       "QoS looks configured but voice still      Policy-map and queue counter
        degrades"                                  interpretation under load

8       "An API call reports success but           Separating transport, auth, and state-
        nothing changed"                           convergence evidence

12.16 Command Safety Checklist
    Preserve an out-of-band path before changing routing, AAA, ACL,
    CoPP, or management services.

    Record the original configuration and the rollback command before
    applying a change.

    Avoid `clear` commands until counters and historical state have been
    captured.

    Avoid broad debugs; use filters and stop them immediately after
    collecting the event.

    Do not remove a security control merely to prove that it caused the
    symptom.

    Do not replace dynamic routing with an unapproved static shortcut.

    Use additive trunk and policy syntax when it reduces the risk of
    overwriting approved state.

    Verify from both sides of a relationship and with the same source,
    destination, protocol, and size as the application.

    After failover testing, restore the normal path and prove
    reconvergence.

    End with a root-cause statement that explains every original symptom.

12.17 Rapid Review Questions
1. What does `show ip route` prove?

It proves the route installed in the RIB and identifies the selected source,
metric, next hop, and interface. It does not prove complete forwarding.

2. What should follow a valid route when traffic still fails?

CEF exact-route, adjacency or ARP resolution, sourced tests, return
routing, and policy checks.

3. Why can a trunk be up while one VLAN fails?

The VLAN can be missing, pruned, blocked, or absent from the allowed
list while other VLANs continue forwarding.

4. What is the value of checking counter deltas?

The rate of increase correlates counters with the current incident and
separates historical noise from active failure.

5. Why is 2-WAY not always an OSPF fault?

DROTHER routers on a broadcast segment normally remain 2-WAY with
each other.

6. What does an EIGRP route in the topology table but not the RIB
suggest?

A competing route source may have a lower administrative distance or
better RIB preference.

7. What does an incomplete ARP entry indicate?

The device is attempting to resolve a directly connected next-hop IP but
has not learned its MAC address.

8. Why should an ACL be checked through hit counters?

Counters prove which sequence actually matches the test traffic and
expose shadowing or wrong attachment.

9. What does IPsec encapsulation without decapsulation suggest?

Outbound traffic is being encrypted, but return encrypted traffic is absent,
misrouted, filtered, or not accepted.

10. Why can small pings mislead tunnel troubleshooting?

They do not test path MTU, TCP MSS, application payload size, or
fragmentation behavior.

11. When should platform hardware commands be used?

After generic operational commands narrow the fault to a forwarding or
hardware path and the exact platform syntax is known.

12. What is the strongest closing verification?

A current business transaction plus protocol, RIB, CEF, policy, and
failover evidence that all agree with the intended design.
