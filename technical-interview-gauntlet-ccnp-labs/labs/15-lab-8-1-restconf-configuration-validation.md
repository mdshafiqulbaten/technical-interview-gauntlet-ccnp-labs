# Lab 8.1: RESTCONF Configuration Validation

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 15855-16254). Review against the printed book before classroom use.*

Time limit: 35 minutes

1. THE TOPOLOGY & OBJECTIVE
A small enterprise uses RESTCONF to validate a standard IOS XE router
build before adding a device to production. R1 passes the standard. R2 was
used during a migration and now has three hidden problems: its
RESTCONF service is disabled, its validation loopback contains the
wrong address and description, and its OSPF-facing interface is
administratively shut down.

                            Management VLAN 200
                       192.0.2.0/24

  Automation Host                 R1               R2
  192.0.2.100                192.0.2.11      192.0.2.12
        |                         |                |
        +-------------------------+---------------+
                                  \              /
                                   \ Gi2   Gi2 /
                                    10.12.12.0/30

 R1 Loopback100: 10.255.1.1/32
 R2 Loopback100: 10.255.2.2/32
 OSPF process 10, area 0

                                                        Gi2             Gi2
                                            R1                                R2
                                                        10.12.12.0/30

    Automation Host

                           Management VLAN 200 — 192.0.2.0/24

                 Figure: RESTCONF Validation Lab — Physical Topology

Business intent:

    RESTCONF must be reachable over HTTPS from the automation
    host.

    The RESTCONF account must be authenticated locally for this lab.

    R2 must be named `R2`.

    R2 Loopback100 must have description `AUTOMATION-
    VALIDATION`.

    R2 Loopback100 must use 10.255.2.2/32.

    R2 GigabitEthernet2 must not be configured with `shutdown`.

   The R1-R2 OSPF adjacency must reach FULL state.

   The audit must remain read-only; remediation is performed through an
   approved CLI change after the drift is proven.

Requirement                         Verification Method

RESTCONF reachable over HTTPS       TCP 443 reachability test, then a read-only
                                    RESTCONF GET

R2 named R2                         Audit hostname container against source of
                                    truth

R2 Loopback100 description and      Audit interface container against source of
address correct                     truth

R2 GigabitEthernet2 not shut down   Audit interface container's shutdown
                                    presence/absence

R1-R2 OSPF adjacency FULL           show ip ospf neighbor on R1, independent of
                                    the API result

2. THE INITIAL GOLDEN CONFIG

R1 Golden Configuration

 enable
 configure terminal
 hostname R1
 no ip domain-lookup
 !
 aaa new-model
 username apiadmin privilege 15 secret LabOnly-ChangeMe!
 ip http authentication local
 ip http secure-server
 restconf
 netconf-yang
 !
 interface GigabitEthernet1
  description MANAGEMENT
  ip address 192.0.2.11 255.255.255.0
  no shutdown
 !
 interface GigabitEthernet2
  description OSPF-TO-R2
  ip address 10.12.12.1 255.255.255.252
  no shutdown
 !
 interface Loopback100
  description AUTOMATION-VALIDATION
  ip address 10.255.1.1 255.255.255.255
 !
 router ospf 10
  router-id 10.255.1.1
  passive-interface default
  no passive-interface GigabitEthernet2
  network 10.12.12.0 0.0.0.3 area 0
  network 10.255.1.1 0.0.0.0 area 0
 end
 write memory

R2 Golden Configuration

 enable
 configure terminal
 hostname R2
 no ip domain-lookup
 !
 aaa new-model
 username apiadmin privilege 15 secret LabOnly-ChangeMe!
 ip http authentication local
 ip http secure-server
 restconf
 netconf-yang
 !
 interface GigabitEthernet1
  description MANAGEMENT
  ip address 192.0.2.12 255.255.255.0
  no shutdown
 !
 interface GigabitEthernet2
  description OSPF-TO-R1
  ip address 10.12.12.2 255.255.255.252
  no shutdown
 !
 interface Loopback100
  description AUTOMATION-VALIDATION
  ip address 10.255.2.2 255.255.255.255
 !
 router ospf 10
  router-id 10.255.2.2
  passive-interface default
  no passive-interface GigabitEthernet2
  network 10.12.12.0 0.0.0.3 area 0
  network 10.255.2.2 0.0.0.0 area 0
 end
 write memory

Automation Host Preparation

 python3 -m venv ~/gauntlet-venv
 source ~/gauntlet-venv/bin/activate
 python -m pip install --upgrade pip
 pip install requests

 export NET_USER=apiadmin
 read -s NET_PASSWORD
 export NET_PASSWORD

