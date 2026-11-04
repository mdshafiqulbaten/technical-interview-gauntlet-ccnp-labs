# Lab 6.2: Secure GRE over IPsec with an MTU

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 12086-12485). Review against the printed book before classroom use.*

Black Hole
Recommended time limit: 50 minutes

1. THE TOPOLOGY & OBJECTIVE
Two sites exchange private routes through an OSPF adjacency over a GRE
tunnel. The GRE packets must be protected by IKEv2 and IPsec. The
provider path has a 1476-byte IP MTU. The final design must support
routing, encryption, and reliable TCP applications without fragmentation-
related black holes.

Lab topology

 10.10.10.0/24
 10.20.20.0/24
 SITE-A Gi0/1
 Gi0/1 SITE-B
 10.10.10.1
 10.20.20.1
       |
 |
  [ SITE-A ] Gi0/0----198.51.100.0/30----[ ISP ]---
 -203.0.113.0/30----Gi0/0 [ SITE-B ]
       | 198.51.100.1                  MTU 1476
 203.0.113.2 |
       +================ GRE over IPsec Tunnel0
 ========================+
                        172.16.0.1/30     172.16.0.2/30

    Gi0/1 LAN 10.10.10.0/24                                 MTU 1476                                    Gi0/1 LAN 10.20.20.0/24

                              Gi0/0          Gi0/0                            Gi0/1             Gi0/0
          SITE-A                                               ISP                                            SITE-B
                               198.51.100.0/30                                 203.0.113.0/30

                                      GRE over IPsec Tunnel0: 172.16.0.1/30 <-> 172.16.0.2/30

   Figure: Lab 6.2: Secure GRE over IPsec with an MTU Black Hole — Physical Topology

 Device/Interface                 Address                        Role

 SITE-A Gi0/0                     198.51.100.1/30                Public underlay to ISP

 SITE-A Gi0/1                     10.10.10.1/24                  Site A LAN

 SITE-A Tunnel0                   172.16.0.1/30                  Protected GRE overlay

 ISP Gi0/0                        198.51.100.2/30                Provider link to Site A, 1476-byte MTU

 ISP Gi0/1                        203.0.113.1/30                 Provider link to Site B, 1476-byte MTU

 SITE-B Gi0/0                     203.0.113.2/30                 Public underlay to ISP

 SITE-B Gi0/1                     10.20.20.1/24                  Site B LAN

 SITE-B Tunnel0                   172.16.0.2/30                  Protected GRE overlay

   The public tunnel endpoints must resolve only through the ISP
   underlay.

   IKEv2 and IPsec SAs must form with AES-256 and SHA-256-based
   integrity protection.

   OSPF must become FULL across Tunnel0.

   10.10.10.0/24 and 10.20.20.0/24 must be learned through OSPF.

   The tunnel IP MTU must be 1380 and the TCP MSS must be adjusted
   to 1340 for this measured lab path.

   Small and large application tests must succeed after repair.

2. THE INITIAL GOLDEN CONFIG
ISP golden configuration

 enable
 configure terminal
 hostname ISP
 no ip domain-lookup
 interface GigabitEthernet0/0
  description TO-SITE-A
  ip address 198.51.100.2 255.255.255.252
  ip mtu 1476
  no shutdown
 interface GigabitEthernet0/1
  description TO-SITE-B
  ip address 203.0.113.1 255.255.255.252
  ip mtu 1476
  no shutdown
 end

SITE-A golden configuration

enable
configure terminal
hostname SITE-A
no ip domain-lookup
!
interface GigabitEthernet0/0
 description PUBLIC-UNDERLAY-TO-ISP
 ip address 198.51.100.1 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description SITE-A-LAN
 ip address 10.10.10.1 255.255.255.0
 no shutdown
!
ip route 203.0.113.2 255.255.255.255 198.51.100.2
!
crypto ikev2 proposal IKEV2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROPOSAL
!
crypto ikev2 keyring IKEV2-KEYRING
 peer SITE-B
  address 203.0.113.2
  pre-shared-key local Gauntlet-VPN-Key
  pre-shared-key remote Gauntlet-VPN-Key
!
crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 203.0.113.2 255.255.255.255
 identity local address 198.51.100.1
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING
!
crypto ipsec transform-set GRE-TS esp-aes 256 esp-sha256-
hmac
 mode transport
!
crypto ipsec profile GRE-IPSEC-PROFILE
 set transform-set GRE-TS
 set ikev2-profile IKEV2-PROFILE
!
interface Tunnel0
 description PROTECTED-GRE-TO-SITE-B

  ip address 172.16.0.1 255.255.255.252
  ip mtu 1380
  ip tcp adjust-mss 1340
  ip ospf network point-to-point
  tunnel source GigabitEthernet0/0
  tunnel destination 203.0.113.2
  tunnel mode gre ip
  tunnel protection ipsec profile GRE-IPSEC-PROFILE
  no shutdown
 !
 router ospf 10
  router-id 10.255.0.1
  network 10.10.10.0 0.0.0.255 area 0
  network 172.16.0.0 0.0.0.3 area 0
  passive-interface GigabitEthernet0/1
 end
 write memory

SITE-B golden configuration

enable
configure terminal
hostname SITE-B
no ip domain-lookup
!
interface GigabitEthernet0/0
 description PUBLIC-UNDERLAY-TO-ISP
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description SITE-B-LAN
 ip address 10.20.20.1 255.255.255.0
 no shutdown
!
ip route 198.51.100.1 255.255.255.255 203.0.113.1
!
crypto ikev2 proposal IKEV2-PROPOSAL
 encryption aes-cbc-256
 integrity sha256
 group 14
