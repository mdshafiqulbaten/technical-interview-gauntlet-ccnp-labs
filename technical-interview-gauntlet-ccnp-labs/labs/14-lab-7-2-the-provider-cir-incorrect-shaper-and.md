# Lab 7.2: The Provider CIR, Incorrect Shaper, and

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 14158-14557). Review against the printed book before classroom use.*

Hidden Business Policer
Time limit: 45 minutes

Difficulty: Advanced multistage QoS triage

1. THE TOPOLOGY & OBJECTIVE
A branch router connects to a provider through a 1-Gbps Ethernet handoff.
The purchased service rate is 20 Mbps. The provider enforces that rate
with an ingress policer. Users report broad packet loss during backups, and
a critical HTTPS application remains slow even after the aggregate loss is
reduced.

Lab 7.2 topology

 Branch data VLAN 10: 10.30.10.0/24
 Branch voice VLAN 20: 10.30.20.0/24
              |
              BR1 Gi0/0 trunk
              |
 BR1 Gi0/1 192.0.2.1/30 ===== 1-Gbps Ethernet handoff =====
 ISP1 Gi0/0 192.0.2.2/30

 |
                                                      Business server
 10.40.40.40/32
                                                      Bulk server
 10.40.50.50/32

   Branch VLANs 10/20

             Gi0/0 trunk

                                   1-Gbps handoff

                           Gi0/1         Gi0/0
         BR1                                        ISP1                   Business Server
                                                                             Bulk Server

       Figure: Lab 7.2 — Branch Shaping and Provider Policer Physical Topology

Device/Interface              Address                      Role

BR1 Gi0/0 (trunk)             n/a                          Branch VLAN trunk to local switch

BR1 Gi0/0.10                  10.30.10.0/24                Data VLAN

BR1 Gi0/0.20                  10.30.20.0/24                Voice VLAN

BR1 Gi0/1                     192.0.2.1/30                 1-Gbps handoff, purchased 20-Mbps
                                                           CIR

ISP1 Gi0/0                    192.0.2.2/30                 Provider edge, enforces 20-Mbps
                                                           ingress policer

Business/Bulk                 10.40.40.40/32,              Destinations for the HTTPS business
servers                       10.40.50.50/32               app and bulk transfer

2. THE INITIAL GOLDEN CONFIG

BR1 Golden Configuration

 hostname BR1
 !
 interface GigabitEthernet0/0
  no ip address
  no shutdown
 !
 interface GigabitEthernet0/0.10
  encapsulation dot1Q 10
  ip address 10.30.10.1 255.255.255.0
 !
 interface GigabitEthernet0/0.20
  encapsulation dot1Q 20
  ip address 10.30.20.1 255.255.255.0
 !
 interface GigabitEthernet0/1
  description 20-MBPS-PROVIDER-SERVICE
  bandwidth 20000
  ip address 192.0.2.1 255.255.255.252
  service-policy output PM-BRANCH-PARENT
  no shutdown
 !
 ip route 10.40.40.40 255.255.255.255 192.0.2.2
 ip route 10.40.50.50 255.255.255.255 192.0.2.2
 !
 ip access-list extended ACL-BUSINESS
  permit tcp 10.30.10.0 0.0.0.255 host 10.40.40.40 eq 443
 ip access-list extended ACL-BULK
  permit tcp 10.30.10.0 0.0.0.255 host 10.40.50.50
 !
 class-map match-any CM-VOICE-EF
  match dscp ef
 class-map match-any CM-BUSINESS-AF31
  match access-group name ACL-BUSINESS
 class-map match-any CM-BULK-AF11
  match access-group name ACL-BULK
 !
 policy-map PM-BRANCH-CHILD
  class CM-VOICE-EF
   priority percent 15
  class CM-BUSINESS-AF31
   set dscp af31
   bandwidth percent 40
  class CM-BULK-AF11

   set dscp af11
   bandwidth percent 10
  class class-default
   fair-queue
 !
 policy-map PM-BRANCH-PARENT
  class class-default
   shape average 20000000
   service-policy PM-BRANCH-CHILD

ISP1 Golden Configuration

 hostname ISP1
 !
 interface GigabitEthernet0/0
  description CUSTOMER-BR1-20M-CIR
  ip address 192.0.2.2 255.255.255.252
  service-policy input PM-PROVIDER-IN
  no shutdown
 !
 interface Loopback40
  ip address 10.40.40.40 255.255.255.255
 interface Loopback50
  ip address 10.40.50.50 255.255.255.255
 !
 ip route 10.30.10.0 255.255.255.0 192.0.2.1
 ip route 10.30.20.0 255.255.255.0 192.0.2.1
 !
 ip access-list extended ACL-BRANCH-AGGREGATE
  permit ip 10.30.0.0 0.0.255.255 any
 !
 class-map match-any CM-BRANCH-AGGREGATE
  match access-group name ACL-BRANCH-AGGREGATE
 !
 policy-map PM-PROVIDER-IN
  class CM-BRANCH-AGGREGATE
   police cir 20000000 bc 625000
    conform-action transmit
    exceed-action drop
  class class-default
   drop

Golden-State Verification

 BR1# show policy-map interface GigabitEthernet0/1 output
 ISP1# show policy-map interface GigabitEthernet0/0 input
 BR1# show interfaces GigabitEthernet0/1 | include BW|rate
 BR1# show access-lists ACL-BUSINESS
 BR1# show access-lists ACL-BULK

Under a combined authorized load above 20 Mbps, BR1 should shape to
20 Mbps. ISP1 should see traffic predominantly conform to its policer. The
business class should receive a minimum share during congestion but
should not show a 2-Mbps policer.

