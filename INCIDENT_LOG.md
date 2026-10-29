## Incident Log — Lab 01 (novapeak)

Group: Team X-Panda
Members: Aung Myo Hlaing (Olary) — 6705140040, Hein Zaw Lin (Hubert) — 6705140038, Hein Thura Naung (Ivan) — 6705140056, Chan Myae Thaw Tar (Chan) — 6705140048, Aye Than Tin (Aye) — 6705140074
Company / domain: novapeak — novapeak.lab
Group id used with the checker: lab01

## Scenario

From `python3 /root/grading/params.py lab01`. This is the source of truth for our group:

| Item | Value |
| --- | --- |
| Network | 192.168.19.0/24 |
| Gateway | 192.168.19.1 |
| DNS (ns1) | 192.168.19.10 |
| www / mail / api | 192.168.19.20 / .30 / .40 |
| DHCP pool | 192.168.19.104–192.168.19.144 |
| Lease | 3600 |
| Reverse zone | 19.168.192.in-addr.arpa |
| Interface | lan0 |

How we worked: we ran the checker first, then worked from each symptom to its cause. A timeout, REFUSED, SERVFAIL, NXDOMAIN, and "NOERROR but wrong answer" are not the same thing. Each one points at a different kind of cause. We also compared every value in the live configs with params.py, because a config can be right in syntax and still wrong in value.

Live run in KillerCoda, 2026-10-27: baseline checker TOTAL 11.0 / 25, final checker TOTAL 25.0 / 25 (see `checker_result.txt`). Recording ended with "Script done" (see `session.log`).

## Summary

| Fault | What was wrong | File / live system changed | Confirmation |
| --- | --- | --- | --- |
| 1 | Firewall drops UDP/53 | Live iptables rule (no file) | Queries reach named, timeout → SERVFAIL |
| 2 | Forward zone file does not exist | `named.conf.local` | Zone loads, www → 192.168.19.20 |
| 3 | api points at the gateway | `db.novapeak.lab` | api → 192.168.19.40, checker C2 passes |
| 4 | DHCP advertises 8.8.8.8 | `dhcpd.conf` | Checker C9 and C11 pass |

Everything else — named.conf.options, the reverse zone and its PTRs, `INTERFACESv4="lan0"`, the DHCP pool, gateway, domain name, and lease times — we checked against params.py and it was already right.

---

## Fault 1 — Firewall drops all DNS queries (UDP port 53)

- **Symptom (what we observed):**
  Client dig timed out: "communications error … timed out / no servers could be reached". Baseline checker: C1 0/2 (client cannot reach named at 192.168.19.10), C2 all four A records got nothing, total 11/25.
  A timeout is not SERVFAIL or REFUSED. The query never gets an answer at all. That points at reachability — something dropping packets — not at zone data.
- **Hypothesis (what we suspected and why):**
  A firewall rule on the host dropping inbound UDP/53. The instructions say faults are not always in a config file, and a firewall rule is exactly that kind of fault. It would also explain why the client could never resolve anything, even if the server looked fine.
- **Proof (the command that confirmed the cause):**

  ```text
  $ iptables -L INPUT -n --line-numbers
  Chain INPUT (policy ACCEPT)
  num  target     prot opt source               destination
  1    DROP       17   --  0.0.0.0/0            0.0.0.0/0   udp dpt:53
  ```

  Rule 1 drops every inbound UDP packet to port 53 before named can see it.
- **Fix (what we changed):**

  ```bash
  iptables -D INPUT -p udp --dport 53 -j DROP
  ```

  No config file was changed. This fault does not exist in any file.
- **Confirmation (output showing it now works):**
  A second `iptables -L INPUT -n --line-numbers` was empty — no DROP rule left.
  After this fix, queries reached named and the failure changed from timeout to SERVFAIL. That proved the firewall was only the first fault, and it showed Fault 2 underneath it.
- **Found/proved by:** Aung Myo Hlaing (Olary) — live KillerCoda run, 2026-10-27

