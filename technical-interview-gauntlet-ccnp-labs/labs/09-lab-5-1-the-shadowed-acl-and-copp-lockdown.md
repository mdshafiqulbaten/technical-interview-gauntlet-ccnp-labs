# Lab 5.1: The Shadowed ACL and CoPP Lockdown

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 9568-9967). Review against the printed book before classroom use.*

Suggested interview time limit: 40 minutes

1. THE TOPOLOGY & OBJECTIVE
EDGE protects a small public DMZ and provides remote management
through a loopback address. A management workstation must be able to
SSH to EDGE. An external test host must reach HTTPS on the public
server but must not reach SSH on that server. An OSPF adjacency to
CORE must remain stable while CoPP protects the device CPU.

Lab 5.1 topology

                          10.0.0.0/30
                    CORE Gi0/0 -------- Gi0/2 EDGE
                  10.0.0.2                 10.0.0.1
                                              |
 MGMT-PC 10.10.50.50 ---- Gi0/0 10.10.50.1 | Loopback0
 10.255.0.1
                                              |
 OUTSIDE 198.51.100.50 --- Gi0/1 198.51.100.2|--- DMZ Gi0/3
 10.20.20.1
                                                    |
                                              WEB-SRV
 10.20.20.10

                                                                               CORE

                                                                          Gi0/0

                                                  Gi0/2

                                Gi0/0                     Loopback0
    MGMT-PC                                EDGE           10.255.0.1
                  10.10.50.50

                                   Gi0/1              Gi0/3

              198.51.100.50                                      10.20.20.10

    OUTSIDE                                                              WEB-SRV

     Figure: Lab 5.1: The Shadowed ACL and CoPP Lockdown — Physical Topology

 Device    Interface      Address             Role

 EDGE      Gi0/0          10.10.50.1/24       Management LAN

 EDGE      Gi0/1          198.51.100.2/24     Outside/public interface

 EDGE      Gi0/2          10.0.0.1/30         OSPF link to CORE

 EDGE      Gi0/3          10.20.20.1/24       Public DMZ

 EDGE      Loopback0      10.255.0.1/32       Management endpoint

 CORE      Gi0/0          10.0.0.2/30         OSPF link to EDGE

The lab assumes console access to EDGE. This is intentional. A candidate
should never modify a suspected management ACL or CoPP policy over
the only affected remote session without a recovery path.

2. THE INITIAL GOLDEN CONFIG

EDGE Golden Configuration

 enable
 configure terminal
 hostname EDGE
 no ip domain-lookup
 !
 interface GigabitEthernet0/0
  description MANAGEMENT-LAN
  ip address 10.10.50.1 255.255.255.0
  no shutdown
 !
 interface GigabitEthernet0/1
  description OUTSIDE-NETWORK
  ip address 198.51.100.2 255.255.255.0
  ip access-group OUTSIDE-IN in
  no shutdown
 !
 interface GigabitEthernet0/2
  description OSPF-TO-CORE
  ip address 10.0.0.1 255.255.255.252
  ip ospf network point-to-point
  no shutdown
 !
 interface GigabitEthernet0/3
  description PUBLIC-DMZ
  ip address 10.20.20.1 255.255.255.0
  no shutdown
 !
 interface Loopback0
  description MANAGEMENT-ENDPOINT
  ip address 10.255.0.1 255.255.255.255
 !
 router ospf 10
  router-id 10.255.0.1
  passive-interface default
  no passive-interface GigabitEthernet0/2
  network 10.0.0.0 0.0.0.3 area 0
  network 10.255.0.1 0.0.0.0 area 0
  network 10.20.20.0 0.0.0.255 area 0
 !
 ip access-list extended OUTSIDE-IN
  10 remark Permit public HTTPS
  10 permit tcp any host 10.20.20.10 eq 443
  20 remark Deny public SSH to the server

 20 deny tcp any host 10.20.20.10 eq 22 log
 30 permit icmp any host 198.51.100.2 echo
 90 deny ip any any log
!
ip access-list standard MGMT-VTY
 10 permit 10.10.50.0 0.0.0.255
 90 deny any log
!
ip access-list extended COPP-ROUTING
 10 permit ospf any any
!
ip access-list extended COPP-MANAGEMENT
 10 permit tcp 10.10.50.0 0.0.0.255 host 10.255.0.1 eq 22
 20 permit icmp 10.10.50.0 0.0.0.255 host 10.255.0.1 echo
!
class-map match-any CM-COPP-ROUTING
 match access-group name COPP-ROUTING
!
class-map match-any CM-COPP-MANAGEMENT
 match access-group name COPP-MANAGEMENT
!
policy-map PM-CONTROL-PLANE
 class CM-COPP-ROUTING
  police 256000 conform-action transmit exceed-action drop
 class CM-COPP-MANAGEMENT
  police 512000 conform-action transmit exceed-action drop
 class class-default
  police 1000000 conform-action transmit exceed-action
drop
!
control-plane
 service-policy input PM-CONTROL-PLANE
!
ip domain name lab.example
username breakglass privilege 15 secret LabOnly-ReplaceMe
crypto key generate rsa modulus 2048
ip ssh version 2
!
line vty 0 4
 transport input ssh
 access-class MGMT-VTY in
 login local
 exec-timeout 10 0
!
end
write memory

CORE Golden Configuration

 enable
 configure terminal
 hostname CORE
 no ip domain-lookup
 !
 interface GigabitEthernet0/0
  description OSPF-TO-EDGE
  ip address 10.0.0.2 255.255.255.252
  ip ospf network point-to-point
  no shutdown
 !
 interface Loopback0
  ip address 10.255.0.2 255.255.255.255
 !
 router ospf 10
  router-id 10.255.0.2
  network 10.0.0.0 0.0.0.3 area 0
  network 10.255.0.2 0.0.0.0 area 0
 !
 end
 write memory

