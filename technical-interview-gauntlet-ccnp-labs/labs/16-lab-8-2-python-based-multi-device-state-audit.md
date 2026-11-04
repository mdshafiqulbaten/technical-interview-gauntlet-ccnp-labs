# Lab 8.2: Python-Based Multi-Device State Audit

*Auto-extracted from "The Technical Interview Gauntlet (CCNP Enterprise Troubleshooting Labs)" (source lines 16516-16915). Review against the printed book before classroom use.*

Time limit: 45 minutes

1. THE TOPOLOGY & OBJECTIVE
An operations team must validate three IOS XE routers before a
maintenance handoff. The standard requires the same NTP server, the same
Syslog collector, and the expected OSPF neighbor count on each device.
Manual review is too slow and has already missed drift.

 Management VLAN 200: 192.0.2.0/24
 Automation Host: 192.0.2.100

 R1 Gi2 -------- R2 Gi2
 10.12.12.0/30
                   R2 Gi3 -------- R3 Gi2
                   10.23.23.0/30

 R1 management: 192.0.2.11                                 Expected OSPF neighbors: 1
 R2 management: 192.0.2.12                                 Expected OSPF neighbors: 2
 R3 management: 192.0.2.13                                 Expected OSPF neighbors: 1

 Approved NTP server:    192.0.2.50
 Approved Syslog server: 192.0.2.60

                      Gi2               Gi2                      Gi3                Gi2
          R1                                         R2                                   R3
                      10.12.12.0/30                              10.23.23.0/30

                                      Management VLAN 200 — 192.0.2.0/24

    Automation Host                     NTP .50                            Syslog .60

        Figure: Multi-Device NETCONF/RESTCONF Audit Lab — Physical Topology

Business intent:

    All devices must be reachable through SSH.

    The automation account must authenticate without embedding its
    password in source code or inventory.

    Every device must use NTP server 192.0.2.50.

    Every device must use Syslog server 192.0.2.60.

    R1 must have one FULL OSPF neighbor.

    R2 must have two FULL OSPF neighbors.

    R3 must have one FULL OSPF neighbor.

   The script must generate human-readable output, JSON output, and a
   nonzero exit code when any required check fails.

   One device failure must not prevent the remaining devices from being
   audited.

2. THE INITIAL GOLDEN CONFIG

Common Management and Services Configuration

 aaa new-model
 username automation privilege 15 secret LabOnly-ChangeMe!
 ip domain name gauntlet.lab
 crypto key generate rsa modulus 2048
 ip ssh version 2
 !
 line vty 0 4
  login local
  transport input ssh
 !
 ntp server 192.0.2.50
 logging host 192.0.2.60
 logging trap informational

R1 Golden Configuration

 configure terminal
 hostname R1
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
 interface Loopback0
  ip address 10.255.1.1 255.255.255.255
 !
 router ospf 10
  router-id 10.255.1.1
  passive-interface default
  no passive-interface GigabitEthernet2
  network 10.12.12.0 0.0.0.3 area 0
  network 10.255.1.1 0.0.0.0 area 0
 end

R2 Golden Configuration

 configure terminal
 hostname R2
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
 interface GigabitEthernet3
  description OSPF-TO-R3
  ip address 10.23.23.1 255.255.255.252
  no shutdown
 !
 interface Loopback0
  ip address 10.255.2.2 255.255.255.255
 !
 router ospf 10
  router-id 10.255.2.2
  passive-interface default
  no passive-interface GigabitEthernet2
  no passive-interface GigabitEthernet3
  network 10.12.12.0 0.0.0.3 area 0
  network 10.23.23.0 0.0.0.3 area 0
  network 10.255.2.2 0.0.0.0 area 0
 end

R3 Golden Configuration

 configure terminal
 hostname R3
 !
 interface GigabitEthernet1
  description MANAGEMENT
  ip address 192.0.2.13 255.255.255.0
  no shutdown
 !
 interface GigabitEthernet2
  description OSPF-TO-R2
  ip address 10.23.23.2 255.255.255.252
  no shutdown
 !
 interface Loopback0
  ip address 10.255.3.3 255.255.255.255
 !
 router ospf 10
  router-id 10.255.3.3
  passive-interface default
  no passive-interface GigabitEthernet2
  network 10.23.23.0 0.0.0.3 area 0
  network 10.255.3.3 0.0.0.0 area 0
 end

Apply the common management and services configuration to R1, R2, and
R3 before adding each router-specific block.

