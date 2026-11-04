# Lab 1.2: LACP and Native VLAN Mismatch

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 1967-2366). Review against the printed book before classroom use.*

Resolution
 Root Cause               Corrective Command           Verification

 Equal priority forcing   spanning-tree vlan           show spanning-
 a MAC-address tie-       10,20,30,999 priority        tree vlan 10
 break                    24576 on DSW1

 Inferior sender port     spanning-tree vlan           show spanning-
 priority on the          10,20,30,999 port-           tree vlan 10
 intended root port       priority 32 on DSW1 Gi0/2    detail
 link

1. THE TOPOLOGY & OBJECTIVE
Two enterprise switches are connected through a two-link LACP
EtherChannel. Both physical interfaces must form Port-channel 10, the

logical interface must operate as an 802.1Q trunk, VLANs 10, 20, 30, and
999 must cross the trunk, and VLAN 999 must be native on both ends.

Topology

                    Gi0/1 ---------------- Gi0/1
               +-----------+            +-----------+
               |    SW1    |            |    SW2    |
               +-----------+            +-----------+
                    Gi0/2 ---------------- Gi0/2

                                  Port-channel 10
                                  LACP trunk
                                  Native VLAN 999

                                   Gi0/1            Gi0/1

                   SW1                                            SW2
                                   Gi0/2            Gi0/2

                       Port-channel 10 (LACP trunk, native VLAN 999)

                            Figure: Lab 1.2 — Physical Topology

 Requirement                               Confirms

 Both physical links join Port-            LACP negotiation succeeded on both members
 channel10

 Port-channel10 operates as an             Trunk encapsulation and allowed-VLAN list are
 802.1Q trunk                              consistent

 VLANs 10, 20, 30, 999 cross the           Allowed-VLAN lists match on both switches
 trunk

 VLAN 999 is native on both ends           No native VLAN mismatch warnings

2. THE INITIAL GOLDEN CONFIG

SW1 Golden Configuration

 enable
 configure terminal
 hostname SW1
 no ip domain-lookup
 spanning-tree mode rapid-pvst
 !
 vlan 10
  name USERS
 vlan 20
  name VOICE
 vlan 30
  name SERVERS
 vlan 999
  name NATIVE-BLACKHOLE
 !
 interface range GigabitEthernet0/1-2
  description LACP-MEMBER-TO-SW2
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  channel-group 10 mode active
  no shutdown
 !
 interface Port-channel10
  description LACP-TRUNK-TO-SW2
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 end
 write memory

SW2 Golden Configuration

 enable
 configure terminal
 hostname SW2
 no ip domain-lookup
 spanning-tree mode rapid-pvst
 !
 vlan 10
  name USERS
 vlan 20
  name VOICE
 vlan 30
  name SERVERS
 vlan 999
  name NATIVE-BLACKHOLE
 !
 interface range GigabitEthernet0/1-2
  description LACP-MEMBER-TO-SW1
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  channel-group 10 mode active
  no shutdown
 !
 interface Port-channel10
  description LACP-TRUNK-TO-SW1
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 end
 write memory

Golden-State Verification

Run on both switches

 show etherchannel summary
 show lacp neighbor
 show interfaces trunk
 show spanning-tree vlan 10

Expected EtherChannel summary

 Group Port-channel Protocol Ports
 ------+-------------+---------+---------------------------
 -
 10     Po10(SU)      LACP      Gi0/1(P) Gi0/2(P)

Spanning tree should display Port-channel10 as the logical interface. The
physical member links should not appear as independent parallel paths.

3. THE SABOTAGE SCRIPT

Fault 1: Passive on Both Sides

Run on SW1

 configure terminal
 interface range GigabitEthernet0/1-2
  no channel-group 10
  channel-group 10 mode passive
 end

Run on SW2

 configure terminal
 interface range GigabitEthernet0/1-2
  no channel-group 10
  channel-group 10 mode passive
 end

