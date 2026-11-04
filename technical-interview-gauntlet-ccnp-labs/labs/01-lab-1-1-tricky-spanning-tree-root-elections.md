# Lab 1.1: Tricky Spanning-Tree Root Elections

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 1382-1781). Review against the printed book before classroom use.*

 Step   Action

 1      Identify the active root bridge per VLAN before trusting any naming convention

 2      Compare configured priority and operational bridge ID on every candidate switch

 3      Walk root-port selection using cost, sender bridge ID, sender port ID, and local
        port ID in order

 4      Confirm EtherChannel and trunk consistency before blaming spanning tree

 5      Apply the smallest corrective change and verify before declaring the fix complete

1. THE TOPOLOGY & OBJECTIVE
A company uses two distribution switches and one access switch. The
intended design is deterministic: DSW1 must be the root for VLANs 10,
20, 30, and 999; DSW2 must be secondary; and ASW1 must prefer its
Gi0/1 link toward DSW1.

Physical topology

                               Gi0/3       Gi0/3
                         +-------------+-------------+
                         |                           |
                     +---+---+                   +---+---+
                     | DSW1 |                    | DSW2 |
                     +---+---+                   +---+---+
                  Gi0/1 | | Gi0/2                    | Gi0/1
                        | |                          |
                  Gi0/1 | | Gi0/2              Gi0/3|
                     +--+--+-------------------------+
                     |             ASW1              |
                     +-------------------------------+

                             Gi0/3                        Gi0/3
             DSW1                                                  DSW2

            Gi0/1   Gi0/2                                          Gi0/1

                                     Gi0/1        Gi0/2   Gi0/3

                                             ASW1

                            Figure: Lab 1.1 — Physical Topology

Link map

 DSW1 Gi0/1 <-> ASW1 Gi0/1
 DSW1 Gi0/2 <-> ASW1 Gi0/2
 DSW1 Gi0/3 <-> DSW2 Gi0/3
 DSW2 Gi0/1 <-> ASW1 Gi0/3

For a deterministic lab, assign the hostname DSW2 to the distribution
switch with the lower bridge MAC address and DSW1 to the switch with
the higher bridge MAC address. When the sabotage creates an equal-
priority election, DSW2 will then win predictably.

Compare base MAC addresses before beginning

 show spanning-tree vlan 10
 show version

 Switch    Base MAC Address             Computed Bridge ID (VLAN 10)

 DSW1      (higher base MAC)            Priority + extended system ID + base MAC

 DSW2      (lower base MAC)             Priority + extended system ID + base MAC

2. THE INITIAL GOLDEN CONFIG

DSW1 Golden Configuration

 enable
 configure terminal
 hostname DSW1
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
 spanning-tree vlan 10,20,30,999 priority 24576
 !
 interface GigabitEthernet0/1
  description TRUNK-TO-ASW1-GI0/1
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  spanning-tree vlan 10,20,30,999 port-priority 64
  no shutdown
 !
 interface GigabitEthernet0/2
  description TRUNK-TO-ASW1-GI0/2
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  spanning-tree vlan 10,20,30,999 port-priority 128
  no shutdown
 !
 interface GigabitEthernet0/3
  description TRUNK-TO-DSW2-GI0/3
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 end
 write memory

DSW2 Golden Configuration

 enable
 configure terminal
 hostname DSW2
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
 spanning-tree vlan 10,20,30,999 priority 28672
 !
 interface GigabitEthernet0/1
  description TRUNK-TO-ASW1-GI0/3
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 !
 interface GigabitEthernet0/3
  description TRUNK-TO-DSW1-GI0/3
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 end
 write memory

