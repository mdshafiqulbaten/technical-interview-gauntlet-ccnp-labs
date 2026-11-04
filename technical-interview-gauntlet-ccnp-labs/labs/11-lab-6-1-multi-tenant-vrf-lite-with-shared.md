# Lab 6.1: Multi-Tenant VRF-Lite with Shared

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 11572-11971). Review against the printed book before classroom use.*

Services
Recommended time limit: 40 minutes

1. THE TOPOLOGY & OBJECTIVE
A services router provides isolated Layer 3 gateways for two business
tenants and one shared-services network. Both tenants must reach the
shared DNS and logging server at 172.16.100.10. The tenants must never

reach each other directly. IOS XE route replication provides selective route
visibility between the VRFs.

Lab topology

 TENANT-A-CLIENT                         SHARED-SERVER
 10.10.10.10/24                         172.16.100.10/24
        | Gi0/0                              | Gi0/0
        |                                    |
        | Gi0/1 TENANT_A       Gi0/3 SHARED |
        +------------[ VRF-HUB ]-------------+
                     Gi0/2 TENANT_B
                            |
                            | Gi0/0
                     TENANT-B-CLIENT
                     10.20.20.10/24

      TENANT-A-CLIENT                                                         SHARED-SERVER

         10.10.10.10/24                                                            172.16.100.10/24
                          Gi0/0                                               Gi0/0

                                  Gi0/1 (TENANT_A)           Gi0/3 (SHARED)

                                           VRF-HUB

                                                     Gi0/2 (TENANT_B)

                                                     Gi0/0

                                      TENANT-B-CLIENT

                                          10.20.20.10/24

     Figure: Lab 6.1: Multi-Tenant VRF-Lite with Shared Services — Physical Topology

 Device/Interface                      VRF                                Address

 VRF-HUB Gi0/1                         TENANT_A                           10.10.10.1/24

 VRF-HUB Gi0/2                         TENANT_B                           10.20.20.1/24

 VRF-HUB Gi0/3                         SHARED                             172.16.100.1/24

 TENANT-A-CLIENT                       n/a (host)                         10.10.10.10/24

 TENANT-B-CLIENT                       n/a (host)                         10.20.20.10/24

 SHARED-SERVER                         n/a (host)                         172.16.100.10/24

   Tenant A must reach 172.16.100.10.

   Tenant B must reach 172.16.100.10.

   The shared server must return traffic to both tenant subnets.

   Tenant A must not receive a route to 10.20.20.0/24.

   Tenant B must not receive a route to 10.10.10.0/24.

   Only the three specifically approved connected subnets may be
   replicated.

2. THE INITIAL GOLDEN CONFIG
VRF-HUB golden configuration

enable
configure terminal
hostname VRF-HUB
no ip domain-lookup
!
vrf definition TENANT_A
 description Finance tenant
 rd 65000:10
 address-family ipv4
 exit-address-family
!
vrf definition TENANT_B
 description Engineering tenant
 rd 65000:20
 address-family ipv4
 exit-address-family
!
vrf definition SHARED
 description Shared infrastructure services
 rd 65000:100
 address-family ipv4
 exit-address-family
!
interface GigabitEthernet0/1
 description TENANT-A-LAN
 vrf forwarding TENANT_A
 ip address 10.10.10.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/2
 description TENANT-B-LAN
 vrf forwarding TENANT_B
 ip address 10.20.20.1 255.255.255.0
 no shutdown
!
interface GigabitEthernet0/3
 description SHARED-SERVICES-LAN
 vrf forwarding SHARED
 ip address 172.16.100.1 255.255.255.0
 no shutdown
!
ip prefix-list PL-TENANT-A seq 5 permit 10.10.10.0/24
ip prefix-list PL-TENANT-B seq 5 permit 10.20.20.0/24
ip prefix-list PL-SHARED seq 5 permit 172.16.100.0/24
!
route-map RM-TENANT-A permit 10
 match ip address prefix-list PL-TENANT-A
!

 route-map RM-TENANT-B permit 10
  match ip address prefix-list PL-TENANT-B
 !
 route-map RM-SHARED permit 10
  match ip address prefix-list PL-SHARED
 !
 vrf definition TENANT_A
  address-family ipv4
   route-replicate from vrf SHARED unicast connected route-
 map RM-SHARED
  exit-address-family
 !
 vrf definition TENANT_B
  address-family ipv4
   route-replicate from vrf SHARED unicast connected route-
 map RM-SHARED
  exit-address-family
 !
 vrf definition SHARED
  address-family ipv4
   route-replicate from vrf TENANT_A unicast connected
 route-map RM-TENANT-A
   route-replicate from vrf TENANT_B unicast connected
 route-map RM-TENANT-B
  exit-address-family
 end
 write memory

TENANT-A-CLIENT

 enable
 configure terminal
 hostname TENANT-A-CLIENT
 interface GigabitEthernet0/0
  ip address 10.10.10.10 255.255.255.0
  no shutdown
 ip route 0.0.0.0 0.0.0.0 10.10.10.1
 end

TENANT-B-CLIENT

 enable
 configure terminal
 hostname TENANT-B-CLIENT
 interface GigabitEthernet0/0
  ip address 10.20.20.10 255.255.255.0
  no shutdown
 ip route 0.0.0.0 0.0.0.0 10.20.20.1
 end

SHARED-SERVER

 enable
 configure terminal
 hostname SHARED-SERVER
 interface GigabitEthernet0/0
  ip address 172.16.100.10 255.255.255.0
  no shutdown
 ip route 0.0.0.0 0.0.0.0 172.16.100.1
 end