---

## Fault 2 — Forward zone points at a file that does not exist (db.MISSING)

- **Symptom (what we observed):**
  After Fault 1, we got SERVFAIL, and baseline checker C4 named-checkconf FAILED.
  SERVFAIL with no `aa` flag means the server is reachable now, but it is not authoritative for novapeak.lab. The zone failed to load.
- **Hypothesis (what we suspected and why):**
  The zone statement in named.conf.local points at a wrong or missing zone file, or the zone file itself is bad. `named-checkconf -z` loads every zone and tells you which one fails.
- **Proof (the command that confirmed the cause):**

  ```text
  $ named-checkconf -z
  zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
  zone novapeak.lab/IN: not loaded due to errors.
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101

  $ cat /etc/bind/named.conf.local
  zone "novapeak.lab" {
          type master;
          file "/etc/bind/db.MISSING";        <-- file does not exist
  };
  ```

  The reverse zone loads fine. Only the forward zone's `file` path is wrong.
- **Fix (what we changed):**
  In `/etc/bind/named.conf.local`: `file "/etc/bind/db.MISSING";` → `file "/etc/bind/db.novapeak.lab";`, then `systemctl restart named`.
- **Confirmation (output showing it now works):**

  ```text
  $ named-checkconf -z
  zone novapeak.lab/IN: loaded serial 2024010101
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101

  $ ip netns exec client dig @192.168.19.10 www.novapeak.lab A
  www.novapeak.lab.    3600    IN    A    192.168.19.20
  ```
- **Found/proved by:** Aung Myo Hlaing (Olary) — live KillerCoda run, 2026-10-27

---

## Fault 3 — Wrong A record: api.novapeak.lab points at the gateway

- **Symptom (what we observed):**
  Most names resolved, but api did not return the app address. The lookup said NOERROR and authoritative, but the answer was wrong: 192.168.19.1 is the gateway in this scenario, not the app host. A "successful" lookup with the wrong value is a data fault, not a service fault.

  Note on evidence: A separate live pre-fix dig for api on its own was not captured, because before Fault 1 the client timed out on everything (checker C2: got nothing), and before Fault 2 the server gave SERVFAIL. Once Faults 1 and 2 were fixed, the file comparison showed the fault: grep api gave 192.168.19.1, while params.py says api goes to 192.168.19.40.
- **Hypothesis (what we suspected and why):**
  A wrong address typed in the forward zone file. params.py is the source of truth: it says api → 192.168.19.40 (ns1 .10, www .20, mail .30, api .40).
- **Proof (the command that confirmed the cause):**

  ```text
  $ python3 /root/grading/params.py lab01
    ns1.novapeak.lab  -> 192.168.19.10
    www.novapeak.lab  -> 192.168.19.20
    mail.novapeak.lab -> 192.168.19.30
    api.novapeak.lab  -> 192.168.19.40

  $ grep -n api /etc/bind/db.novapeak.lab
  api IN A 192.168.19.1
  ```

  The zone file disagrees with the scenario. Note that `named-checkzone` said this file was OK before the fix. The file is right in syntax. Only comparing the value with params.py catches this fault.
- **Fix (what we changed):**
  In `/etc/bind/db.novapeak.lab`: `api IN A 192.168.19.1` → `api IN A 192.168.19.40`, zone serial 2024010101 → 2024010102, then `systemctl restart named`.
- **Confirmation (output showing it now works):**

  ```text
  $ ip netns exec client dig @192.168.19.10 api.novapeak.lab A

  ; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @192.168.19.10 api.novapeak.lab A
  ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63165
  ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

  ;; ANSWER SECTION:
  api.novapeak.lab.    3600    IN    A    192.168.19.40

  ;; SERVER: 192.168.19.10#53(192.168.19.10) (UDP)
  ;; WHEN: Tue Oct 27 07:51:53 UTC 2026
    ```

    Final checker C2 3.0/3 — all four A records correct, including api → 192.168.19.40.
  
