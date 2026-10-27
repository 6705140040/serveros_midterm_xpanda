Incident Log
Group: [write your group name/number here, then delete these brackets]
Members: Aung Myo Hlaing — 6705140040, plus [teammate names — replace this line with the real list]
Company / domain: novapeak.lab
Group id used with the checker: lab01
Scenario (from python3 /root/grading/params.py lab01 — the single source of truth for this group):
network 192.168.19.0/24, gateway 192.168.19.1, DNS (ns1) 192.168.19.10,
pool 192.168.19.104–192.168.19.144, lease 3600,
A records: ns1→.10, www→.20, mail→.30, api→.40, reverse zone 19.168.192.in-addr.arpa, interface lan0.
Method: run the checker, then work from each symptom to its cause. A timeout,
REFUSED, SERVFAIL, NXDOMAIN and "NOERROR but wrong answer" are different
failure modes and point at different causes. Every value in the live configs
was compared against params.py lab01 — a config can be syntactically valid
and still be semantically wrong.
Live run in KillerCoda, 2026-10-27: baseline checker TOTAL 11.0 / 25,
final checker TOTAL 25.0 / 25 (see checker_result.txt). Recording ended
with "Script done" (see session.log).
4 faults were found. Everything else (named.conf.options, the reverse
zone and its PTRs, INTERFACESv4="lan0", the DHCP pool, gateway, domain name
and lease times) was checked against params.py and was already correct.

Fault 1 — Firewall silently drops all DNS queries (UDP port 53)
Symptom (what we observed):
  Client dig timed out: "communications error … timed out / no servers
  could be reached". Baseline checker: C1 0/2 (client cannot reach named
  at 192.168.19.10), C2 all four A records got nothing, total 11/25.
  A timeout is not SERVFAIL/REFUSED — the query never gets an answer at
  all, which points at reachability (something dropping packets), not at
  zone data.
Hypothesis (what we suspected and why):
  A host firewall rule dropping inbound UDP/53. The instructions warn that
  "faults are not always in a config file" — a firewall rule is exactly
  that kind of fault, and it would also explain why the client can never
  resolve anything even if the server looks fine.
Proof (the command that confirmed the cause):
  $ iptables -L INPUT -n --line-numbers
  Chain INPUT (policy ACCEPT)
  num  target     prot opt source               destination
  1    DROP       17   --  0.0.0.0/0            0.0.0.0/0   udp dpt:53
  Rule 1 drops every inbound UDP packet to port 53 before named can see it.
Fix (what we changed):
  iptables -D INPUT -p udp --dport 53 -j DROP
  (No config file was changed — this fault does not exist in any file.)
Confirmation (output showing it now works):
  A second iptables -L INPUT -n --line-numbers was empty — no DROP rule.
  After this fix, queries reached named and the failure mode changed from
  timeout to SERVFAIL — proving the firewall was only the first fault,
  and exposing Fault 2 underneath it.
Found/proved by: [write the name of the member who did this fault]

Fault 2 — Forward zone points at a file that does not exist (db.MISSING)
Symptom (what we observed):
  After Fault 1, SERVFAIL; baseline checker C4 named-checkconf FAILED.
  SERVFAIL with no aa flag: the server is reachable now, but it is not
  authoritative for novapeak.lab — the zone failed to load.
Hypothesis (what we suspected and why):
  The zone statement in named.conf.local points at a wrong/missing zone file,
  or the zone file itself is invalid. named-checkconf -z loads every zone
  and says which.
Proof (the command that confirmed the cause):
  $ named-checkconf -z
  zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found
  zone novapeak.lab/IN: not loaded due to errors.
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101

  $ cat /etc/bind/named.conf.local
  zone "novapeak.lab" {
          type master;
          file "/etc/bind/db.MISSING";        <-- file does not exist
  };
  The reverse zone loads fine; only the forward zone's file path is wrong.
Fix (what we changed):
  In /etc/bind/named.conf.local: file "/etc/bind/db.MISSING"; →
  file "/etc/bind/db.novapeak.lab";, then systemctl restart named.