Both sides now wait for an initiating LACP partner. The physical links
may remain up, but the logical channel does not form.

Fault 2: Native VLAN Mismatch

Run on SW2

 configure terminal
 interface Port-channel10
  switchport trunk native vlan 30
 end

If the virtual image removes Port-channel10 after member detachment,
recreate the logical interface and then apply the mismatched native VLAN.

Conditional recreation

 configure terminal
 interface Port-channel10
  description LACP-TRUNK-TO-SW1
  switchport mode trunk
  switchport trunk native vlan 30
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 end

    Before LACP repair, Po10 is down and members may appear
    individual or suspended.

    LACP neighbor information may be absent.

    The trunk mismatch may be concealed while the logical channel is
    down.

    After LACP repair, native VLAN mismatch and PVID inconsistency
    become visible.

4. THE VERIFICATION WORKFLOW

Step 1: Confirm the Physical Links

Run on SW1

 show interfaces status
 show interfaces GigabitEthernet0/1
 show interfaces GigabitEthernet0/2

A connected physical state proves only that Layer 1 is available. It does
not prove that the interfaces are participating in the logical bundle.

Step 2: Examine the Logical Bundle

 show etherchannel summary

Representative unhealthy output

 Group Port-channel Protocol Ports
 ------+-------------+---------+---------------------------
 -
 10     Po10(SD)      LACP      Gi0/1(I) Gi0/2(I)

The precise flags can vary by software release. The important fact is that
the members are not marked P and the logical channel is not in use.

Step 3: Verify LACP Partner Discovery

Run locally and then on the peer

 show lacp neighbor
 show running-config interface GigabitEthernet0/1
 show running-config interface GigabitEthernet0/2
 show etherchannel 10 detail

No LACP neighbor can indicate passive-to-passive configuration, a non-
LACP peer, a down link, incompatible members, static channel mode on
one side, or an incorrect cable mapping. Verify both ends before selecting
a cause.

 Flag   Meaning

 S      Device is sending according to the LACP standard rate

 A      Device is in active mode

 D      Port is down

 C      Port is bundled in the port channel

 P      Port is bundled and passing traffic (healthy)

 I      Port is individual — not bundled, LACP not negotiated with any neighbor

Step 4: Correct LACP Negotiation

Run on SW1; leave SW2 passive

 configure terminal
 interface range GigabitEthernet0/1-2
  no channel-group 10
  channel-group 10 mode active
 end

Active-to-passive is a valid combination. After the change, verify the
channel on both switches rather than assuming the peer accepted the
bundle.

Step 5: Verify Bundle Formation

Run on both switches

 show etherchannel summary
 show lacp neighbor

Healthy target

 Po10(SU)
 Gi0/1(P)
 Gi0/2(P)

Do not declare success. EtherChannel formation does not prove that the
logical trunk parameters match.

Step 6: Check the Trunk State

Run on both switches

 show interfaces trunk
 show interfaces Port-channel10 switchport
 show running-config interface Port-channel10

Mismatch evidence

 SW1 Po10 native VLAN: 999
 SW2 Po10 native VLAN: 30

Step 7: Examine Logs and Spanning-Tree Protection

 show logging | include NATIVE|PVID|SPANTREE|CDP
 show spanning-tree inconsistentports
 show spanning-tree vlan 30
 show spanning-tree vlan 999

Depending on the platform and release, the output may contain native
VLAN mismatch, PVID inconsistency, broken, blocking, or inconsistent
indicators. Treat the block as protective evidence. Do not disable spanning
tree.

 Log/Output Pattern             Meaning

 Native VLAN mismatch           Untagged traffic on the trunk is being associated with
 discovered                     different VLANs on each end

 Port blocking on one or more   Spanning tree's PVID-inconsistency protection has
 VLANs                          engaged to prevent a Layer 2 loop or VLAN leak

 Inconsistent / broken          Confirms the mismatch rather than a transient
 indicator in trunk output      negotiation delay

Step 8: Correct the Native VLAN

Run on SW2
