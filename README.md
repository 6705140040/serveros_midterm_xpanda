# Server OS Midterm — Lab 01 (novapeak)

Group: Team X-Panda
Company / domain: novapeak — novapeak.lab
Group id used with the checker: lab01

## Members

| Name | GitHub username |
| --- | --- |
| Aung Myo Hlaing (Olary) | 6705140040 |
| Hein Zaw Lin (Hubert) | 6705140038 |
| Hein Thura Naung (Ivan) | 6705140056 |
| Chan Myae Thaw Tar (Chan) | 6705140048 |
| Aye Than Tin (Aye) | 6705140074 |

## What this repo contains

```
.
├── README.md
├── fixed-configs/
│   ├── named.conf.options
│   ├── named.conf.local
│   ├── db.novapeak.lab
│   ├── db.novapeak.lab.rev
│   ├── dhcpd.conf
│   └── isc-dhcp-server
├── INCIDENT_LOG.md
├── AI_CRITIQUE.md
├── session.log
└── checker_result.txt
```

- `fixed-configs/` — the configs we fixed. The firewall fault has no file, because it was a live rule. We wrote about it in INCIDENT_LOG.md, Fault 1. Peer review by Chan: fixed-configs contains exactly six files, as listed.
- `INCIDENT_LOG.md` — how we found and proved each of the 4 faults.
- `AI_CRITIQUE.md` — what AI tools we used, how they did, and what we learned.
- `session.log` — our recorded terminal session.
- `checker_result.txt` — the live checker output from KillerCoda.

## Faults fixed

1. Firewall rule dropping inbound UDP/53 removed — `iptables -D INPUT -p udp --dport 53 -j DROP`. Peer review by Hubert: Fault 1 description verified — the firewall fault has no file, as documented in the incident log.
2. Forward zone file `db.MISSING` → `db.novapeak.lab` in `named.conf.local`, then `systemctl restart named`.
3. api A record `192.168.19.1` (the gateway) → `192.168.19.40`, SOA serial `2024010101` → `2024010102`.
4. DHCP resolver `8.8.8.8` → company DNS `192.168.19.10` in `dhcpd.conf`, then `systemctl restart isc-dhcp-server`.

Scenario, from `params.py lab01`: network `192.168.19.0/24`, gateway `192.168.19.1`,
DNS ns1 `192.168.19.10`, www `.20`, mail `.30`, api `.40`,
DHCP pool `192.168.19.104–192.168.19.144`, lease `3600`, interface `lan0`.

Peer review by Ivan: Scenario values verified against params.py lab01 — network, gateway, ns1, www, mail, api and pool.
## Result

Live checker in KillerCoda, 2026-10-27:

```text
python3 /root/grading/check_lab.py --group lab01 --phase build
```

| Check | Result |
| --- | --- |
| C4 — named-checkconf / named-checkzone clean | 2/2 |
| C5 — dhcpd clean / service running | 2/2 |
| C1 — named authoritative and reachable from client | 2/2 |
| C2 — all four A records correct | 3.0/3 |
| C3 — reverse lookup of 192.168.19.10 returns ns1 | 2/2 |
| C6 — client obtained lease 192.168.19.104 | 3/3 |
| C7 — pool + netmask correct | 2.0/2 |
| C8 — gateway advertised correctly | 2/2 |
| C9 — DHCP advertises company DNS (192.168.19.10) | 3/3 |
| C10 — company domain advertised | 1/1 |
| C11 — client resolves www via the leased resolver | 3/3 |
| **Total** | **25.0 / 25** |

Peer review by Hubert: Checker rows C1 and C4 verified against checker_result.txt.

Peer review by Hubert: Baseline checker in session.log verified — C1 0/2, C2 got nothing, total 11.0/25.

Peer review by Chan: Checker rows C6, C7 and C8 verified against checker_result.txt — lease .104, pool and gateway.

Peer review by Chan: Checker rows C9, C10 and C11 verified against checker_result.txt — DNS, domain and client resolves www.

Baseline before the fixes was 11/25. Full output is in `checker_result.txt`.