!
crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROPOSAL
!
crypto ikev2 keyring IKEV2-KEYRING
 peer SITE-A
  address 198.51.100.1
  pre-shared-key local Gauntlet-VPN-Key
  pre-shared-key remote Gauntlet-VPN-Key
!
crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 198.51.100.1
255.255.255.255
 identity local address 203.0.113.2
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING
!
crypto ipsec transform-set GRE-TS esp-aes 256 esp-sha256-
hmac
 mode transport
!
crypto ipsec profile GRE-IPSEC-PROFILE
 set transform-set GRE-TS
 set ikev2-profile IKEV2-PROFILE
!
interface Tunnel0

  description PROTECTED-GRE-TO-SITE-A
  ip address 172.16.0.2 255.255.255.252
  ip mtu 1380
  ip tcp adjust-mss 1340
  ip ospf network point-to-point
  tunnel source GigabitEthernet0/0
  tunnel destination 198.51.100.1
  tunnel mode gre ip
  tunnel protection ipsec profile GRE-IPSEC-PROFILE
  no shutdown
 !
 router ospf 10
  router-id 10.255.0.2
  network 10.20.20.0 0.0.0.255 area 0
  network 172.16.0.0 0.0.0.3 area 0
  passive-interface GigabitEthernet0/1
 end
 write memory

Golden-state verification

 SITE-A# ping 203.0.113.2 source 198.51.100.1
 SITE-A# show crypto ikev2 sa
 SITE-A# show crypto ipsec sa
 SITE-A# show ip ospf neighbor
 SITE-A# show ip route 10.20.20.0
 SITE-A# ping 10.20.20.1 source 10.10.10.1 size 1360 df-bit
 repeat 3

3. THE SABOTAGE SCRIPT
The first fault breaks IKEv2 authentication. After it is repaired, OSPF and
small pings recover, but the second fault leaves a packet-size black hole.

Sabotage commands

 SITE-B# configure terminal
 SITE-B(config)# crypto ikev2 keyring IKEV2-KEYRING
 SITE-B(config-ikev2-keyring)# peer SITE-A
 SITE-B(config-ikev2-keyring-peer)# pre-shared-key local
 Wrong-Gauntlet-Key
 SITE-B(config-ikev2-keyring-peer)# pre-shared-key remote
 Wrong-Gauntlet-Key
 SITE-B(config-ikev2-keyring-peer)# exit
 SITE-B(config-ikev2-keyring)# exit
 !
 SITE-B(config)# interface Tunnel0
 SITE-B(config-if)# no ip mtu 1380
 SITE-B(config-if)# no ip tcp adjust-mss 1340
 SITE-B(config-if)# end
 !
 SITE-A# configure terminal
 SITE-A(config)# interface Tunnel0
 SITE-A(config-if)# no ip mtu 1380
 SITE-A(config-if)# no ip tcp adjust-mss 1340
 SITE-A(config-if)# end

    The public tunnel endpoints remain reachable through the ISP.

    IKEv2 authentication fails, so no working IPsec data SA forms.

    After the key is corrected, OSPF reaches FULL and remote routes
    appear.

    Small pings succeed after crypto repair.

    Large DF packets and some TCP applications fail because the tunnel
    and endpoints overestimate the usable packet size.

    The provider interfaces remain limited to a 1476-byte IP MTU.

4. THE VERIFICATION WORKFLOW

Step 1: Draw the Layer Stack

State the layers before running commands: global underlay, GRE overlay,
IPsec protection, OSPF routing, and application packet size. This prevents
a successful underlay ping from being mistaken for complete recovery.

Step 2: Verify Underlay Endpoint Reachability

 SITE-A# show ip route 203.0.113.2
 SITE-A# show ip cef 203.0.113.2 detail
 SITE-A# ping 203.0.113.2 source 198.51.100.1
 SITE-B# show ip route 198.51.100.1
 SITE-B# ping 198.51.100.1 source 203.0.113.2

The host routes must resolve through the ISP-facing interfaces, not
Tunnel0. Successful pings prove basic public reachability but not GRE or
IPsec permission.

Step 3: Inspect Tunnel and OSPF State

 SITE-A# show interfaces Tunnel0
 SITE-A# show ip ospf neighbor
 SITE-A# show ip route 10.20.20.0
 SITE-A# show logging | include TUN|OSPF|CRYPTO

The tunnel may display up/up on some platforms because the source is
active and the destination is routable, yet OSPF has no FULL neighbor and
the remote LAN route is absent. This directs the investigation toward GRE
delivery or tunnel protection.

Step 4: Inspect IKEv2 and IPsec

 SITE-A# show crypto ikev2 sa detailed
 SITE-A# show crypto session detail
 SITE-A# show crypto ipsec sa
 SITE-B# show crypto ikev2 sa detailed
 SITE-B# show logging | include IKEV2|AUTH|CRYPTO

No authenticated IKEv2 SA or repeated authentication-failure logs indicate
a key or identity problem. Compare the peer address, local identity, remote
match, authentication method, and keyring on both sides.

Step 5: Compare Keyring and Profile Configuration

 SITE-A# show running-config | section crypto ikev2 keyring
 SITE-A# show running-config | section crypto ikev2 profile
 SITE-B# show running-config | section crypto ikev2 keyring
 SITE-B# show running-config | section crypto ikev2 profile

 Item Compared            SITE-A                    SITE-B

 Pre-shared key           Gauntlet-VPN-Key          Wrong-Gauntlet-Key —
 (local/remote)                                     mismatch found here

 IKEv2 profile identity   Matches SITE-B's public   Matches SITE-A's public address
 match                    address

Step 6: Restore the Matching Pre-Shared Key