- **Found/proved by:** Aung Myo Hlaing (Olary) — live KillerCoda run, 2026-10-27

---

## Fault 4 — DHCP advertises a public DNS server (8.8.8.8) instead of the company DNS

- **Symptom (what we observed):**
  Clients could get a lease, but a client using only what the lease gave it still could not resolve company host names. The end-to-end test fails even with DNS itself fixed. Baseline checker: C9 0/3, C11 0/3 (client using leased resolver 8.8.8.8, www did not resolve). This is the integration fault the whole project is about: the lease points clients at the wrong resolver.
- **Hypothesis (what we suspected and why):**
  `option domain-name-servers` in dhcpd.conf is set to a public resolver. A public resolver knows nothing about the private novapeak.lab zone, so every internal name fails for a DHCP client — while working fine when you test by hand against the company server. That is why this fault is easy to miss.
- **Proof (the command that confirmed the cause):**

  ```text
  $ grep -n domain-name-servers /etc/dhcp/dhcpd.conf
        option domain-name-servers 8.8.8.8;

  $ dhcpd -t -cf /etc/dhcp/dhcpd.conf
  Config file: /etc/dhcp/dhcpd.conf
  dhcpd-t-exit:0
  ```

  These two facts together prove it: the advertised resolver is 8.8.8.8 while params.py says DHCP must advertise 192.168.19.10, and `dhcpd -t` exits 0, so the file is right in syntax. Syntax checking can never catch this. Only comparing the advertised value with the scenario does. (The rest of dhcpd.conf — pool .104–.144, routers .1, domain novapeak.lab, lease 3600/7200 — and INTERFACESv4="lan0" were all checked and are right.) Peer review by Chan: DHCP pool 192.168.19.104–144, routers 192.168.19.1, domain novapeak.lab and lease 3600/7200 were already correct and dhcpd -t exited 0 before and after the fix, proving the fault was a wrong value, not a syntax error.
- **Fix (what we changed):**
  In `/etc/dhcp/dhcpd.conf`: `option domain-name-servers 8.8.8.8;` → `option domain-name-servers 192.168.19.10;`, then `systemctl restart isc-dhcp-server`.
- **Confirmation (output showing it now works):**
  Final checker: C9 3/3 (DHCP advertises the company DNS 192.168.19.10) and C11 3/3 (client resolves www via the leased resolver).
  
- Peer review by Chan: dhcpd.conf advertises 192.168.19.10, not 8.8.8.8, and checker C9 and C11 pass and INTERFACESv4="lan0" in isc-dhcp-server was checked and needed no change.
- **Found/proved by:** Aung Myo Hlaing (Olary) — live KillerCoda run, 2026-10-27

---

## Final verification (all faults fixed)

```text
$ named-checkconf -z
zone novapeak.lab/IN: loaded serial 2024010102
zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010102

$ named-checkzone novapeak.lab /etc/bind/db.novapeak.lab            -> OK
$ named-checkzone 19.168.192.in-addr.arpa /etc/bind/db.novapeak.lab.rev -> OK
$ dhcpd -t -cf /etc/dhcp/dhcpd.conf                                -> exit 0

$ ip netns exec client dig @192.168.19.10 ns1.novapeak.lab  A  -> 192.168.19.10
$ ip netns exec client dig @192.168.19.10 www.novapeak.lab  A  -> 192.168.19.20
$ ip netns exec client dig @192.168.19.10 mail.novapeak.lab A  -> 192.168.19.30
$ ip netns exec client dig @192.168.19.10 api.novapeak.lab  A  -> 192.168.19.40
$ ip netns exec client dig @192.168.19.10 -x 192.168.19.10     -> ns1.novapeak.lab.
$ iptables -L INPUT -n                    -> no DROP rule on udp dpt:53
$ cat /etc/default/isc-dhcp-server        -> INTERFACESv4="lan0"
```

Checker in KillerCoda:

```text
python3 /root/grading/check_lab.py --group lab01 --phase build
TOTAL: 25.0 / 25
```