Confirmation (output showing it now works):
  $ named-checkconf -z
  zone novapeak.lab/IN: loaded serial 2024010101
  zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101

  $ ip netns exec client dig @192.168.19.10 www.novapeak.lab A
  www.novapeak.lab.    3600    IN    A    192.168.19.20
Found/proved by: [write the name of the member who did this fault]

Fault 3 — Wrong A record: api.novapeak.lab points at the gateway
Symptom (what we observed):
  Most names resolved, but api did not return the app address. NOERROR
  and authoritative — but the answer was wrong: 192.168.19.1 is the
  gateway address in this scenario, not the app host. A "successful"
  lookup with the wrong value is a data fault, not a service fault.
Hypothesis (what we suspected and why):
  A typo'd address in the forward zone file. params.py is the source of
  truth: it says api → 192.168.19.40 (ns1 .10, www .20, mail .30, api .40).
Proof (the command that confirmed the cause):
  $ python3 /root/grading/params.py lab01
    ns1.novapeak.lab  -> 192.168.19.10
    www.novapeak.lab  -> 192.168.19.20
    mail.novapeak.lab -> 192.168.19.30
    api.novapeak.lab  -> 192.168.19.40

  $ grep -n api /etc/bind/db.novapeak.lab
  api IN A 192.168.19.1
  Zone file disagrees with the scenario. Note named-checkzone
  reported this file as OK before the fix — the file is syntactically valid;
  only comparing the value against params.py catches this fault.
Fix (what we changed):
  In /etc/bind/db.novapeak.lab: api IN A 192.168.19.1 →
  api IN A 192.168.19.40, zone serial bumped 2024010101 → 2024010102,
  then systemctl restart named.
Confirmation (output showing it now works):
  Final checker C2 3.0/3 — all four A records correct, including
  api → 192.168.19.40.
Found/proved by: [write the name of the member who did this fault]

Fault 4 — DHCP advertises a public DNS server (8.8.8.8) instead of the company DNS
Symptom (what we observed):
  Clients could get a lease, but a client using only what the lease advertised
  still could not resolve company host names — the end-to-end test fails even
  with DNS itself fixed. Baseline checker: C9 0/3, C11 0/3 (client using
  leased resolver 8.8.8.8, www did not resolve). This is the integration
  fault the whole project is about: the lease points clients at the wrong
  resolver.
Hypothesis (what we suspected and why):
  option domain-name-servers in dhcpd.conf is set to a public resolver.
  A public resolver knows nothing about the private novapeak.lab zone, so
  every internal name fails for a DHCP-configured client — while working fine
  when tested by hand against the company server, which is why this fault is
  easy to miss.
Proof (the command that confirmed the cause):
  $ grep -n domain-name-servers /etc/dhcp/dhcpd.conf
        option domain-name-servers 8.8.8.8;

  $ dhcpd -t -cf /etc/dhcp/dhcpd.conf
  Config file: /etc/dhcp/dhcpd.conf
  dhcpd-t-exit:0
  Two facts together prove it: the advertised resolver is 8.8.8.8 while
  params.py says DHCP MUST advertise 192.168.19.10 — and dhcpd -t exits 0,
  so the file is syntactically valid. Syntax checking can never catch this;
  only comparing the advertised value against the scenario does. (The rest of
  dhcpd.conf — pool .104–.144, routers .1, domain novapeak.lab, lease
  3600/7200 — and INTERFACESv4="lan0" were all checked and are correct.)
Fix (what we changed):
  In /etc/dhcp/dhcpd.conf: option domain-name-servers 8.8.8.8; →
  option domain-name-servers 192.168.19.10;, then
  systemctl restart isc-dhcp-server.
Confirmation (output showing it now works):
  Final checker: C9 3/3 (DHCP advertises the company DNS 192.168.19.10)
  and C11 3/3 (client resolves www via the leased resolver).
Found/proved by: [write the name of the member who did this fault]

Final verification (all faults fixed)
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

Checker in KillerCoda:
python3 /root/grading/check_lab.py --group lab01 --phase build
TOTAL: 25.0 / 25