OUTSIDE Test Host Configuration

 hostname OUTSIDE
 interface GigabitEthernet0/0
  ip address 198.51.100.50 255.255.255.0
  no shutdown
 ip route 10.20.20.0 255.255.255.0 198.51.100.2

MGMT-PC Test Host Configuration

 hostname MGMT-PC
 interface GigabitEthernet0/0
  ip address 10.10.50.50 255.255.255.0
  no shutdown
 ip route 10.255.0.1 255.255.255.255 10.10.50.1

WEB-SRV Test Host Configuration

 hostname WEB-SRV
 interface GigabitEthernet0/0
  ip address 10.20.20.10 255.255.255.0
  no shutdown
 ip route 0.0.0.0 0.0.0.0 10.20.20.1
 !
 ip http secure-server
 ip ssh version 2

If the selected virtual image does not provide application services, use TCP
probes supported by the platform or replace WEB-SRV with a Linux host
running HTTPS and SSH. The ACL logic remains identical.

Golden-State Verification

 EDGE# show ip ospf neighbor
 EDGE# show ip access-lists OUTSIDE-IN
 EDGE# show policy-map control-plane input
 EDGE# show users
 MGMT-PC# ssh -l breakglass 10.255.0.1
 OUTSIDE# telnet 10.20.20.10 443
 OUTSIDE# telnet 10.20.20.10 22

The HTTPS probe should succeed, the SSH probe to WEB-SRV should
fail, management SSH to EDGE should succeed, and OSPF should remain
FULL.

3. THE SABOTAGE SCRIPT

Sabotage Fault 1: Shadow the Security Entries

 EDGE# configure terminal
 EDGE(config)# ip access-list extended OUTSIDE-IN
 EDGE(config-ext-nacl)# 5 permit ip any 10.20.20.0
 0.0.0.255
 EDGE(config-ext-nacl)# end

The broad sequence 5 permit makes both the SSH deny and HTTPS permit
unreachable for traffic to the DMZ subnet.

Sabotage Fault 2: Overpolice the Management Class

 EDGE# configure terminal
 EDGE(config)# ip access-list extended COPP-MANAGEMENT
 EDGE(config-ext-nacl)# no 10
 EDGE(config-ext-nacl)# no 20
 EDGE(config-ext-nacl)# 10 permit ip 10.10.50.0 0.0.0.255
 host 10.255.0.1
 EDGE(config-ext-nacl)# exit
 EDGE(config)# policy-map PM-CONTROL-PLANE
 EDGE(config-pmap)# class CM-COPP-MANAGEMENT
 EDGE(config-pmap-c)# police 8000 conform-action transmit
 exceed-action drop
 EDGE(config-pmap-c)# end

The class now includes every IPv4 packet from the management subnet to
the loopback and allows only a very small laboratory rate. Repeated pings
can consume the class allowance and cause SSH to become slow or fail.

Generate the Symptom

 MGMT-PC# ping 10.255.0.1 repeat 500 size 1400 timeout 1
 MGMT-PC# ssh -l breakglass 10.255.0.1
 OUTSIDE# telnet 10.20.20.10 22
 OUTSIDE# telnet 10.20.20.10 443

Exact traffic generation commands vary by image. The goal is controlled
legitimate load, not an uncontrolled flood.

4. THE VERIFICATION WORKFLOW

Step 1: Restate the Positive and Negative Tests

 Test                                           Expected Result

 OUTSIDE to WEB-SRV HTTPS (TCP/443)             Permitted

 OUTSIDE to WEB-SRV SSH (TCP/22)                Denied

 MGMT-PC to EDGE SSH (loopback)                 Permitted

 EDGE-to-CORE OSPF adjacency                    Remains FULL throughout

This prevents the candidate from declaring success after making
everything reachable or after blocking all traffic.

Step 2: Confirm the Security Enforcement Points

 EDGE# show ip interface GigabitEthernet0/1
 EDGE# show running-config interface GigabitEthernet0/1
 EDGE# show running-config | section line vty
 EDGE# show policy-map control-plane input

The outside ACL is inbound on Gi0/1. The VTY access class limits
terminal sources. CoPP applies to traffic destined to the control plane.
These are separate controls.

Step 3: Reproduce the Outside Policy Failure

 EDGE# clear access-list counters OUTSIDE-IN
 OUTSIDE# telnet 10.20.20.10 22
 OUTSIDE# telnet 10.20.20.10 443
 EDGE# show ip access-lists OUTSIDE-IN

Sequence 5 increases for both probes. Sequence 20, the SSH deny, remains
at zero. This proves the packet reached the ACL and matched an earlier
entry.

Step 4: Identify the Shadowed Entries

 EDGE# show ip access-lists OUTSIDE-IN
 EDGE# show running-config | section ip access-list
 extended OUTSIDE-IN

Read the ACL in sequence order. Sequence 5 permits all IP traffic from
any source to the entire DMZ. Therefore, the later TCP/22 deny and
TCP/443 permit cannot be reached.

 Sequence   Entry                            Counter Behavior Observed

 5          permit ip any 10.20.20.0         Increments on both the HTTPS and SSH
            0.0.0.255                        probes

 20         deny tcp any host 10.20.20.10    Remains at zero; unreachable because of
            eq 22 log                        sequence 5

 30         permit tcp any host              Remains at zero; also unreachable
            10.20.20.10 eq 443               because of sequence 5

Step 5: Restore Least-Privilege Ordering