ASW1 Golden Configuration

 enable
 configure terminal
 hostname ASW1
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
 interface GigabitEthernet0/1
  description TRUNK-TO-DSW1-GI0/1
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 !
 interface GigabitEthernet0/2
  description TRUNK-TO-DSW1-GI0/2
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 !
 interface GigabitEthernet0/3
  description TRUNK-TO-DSW2-GI0/1
  switchport mode trunk
  switchport trunk native vlan 999
  switchport trunk allowed vlan 10,20,30,999
  no shutdown
 !
 interface range GigabitEthernet0/10-20
  description USER-EDGE-PORT
  switchport mode access
  switchport access vlan 10
  spanning-tree portfast
  spanning-tree bpduguard enable
  shutdown
 end
 write memory

Golden-State Verification

Run on DSW1 and ASW1

 show spanning-tree root
 show spanning-tree vlan 10
 show spanning-tree vlan 20
 show spanning-tree vlan 30
 show spanning-tree vlan 999

DSW1 should identify itself as root. ASW1 should show Gi0/1 as root and
forwarding. The other redundant uplinks should remain alternate and
discarding for the relevant VLANs.

3. THE SABOTAGE SCRIPT

Fault 1: Create a Root-Priority Tie

Run on DSW1

 configure terminal
 spanning-tree vlan 10,20,30,999 priority 28672
 end

DSW1 and DSW2 now use the same configured priority. DSW2 wins
because the lab preparation assigned DSW2 to the switch with the lower
bridge MAC address.

Fault 2: Manipulate the Upstream Sender Port ID

Run on DSW1

 configure terminal
 interface GigabitEthernet0/1
  spanning-tree vlan 10,20,30,999 port-priority 128
 interface GigabitEthernet0/2
  spanning-tree vlan 10,20,30,999 port-priority 32
 end

After the root election is repaired, ASW1 will prefer the BPDU sent from
DSW1 Gi0/2. The root ID, path cost, and sender bridge ID are identical
across the two direct links, so the lower sender port ID wins.

    DSW2 unexpectedly operates as root.

    ASW1 initially points toward DSW2.

    After correcting the root, ASW1 selects Gi0/2 instead of the required
    Gi0/1.

    No physical link has failed.

    The network may continue forwarding, but the topology violates the
    business design.

4. THE VERIFICATION WORKFLOW

Step 1: Confirm the physical topology

 show cdp neighbors
 show interfaces description
 show interfaces status

Run on ASW1. Confirm that Gi0/1 and Gi0/2 connect to DSW1 and Gi0/3
connects to DSW2. Treat interface descriptions as supporting evidence
rather than proof.

Step 2: Establish the actual root

 show spanning-tree root

If the root port for all affected VLANs is Gi0/3, the active root path points
toward DSW2.

Step 3: Read the complete bridge IDs

 show spanning-tree vlan 10

Run on ASW1, DSW1, and DSW2. Identify which switch states “This
bridge is the root.” Compare Root ID, Bridge ID, priority, and MAC
address.

Step 4: Compare configuration with operational values

 show running-config | include spanning-tree vlan

Run on both distribution switches. Equal configured priorities force the
election to continue to the MAC-address comparison.

 Switch   Configured Priority (VLAN        Operational (Displayed) Priority
          10)

 DSW1     24576                            24586 (priority + VLAN 10 extended
                                           system ID)

 DSW2     24576                            24586 (priority + VLAN 10 extended
                                           system ID)

Step 5: Restore the Intended Root

Run on DSW1

 configure terminal
 spanning-tree vlan 10,20,30,999 priority 24576
 end

An alternative is the root-primary macro, but an explicit value is easier to
audit during a controlled lab. State that the change triggers reconvergence
and may briefly affect forwarding.

Step 6: Verify Root Recovery

Run on all three switches

 show spanning-tree root
 show spanning-tree vlan 10

DSW1 should now report that it is root. Do not stop. The business
requirement also specifies which parallel link ASW1 must prefer.

Step 7: Identify the Unexpected Root Port

Run on ASW1

 show spanning-tree vlan 10

Representative result

 Interface    Role    Sts   Cost   Prio.Nbr    Type
 Gi0/1        Altn    BLK   4      128.1       P2p
 Gi0/2        Root    FWD   4      128.2       P2p
 Gi0/3        Altn    BLK   4      128.3       P2p