Golden-state verification

 VRF-HUB# show vrf
 VRF-HUB# show ip route vrf TENANT_A
 VRF-HUB# show ip route vrf TENANT_B
 VRF-HUB# show ip route vrf SHARED
 VRF-HUB# ping vrf TENANT_A 172.16.100.10 source 10.10.10.1
 VRF-HUB# ping vrf TENANT_B 172.16.100.10 source 10.20.20.1

In the VRF routing tables, replicated routes should be identified by a plus
sign on supporting IOS XE releases. The SHARED table should contain
only the two tenant connected subnets, and each tenant table should
contain only the shared-services subnet in addition to its native connected
route.

3. THE SABOTAGE SCRIPT
The interviewer injects two independent policy errors. The first removes
the return path for Tenant A. The second creates an unauthorized cross-
tenant leak.

Sabotage commands

 VRF-HUB# configure terminal
 VRF-HUB(config)# vrf definition SHARED
 VRF-HUB(config-vrf)# address-family ipv4
 VRF-HUB(config-vrf-af)# no route-replicate from vrf
 TENANT_A unicast connected route-map RM-TENANT-A
 VRF-HUB(config-vrf-af)# exit-address-family
 VRF-HUB(config-vrf)# exit
 !
 VRF-HUB(config)# vrf definition TENANT_A
 VRF-HUB(config-vrf)# address-family ipv4
 VRF-HUB(config-vrf-af)# route-replicate from vrf TENANT_B
 unicast connected route-map RM-TENANT-B
 VRF-HUB(config-vrf-af)# end

    TENANT_A still has a route to the shared subnet, so the forward
    packet reaches SHARED.

    SHARED has no route to 10.10.10.0/24, so the reply follows no valid
    return path.

    TENANT_A now contains 10.20.20.0/24, violating tenant isolation.

    TENANT_B shared-services access may continue to work, creating
    misleading partial success.

    All three physical interfaces remain up and connected.

4. THE VERIFICATION WORKFLOW

Step 1: Establish the VRF and Interface Inventory

 VRF-HUB# show vrf
 VRF-HUB# show ip interface brief vrf all
 VRF-HUB# show running-config | section vrf definition

Confirm that Gi0/1 belongs to TENANT_A, Gi0/2 belongs to
TENANT_B, and Gi0/3 belongs to SHARED. A correct interface
assignment does not prove that route leaking is correct, but it establishes
the forwarding contexts.

Step 2: Test the Native Connected Gateways

 TENANT-A-CLIENT# ping 10.10.10.1
 TENANT-B-CLIENT# ping 10.20.20.1
 SHARED-SERVER# ping 172.16.100.1

All three should succeed. This rules out basic addressing, interface state,
and local Layer 2 reachability.

Step 3: Compare Tenant Outcomes

 TENANT-A-CLIENT# ping 172.16.100.10
 TENANT-B-CLIENT# ping 172.16.100.10

Tenant A fails while Tenant B succeeds. This narrows the fault to tenant-
specific routing or policy rather than a general shared-server outage.

Step 4: Inspect the Forward Route in TENANT_A

 VRF-HUB# show ip route vrf TENANT_A 172.16.100.0
 VRF-HUB# show ip cef vrf TENANT_A 172.16.100.10 detail
 VRF-HUB# ping vrf TENANT_A 172.16.100.10 source 10.10.10.1

The shared subnet is present in TENANT_A and resolves toward Gi0/3 in
the SHARED context through route replication. A router-generated ping
sourced from 10.10.10.1 still fails because the reply cannot return.

Step 5: Prove the Missing Return Route

 VRF-HUB# show ip route vrf SHARED 10.10.10.0
 VRF-HUB# show ip cef vrf SHARED 10.10.10.10 detail
 VRF-HUB# show ip route vrf SHARED 10.20.20.0

The SHARED table has a route to Tenant B but no route to Tenant A. That
is the decisive evidence for a one-direction leak.

 Routing Table   Route to Tenant B           Route to Tenant A
                 (10.20.20.0/24)             (10.10.10.0/24)

 SHARED          Present                     Missing — this is the fault
 VRF

Step 6: Inspect the Route-Replication Configuration

 VRF-HUB# show running-config | section ^vrf definition
 VRF-HUB# show route-map RM-TENANT-A
 VRF-HUB# show ip prefix-list PL-TENANT-A

Confirm that the prefix list and route map exist, but the SHARED VRF no
longer contains the `route-replicate from vrf TENANT_A` command.

Step 7: Restore the Approved Return Leak

 VRF-HUB# configure terminal
 VRF-HUB(config)# vrf definition SHARED
 VRF-HUB(config-vrf)# address-family ipv4
 VRF-HUB(config-vrf-af)# route-replicate from vrf TENANT_A
 unicast connected route-map RM-TENANT-A
 VRF-HUB(config-vrf-af)# end

Step 8: Verify Tenant A Shared-Service Reachability

 VRF-HUB# show ip route vrf SHARED 10.10.10.0
 VRF-HUB# ping vrf TENANT_A 172.16.100.10 source 10.10.10.1
 TENANT-A-CLIENT# ping 172.16.100.10

The test should now succeed. Do not stop. The lab contains a second
policy violation that does not prevent the shared-service test.

Step 9: Audit the Complete TENANT_A Table

 VRF-HUB# show ip route vrf TENANT_A
 VRF-HUB# show ip route vrf TENANT_A 10.20.20.0
 TENANT-A-CLIENT# ping 10.20.20.10

The unexpected 10.20.20.0/24 route proves that TENANT_A imported a
