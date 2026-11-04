# Lab 7.1: The Broken Trust Boundary and Stolen

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 13632-14031). Review against the printed book before classroom use.*

Voice Queue
Time limit: 40 minutes

Difficulty: Advanced CCNP interview lab

1. THE TOPOLOGY & OBJECTIVE
A branch office uses a Cisco IP phone with a workstation connected
behind the phone. Voice and data enter separate VLANs on SW1 and are
routed by R1. R1 shapes the branch uplink to 10 Mbps and uses a strict
priority class for EF voice traffic. During nightly backups, calls become
choppy even though routing is stable and the WAN remains below its
physical interface speed.

Lab 7.1 topology

 PHONE1 10.10.20.11/24 (voice VLAN 20, marks RTP EF and
 signaling CS3)
     |
 PC1    10.10.10.11/24 (data VLAN 10, can generate an
 authorized test backup)
     |
 SW1 Gi1/0/10
     |
 SW1 Gi1/0/48 ===== trunk ===== R1 Gi0/0
                                   | Gi0/0.10 10.10.10.1/24
                                   | Gi0/0.20 10.10.20.1/24
                                   |
                              R1 Gi0/1 172.16.12.1/30
                                   |
                              R2 Gi0/0 172.16.12.2/30
                                   |
                    HQ voice and application network
 10.20.20.0/24

    PHONE1

      PC1

                Gi1/0/48             Gi0/1
     SW1                     R1                    R2               HQ Net 10.20.20.0/24
                Gi0/0                Gi0/0

            Figure: Lab 7.1 — Branch VoIP QoS Trust Boundary Physical Topology

 Device/Interface   Address          Role

 PHONE1             10.10.20.11/24   Voice VLAN 20, marks RTP EF and signaling
                                     CS3

 PC1                10.10.10.11/24   Data VLAN 10, generates authorized test backup

 SW1 Gi1/0/10       n/a              Access port, voice VLAN 20 / data VLAN 10

 SW1 Gi1/0/48       n/a              Trunk to R1

 R1 Gi0/0.10        10.10.10.1/24    Data subinterface

 R1 Gi0/0.20        10.10.20.1/24    Voice subinterface

 R1 Gi0/1           172.16.12.1/30   WAN uplink, shaped to 10 Mbps

Traffic Generation Assumptions
The interview environment includes authorized Linux test hosts. PHONE1
or a voice generator sends a low-rate UDP stream marked EF. PC1 sends a
higher-rate backup stream and can deliberately mark it EF to test the trust
boundary. The exact tools can be replaced with a commercial traffic
generator.

Optional authorized traffic generation

 # On the authorized HQ test receiver
 iperf3 -s

 # On PHONE1 or a voice test host: about 100 kbps, DSCP EF
 (TOS 0xb8)
 iperf3 -c 10.20.20.10 -u -b 100K -l 160 -t 120 -S 0xb8

 # On PC1: 8-Mbps backup stream falsely marked EF
 iperf3 -c 10.20.20.20 -u -b 8M -t 120 -S 0xb8

2. THE INITIAL GOLDEN CONFIG

SW1 Golden Configuration

 hostname SW1
 spanning-tree mode rapid-pvst
 !
 vlan 10
  name DATA
 vlan 20
  name VOICE
 !
 interface GigabitEthernet1/0/10
  description PHONE1-WITH-PC1
  switchport mode access
  switchport access vlan 10
  switchport voice vlan 20
  spanning-tree portfast
  spanning-tree bpduguard enable
  no shutdown
 !
 interface GigabitEthernet1/0/48
  description TRUNK-TO-R1
  switchport mode trunk
  switchport trunk allowed vlan 10,20
  no shutdown

R1 Golden Configuration

 hostname R1
 !
 interface GigabitEthernet0/0
  description 802.1Q-TRUNK-TO-SW1
  no ip address
  no shutdown
 !
 interface GigabitEthernet0/0.10
  encapsulation dot1Q 10
  ip address 10.10.10.1 255.255.255.0
  service-policy input PM-DATA-SANITIZE-IN
 !
 interface GigabitEthernet0/0.20
  encapsulation dot1Q 20
  ip address 10.10.20.1 255.255.255.0
  service-policy input PM-PHONE-TRUST-IN
 !
 interface GigabitEthernet0/1
  description WAN-TO-R2
  ip address 172.16.12.1 255.255.255.252
  service-policy output PM-WAN-PARENT
  no shutdown
 !
 ip route 10.20.20.0 255.255.255.0 172.16.12.2
 !
 class-map match-any CM-VOICE-EF
  match dscp ef
 class-map match-any CM-SIGNALING-CS3
  match dscp cs3
 class-map match-any CM-BUSINESS-AF31
  match dscp af31
 !
 policy-map PM-DATA-SANITIZE-IN
  class class-default
   set dscp default
 !
 policy-map PM-PHONE-TRUST-IN
  class CM-VOICE-EF
   set dscp ef
  class CM-SIGNALING-CS3
   set dscp cs3
  class class-default
   set dscp default
 !
 policy-map PM-WAN-CHILD

  class CM-VOICE-EF
   priority percent 20
  class CM-SIGNALING-CS3
   bandwidth percent 5
  class CM-BUSINESS-AF31
   bandwidth percent 25
  class class-default
   fair-queue
 !
 policy-map PM-WAN-PARENT
  class class-default
   shape average 10000000
   service-policy PM-WAN-CHILD