Automation Host Preparation

 python3 -m venv ~/gauntlet-netmiko
 source ~/gauntlet-netmiko/bin/activate
 python -m pip install --upgrade pip
 pip install netmiko

 export NET_USERNAME=automation
 read -s NET_PASSWORD
 export NET_PASSWORD

Inventory File
Save as inventory.json

[
    {
      "name": "R1",
      "host": "192.0.2.11",
      "expected_neighbors": 1
    },
    {
      "name": "R2",
      "host": "192.0.2.12",
      "expected_neighbors": 2
    },
    {
      "name": "R3",
      "host": "192.0.2.13",
      "expected_neighbors": 1
    }
]

Multi-Device Audit Script

 #!/usr/bin/env python3
 import json
 import os
 import re
 import sys
 from concurrent.futures import ThreadPoolExecutor,
 as_completed
 from pathlib import Path
 from typing import Any

 from netmiko import ConnectHandler
 from netmiko.exceptions import (
     NetmikoAuthenticationException,
     NetmikoTimeoutException,
 )

 APPROVED_NTP = "192.0.2.50"
 APPROVED_SYSLOG = "192.0.2.60"
 MAX_WORKERS = 3

 def audit_device(device: dict[str, Any], username: str,
                  password: str) -> dict[str, Any]:
     result: dict[str, Any] = {
         "name": device["name"],
         "host": device["host"],
         "reachable": False,
         "checks": {},
         "errors": [],
     }

     params = {
         "device_type": "cisco_ios",
         "host": device["host"],
         "username": username,
         "password": password,
         "conn_timeout": 8,
         "auth_timeout": 8,
         "banner_timeout": 12,
         "fast_cli": False,
     }

     try:
         with ConnectHandler(**params) as connection:
             result["reachable"] = True

             services = connection.send_command(
                 "show running-config | include "
                 "^ntp server|^logging host"
             )
             neighbors = connection.send_command(
                 "show ip ospf neighbor"
             )

             ntp_ok = bool(re.search(
                 rf"^ntp server {re.escape(APPROVED_NTP)}
(?:\s|$)",
                services,
                re.MULTILINE,
             ))
             syslog_ok = bool(re.search(
                 rf"^logging host "
                 rf"{re.escape(APPROVED_SYSLOG)}(?:\s|$)",
                 services,
                 re.MULTILINE,
             ))
             full_neighbors = len(re.findall(
                 r"\bFULL(?:/|-)",
                 neighbors,
             ))
             expected = int(device["expected_neighbors"])

             result["checks"] = {
                 "approved_ntp": {
                     "expected": APPROVED_NTP,
                     "pass": ntp_ok,
                 },
                 "approved_syslog": {
                     "expected": APPROVED_SYSLOG,
                     "pass": syslog_ok,
                 },
                 "ospf_full_neighbors": {
                     "expected": expected,
                     "actual": full_neighbors,
                     "pass": full_neighbors == expected,
                 },
             }

    except NetmikoAuthenticationException:
        result["errors"].append("authentication failed")
    except NetmikoTimeoutException:
        result["errors"].append("connection timed out")
    except OSError as exc:
        result["errors"].append(f"transport error: {exc}")

    except Exception as exc:
        result["errors"].append(
            f"unexpected collection error:
{type(exc).__name__}: {exc}"
        )

    checks_pass = bool(result["checks"]) and all(
        check["pass"] for check in
result["checks"].values()
    )
    result["pass"] = (
        result["reachable"]
        and not result["errors"]
        and checks_pass
    )
    return result

def main() -> int:
    username = os.environ["NET_USERNAME"]
    password = os.environ["NET_PASSWORD"]
    inventory =
json.loads(Path("inventory.json").read_text())

    results: list[dict[str, Any]] = []
    with ThreadPoolExecutor(max_workers=MAX_WORKERS) as
pool:
        future_map = {
            pool.submit(
                audit_device,
                device,
                username,
                password,
            ): device["name"]
            for device in inventory
        }
        for future in as_completed(future_map):
            results.append(future.result())

    results.sort(key=lambda item: item["name"])
    Path("audit-report.json").write_text(
        json.dumps(results, indent=2, sort_keys=True)
    )

    for item in results:
        state = "PASS" if item["pass"] else "FAIL"
        print(f"{item['name']:<4} {state}")
        for error in item["errors"]:
            print(f"     ERROR: {error}")

           for name, check in item["checks"].items():
               mark = "PASS" if check["pass"] else "FAIL"
               print(f"     {name:<24} {mark}")