Install the enterprise CA certificate and use it with `verify=` in production.
The lab script allows an explicit insecure mode only so a self-signed
virtual router can be tested.

Golden RESTCONF Audit Script

 #!/usr/bin/env python3
 import argparse
 import json
 import os
 import sys
 from typing import Any

 import requests
 from requests.auth import HTTPBasicAuth

 def first_record(payload: dict[str, Any]) -> dict[str,
 Any]:
     for value in payload.values():
         if isinstance(value, list) and value:
             if isinstance(value[0], dict):
                 return value[0]
         if isinstance(value, dict):
             return value
     return {}

 def get_json(session: requests.Session, base: str,
              path: str) -> dict[str, Any]:
     response = session.get(
         f"{base}{path}",
         timeout=(5, 20),
     )
     response.raise_for_status()
     return response.json()

 def main() -> int:
     parser = argparse.ArgumentParser()
     parser.add_argument("--host", required=True)
     parser.add_argument("--insecure", action="store_true")
     args = parser.parse_args()

     username = os.environ["NET_USER"]
     password = os.environ["NET_PASSWORD"]
     base = f"https://{args.host}/restconf/data"

     session = requests.Session()
     session.auth = HTTPBasicAuth(username, password)
     session.headers.update({

        "Accept": "application/yang-data+json"
    })
    session.verify = not args.insecure

    expected = {
        "hostname": "R2",
        "loopback_description": "AUTOMATION-VALIDATION",
        "loopback_address": "10.255.2.2",
        "gi2_shutdown": False,
    }

    report: dict[str, Any] = {
        "host": args.host,
        "checks": {},
        "errors": [],
    }

    try:
        hostname_payload = get_json(
            session,
            base,
            "/Cisco-IOS-XE-native:native/hostname",
        )
        hostname = next(iter(hostname_payload.values()))

        loop_payload = get_json(
            session,
            base,
            "/Cisco-IOS-XE-
native:native/interface/Loopback=100",
        )
        loop = first_record(loop_payload)
        primary = (
            loop.get("ip", {})
                .get("address", {})
                .get("primary", {})
        )

        gi_payload = get_json(
            session,
            base,
            "/Cisco-IOS-XE-native:native/interface/"
            "GigabitEthernet=2",
        )
        gi2 = first_record(gi_payload)

        actual = {
            "hostname": hostname,
            "loopback_description":

 loop.get("description"),
             "loopback_address": primary.get("address"),
             "gi2_shutdown": "shutdown" in gi2,
         }

           for key, wanted in expected.items():
               found = actual.get(key)
               report["checks"][key] = {
                   "expected": wanted,
                   "actual": found,
                   "pass": found == wanted,
               }

     except requests.exceptions.SSLError as exc:
         report["errors"].append(f"TLS validation failed:
 {exc}")
     except requests.exceptions.ConnectTimeout:
         report["errors"].append("TCP connection timed
 out")
     except requests.exceptions.ReadTimeout:
         report["errors"].append("RESTCONF response timed
 out")
     except requests.exceptions.HTTPError as exc:
         status = exc.response.status_code
         report["errors"].append(f"HTTP {status}: {exc}")
     except (KeyError, TypeError, ValueError) as exc:
         report["errors"].append(f"Response validation
 failed: {exc}")
     except requests.exceptions.RequestException as exc:
         report["errors"].append(f"Transport failure:
 {exc}")

      passed = (
          not report["errors"]
          and all(
              item["pass"]
              for item in report["checks"].values()
          )
      )
      report["pass"] = passed
      print(json.dumps(report, indent=2, sort_keys=True))
      return 0 if passed else 2

 if __name__ == "__main__":
     sys.exit(main())

Save as `restconf_audit.py`, then make it executable:

 chmod 700 restconf_audit.py
 ./restconf_audit.py --host 192.0.2.12 --insecure

Golden-State Verification

 R1# show ip ospf neighbor
 R1# show platform software yang-management process
 R2# show ip ospf neighbor
 R2# show platform software yang-management process

Expected audit result:

 {
     "checks": {
       "gi2_shutdown": {"actual": false, "pass": true},
       "hostname": {"actual": "R2", "pass": true},
       "loopback_address": {
          "actual": "10.255.2.2",
          "pass": true
       },
       "loopback_description": {
          "actual": "AUTOMATION-VALIDATION",
          "pass": true
       }
     },
     "errors": [],
     "host": "192.0.2.12",
     "pass": true
 }

3. THE SABOTAGE SCRIPT