R2 Golden Configuration

 hostname R2
 !
 interface GigabitEthernet0/0
  description WAN-TO-R1
  ip address 172.16.12.2 255.255.255.252
  no shutdown
 !
 interface GigabitEthernet0/1
  description HQ-LAN
  ip address 10.20.20.1 255.255.255.0
  no shutdown
 !
 ip route 10.10.10.0 255.255.255.0 172.16.12.1
 ip route 10.10.20.0 255.255.255.0 172.16.12.1

Golden-State Verification

 R1# show policy-map interface GigabitEthernet0/0.10 input
 R1# show policy-map interface GigabitEthernet0/0.20 input
 R1# show policy-map interface GigabitEthernet0/1 output
 R1# show ip route 10.20.20.0
 R1# ping 10.20.20.1 source 10.10.20.1
 SW1# show interfaces trunk
 SW1# show interfaces GigabitEthernet1/0/10 switchport

With both test streams running, the voice VLAN input policy should
preserve EF and CS3. The data VLAN input policy should rewrite the

falsely marked PC stream to DSCP 0. On the WAN output, the EF class
should show only the low-rate voice stream and should not drop packets.

3. THE SABOTAGE SCRIPT
The interviewer injects two trust-boundary errors. They create misleading
output because the priority class remains busy, but the legitimate phone
traffic is no longer in it.

Sabotage commands

 R1# configure terminal
 !
 interface GigabitEthernet0/0.10
  no service-policy input PM-DATA-SANITIZE-IN
 !
 interface GigabitEthernet0/0.20
  no service-policy input PM-PHONE-TRUST-IN
  service-policy input PM-DATA-SANITIZE-IN
 !
 end

 Interface                Change

 GigabitEthernet0/0.10    Input sanitizer policy removed; forged EF markings can
 (data)                   now survive

 GigabitEthernet0/0.20    Phone trust policy replaced with the data sanitizer;
 (voice)                  legitimate EF/CS3 is erased

 Net effect               Priority queue is fed by the PC backup stream instead of
                          the phone

4. THE VERIFICATION WORKFLOW

Step 1: Confirm Basic Reachability and Physical Health

 R1# show ip interface brief
 R1# show interfaces GigabitEthernet0/1
 R1# show interfaces counters errors
 R1# show ip route 10.20.20.0
 R1# ping 10.20.20.1 source 10.10.20.1 repeat 5

The interfaces and route are healthy, and small pings succeed. This
prevents the investigation from incorrectly treating a routing or physical
failure as a QoS problem.

Step 2: Confirm the Egress Policy Is Attached

 R1# show policy-map interface GigabitEthernet0/1 output

Hunt for the following lines or equivalent output:

    `Service-policy output: PM-WAN-PARENT`

    `shape (average) cir 10000000`

    `Service-policy : PM-WAN-CHILD`

    Packet and byte counters under `CM-VOICE-EF`, `CM-SIGNALING-
    CS3`, and `class-default`.

    Priority class offered rate and any `drop` or `exceed` evidence.

    Class-default queue depth and total drops during the backup test.

A busy EF class does not prove that the packets are voice. The class is
matching a marking, and an untrusted PC may have forged that marking.

Step 3: Inspect Input Policy Attachment on the Data VLAN

 R1# show running-config interface GigabitEthernet0/0.10
 R1# show policy-map interface GigabitEthernet0/0.10 input

The expected `PM-DATA-SANITIZE-IN` policy is missing. The absence
of an input service policy means PC1 can retain DSCP EF. This is the first
trust-boundary failure.

Step 4: Inspect Input Policy Attachment on the Voice VLAN

 R1# show running-config interface GigabitEthernet0/0.20
 R1# show policy-map interface GigabitEthernet0/0.20 input

The voice subinterface is using `PM-DATA-SANITIZE-IN`. Its class-
default counter increases and the action shows DSCP being set to default.
Legitimate voice markings are therefore erased before the egress classifier
sees them.

Step 5: Prove the Packet Markings

Use an authorized packet capture on R1 or a SPAN destination when
supported. Capture one flow from PHONE1 and one from PC1 on the
WAN-facing side. The expected evidence is:

    PHONE1 RTP leaves R1 with DSCP 0 because the voice input policy
    was replaced.

    PC1 backup traffic leaves the data subinterface with DSCP EF
    because no sanitizer is attached.

    The WAN EF class is therefore serving the wrong application.

Representative IOS XE embedded capture workflow; verify platform
syntax

 R1# monitor capture QOS interface GigabitEthernet0/1 out
 match any
 R1# monitor capture QOS start
 ! Generate several seconds of authorized traffic.
 R1# monitor capture QOS stop
 R1# monitor capture QOS buffer brief
 R1# no monitor capture QOS

Step 6: Form the Root-Cause Hypothesis

 Evidence                                    Hypothesis

 Data subinterface missing PM-DATA-          PC1 can preserve a forged DSCP EF
 SANITIZE-IN                                 marking

 Voice subinterface running PM-DATA-         Legitimate phone markings are reset to
 SANITIZE-IN instead of PM-PHONE-            default before reaching the WAN
 TRUST-IN                                    classifier

Step 7: Restore the Data Sanitizer

 R1# configure terminal
 R1(config)# interface GigabitEthernet0/0.10
 R1(config-subif)# service-policy input PM-DATA-SANITIZE-IN
 R1(config-subif)# end