3. THE SABOTAGE SCRIPT
The sabotage introduces one decimal-place error in the parent shaper and
replaces the business bandwidth guarantee with a hard policer.

Sabotage commands

 BR1# configure terminal
 !
 policy-map PM-BRANCH-CHILD-BROKEN
  class CM-VOICE-EF
   priority percent 15
  class CM-BUSINESS-AF31
   set dscp af31
   police cir 2000000
     conform-action transmit
     exceed-action drop
  class CM-BULK-AF11
   set dscp af11
   bandwidth percent 10
  class class-default
   fair-queue
 !
 policy-map PM-BRANCH-PARENT-BROKEN
  class class-default
   shape average 200000000
   service-policy PM-BRANCH-CHILD-BROKEN
 !
 interface GigabitEthernet0/1
  no service-policy output PM-BRANCH-PARENT
  service-policy output PM-BRANCH-PARENT-BROKEN
 end

 Element          Change

 Parent shaper    Changed from shape average 20000000 to shape average 200000000
                  — a decimal-place error

 Business class   Changed from bandwidth percent 40 to police cir 2000000 — a hard
 action           2-Mbps cap

 Policy map       Replaced with PM-BRANCH-PARENT-BROKEN and PM-
 names            BRANCH-CHILD-BROKEN

 Net effect       The branch now overruns the provider's 20-Mbps CIR, and business
                  traffic is capped far below its intended guarantee

4. THE VERIFICATION WORKFLOW

Step 1: Establish the Contract and Physical Rate

 BR1# show interfaces GigabitEthernet0/1
 BR1# show running-config interface GigabitEthernet0/1

The physical interface can transmit at 1 Gbps. The output shows a
reference bandwidth of 20000 Kbit/sec, but that value alone does not
enforce the provider CIR. The candidate should state this explicitly.

Step 2: Inspect the Customer Egress Policy

 BR1# show policy-map interface GigabitEthernet0/1 output

Hunt for:

    `Service-policy output: PM-BRANCH-PARENT-BROKEN`

    `shape (average) cir 200000000` or an equivalent 200-Mbps target.

    Child policy name `PM-BRANCH-CHILD-BROKEN`.

    Business class policer rate of 2,000,000 bps.

    Business conform and exceed counters.

    Parent and class offered rates during the load test.

Step 3: Locate the First Drop Point

 ISP1# show policy-map interface GigabitEthernet0/0 input

The provider policer should show significant exceed drops while the
branch sends above 20 Mbps. This proves that the first broad-loss point is
ISP1 ingress, not the BR1 child queues.

 Evidence                         Conclusion

 ISP1 ingress policer shows       The provider edge is the first broad-loss point
 significant exceed/drop counts   while BR1 shapes above the contracted rate

Step 4: Build a Corrected Parent with the Existing Child

Repair the aggregate rate first while preserving the current child policy so
the second fault can be evaluated independently.

 BR1# configure terminal
 BR1(config)# policy-map PM-BRANCH-PARENT-STAGE1
 BR1(config-pmap)# class class-default
 BR1(config-pmap-c)# shape average 20000000
 BR1(config-pmap-c)# service-policy PM-BRANCH-CHILD-BROKEN
 BR1(config-pmap-c)# exit
 BR1(config-pmap)# exit
 BR1(config)# interface GigabitEthernet0/1
 BR1(config-if)# no service-policy output PM-BRANCH-PARENT-
 BROKEN
 BR1(config-if)# service-policy output PM-BRANCH-PARENT-
 STAGE1
 BR1(config-if)# end

Step 5: Repeat the Aggregate Load Test

 BR1# show policy-map interface GigabitEthernet0/1 output
 ISP1# show policy-map interface GigabitEthernet0/0 input

The parent now targets 20 Mbps. ISP1 exceed drops should stop or fall to
negligible transient values. Broad loss improves. However, the business

application remains limited because the child policy still polices it at 2
Mbps.

Step 6: Prove the Remaining Business-Class Drop

 BR1# show policy-map interface GigabitEthernet0/1 output
 BR1# show access-lists ACL-BUSINESS

The ACL counter confirms that the HTTPS flow matches the intended
business class. Under that class, the 2-Mbps policer shows exceed drops.
The parent shaper is no longer the problem; the business class is being
intentionally discarded by the child policy.

 Evidence                              Conclusion

 ACL-BUSINESS counter increases        The HTTPS flow is classified correctly but is
 and the 2-Mbps policer shows exceed   being intentionally capped by an unnecessary
 drops                                 inner policer

Step 7: Replace the Broken Child Policy

 BR1# configure terminal
 !
 policy-map PM-BRANCH-CHILD-REPAIRED
  class CM-VOICE-EF
   priority percent 15
  class CM-BUSINESS-AF31
   set dscp af31
   bandwidth percent 40
  class CM-BULK-AF11
   set dscp af11
   bandwidth percent 10
  class class-default
   fair-queue
 !
 policy-map PM-BRANCH-PARENT-REPAIRED
  class class-default
   shape average 20000000
   service-policy PM-BRANCH-CHILD-REPAIRED
 !
 interface GigabitEthernet0/1
  no service-policy output PM-BRANCH-PARENT-STAGE1
  service-policy output PM-BRANCH-PARENT-REPAIRED
 end

Step 8: Verify Queue Behavior Under Congestion

 BR1# show policy-map interface GigabitEthernet0/1 output
 ISP1# show policy-map interface GigabitEthernet0/0 input

   Parent shaper target is 20 Mbps.

   Voice class receives priority treatment and does not show
   inappropriate drops.

   Business class shows a bandwidth guarantee rather than a police
   action.

   Business throughput can exceed 2 Mbps when capacity is available.

   Bulk and class-default absorb congestion according to the policy.
