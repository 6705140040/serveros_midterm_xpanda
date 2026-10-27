## AI Usage and Critique

Group: Team X-Panda
Members: Aung Myo Hlaing (Olary) — 6705140040, Hein Zaw Lin (Hubert) — 6705140038, Hein Thura Naung (Ivan) — 6705140056, Chan Myae Thaw Tar (Chan) — 6705140048, Aye Than Tin (Aye) — 6705140074

## Part 3a: Declaration (required)

| AI tool | What we used it for |
| --- | --- |
| Muse (Muse Spark, via Muse) | Reading the handout/repo, comparing the broken configs against `params.py`, suggesting diagnostic commands and fixes, drafting this submission's structure. All of its suggestions were verified against the live configs and tools before being used — see 3c. |
| Muse Deep Research agent (isolated research agent, second AI tool) | Given only the broken config files and the symptom, with no information about what was wrong, and asked to find every fault, give a proof command for each, and say what it was unsure about. Its report is summarised in 3c below. It is not ChatGPT or Gemini — it is a second, independent AI tool, named honestly here. |

## Part 3b: Confirm you diagnosed first

We completed our own diagnosis (INCIDENT\_LOG.md) before the comparison below.
Briefly, the faults WE found on our own:

1. Firewall rule dropping inbound UDP/53 (not in any config file).
2. named.conf.local pointing the forward zone at a non-existent file (db.MISSING) — zone never loads, SERVFAIL.
3. api.novapeak.lab A record = 192.168.19.1 (the gateway) instead of 192.168.19.40.
4. dhcpd.conf advertising the public resolver 8.8.8.8 instead of the company DNS 192.168.19.10.

## Part 3c: Putting the AIs to the test

We gave the following to at least two AI tools: the broken configuration files
(named.conf.local, named.conf.options, both zone files, dhcpd.conf,
isc-dhcp-server) plus the symptom "a machine joining this network cannot
resolve the company's own host names", and asked each AI to diagnose and fix.

Ready-made prompt used (paste the broken files after it for AI #2):

> "You are a Linux sysadmin. A small company runs BIND9 DNS and ISC DHCP on
> Ubuntu for the domain novapeak.lab (network 192.168.19.0/24, gateway .1,
> DNS server .10, DHCP pool .104–.144, app host api should be .40). A machine
> that joins the network with no manual configuration cannot resolve the
> company's own host names. Here are the config files. Find every fault,
> explain how you would prove each one on the live system, and give the fix."

| Fault (in our words) | We found it? | AI #1 (Muse) | AI #2 (Muse Deep Research agent) | Was each AI actually right? |
| --- | --- | --- | --- | --- |
| Firewall drops UDP/53 | Yes — `iptables -L INPUT` | Found, **but only because it read the lab's setup.sh**, which contains the iptables command. From the config files alone there is no trace of this fault. | Did not claim it from the files; listed server firewall blocking DNS as a live-only possibility, with localhost-works / client-times-out as its signature | AI #1: right about the fix (`iptables -D ...`), but its "diagnosis" was reading the answer in a script, not reasoning from the live system. On a real server with no setup script it could only have found this by being told the symptom was a *timeout* and checking the live firewall — which it cannot do. |
| Forward zone file = db.MISSING | Yes — `named-checkconf -z` | Found immediately from named.conf.local | Found immediately from named.conf.local: db.MISSING does not exist, zone never loads | AI #1: correct, and correctly predicted SERVFAIL as the symptom. Verified live: named-checkconf -z printed "file not found / not loaded" before the fix and "loaded serial" after. |
| api A record .1 instead of .40 | Yes — compared zone vs params.py | Found by comparing the zone file against params.py values | Found by comparing zone vs scenario: api .1 is the gateway, not .40; reverse PTR already correct | AI #1: correct. It also made the right subtle point that named-checkzone says "OK" for this file — a syntax check cannot catch a wrong-but-valid address. Verified live by dig and by checker C2. |
| DHCP advertises 8.8.8.8 | Yes — compared dhcpd.conf vs params.py | Found, and flagged it as the key integration fault (C9/C11) | Found: DHCP hands out 8.8.8.8, which knows nothing about novapeak.lab; proved by NXDOMAIN via default resolver vs correct answer via .10 | AI #1: correct. Verified live: dhcpd -t exits 0 both before and after, proving the fault is semantic; the fix (advertise 192.168.19.10) is what the checker tests from the client, and C9/C11 now pass. |

For each AI suggestion that was WRONG or misleading, describe it and how you
verified it was wrong:

- \*\*AI #1 initially treated named.conf.options and /etc/default/isc-dhcp-server
  as suspects\*\* (they are named in the handout as places faults "can" live).
  Checking them line-by-line against params.py showed both were already
  correct (`listen-on/allow-query { any; }`, `INTERFACESv4="lan0"`). An AI
  working from a symptom alone tends to rewrite files that are not broken;
  we verified instead of changing them, and the checker confirms they needed
  no change.
- **AI #1 could not prove its own fix end-to-end at first.** In its sandbox,
  network namespaces were blocked and UDP/53 was intercepted, so it verified
  with named-checkconf -z, named-checkzone, dhcpd -t and dig over TCP — strong
  evidence, but NOT the client's-point-of-view test. That claim only became
  true when a human ran check\_lab.py in KillerCoda: baseline 11/25 →
  final 25.0/25. We verified that distinction instead of trusting the claim.
- **AI #2 flagged an extra, probable fault that we did not adopt:** unquoted `option domain-name novapeak.lab;` in dhcpd.conf, which it said must be quoted. We checked the live fixed file: it already uses `option domain-name "novapeak.lab";`, and `dhcpd -t` passes and checker C5 passes, so this was not a live fault in our submission — AI #2 itself flagged it as probable, not certain, and said to settle it with `dhcpd -t`.
- **AI #2 did not prove the specific firewall rule from the files alone.** It correctly listed firewall blocking as a live-only possibility, but it could not name the iptables DROP on UDP/53 rule, because that rule is not in any config file. Our human diagnosis found that specific rule with `iptables -L INPUT`, which is why human diagnosis beat both AIs on Fault 1.
- **AI #2 correctly did NOT invent faults:** it explicitly said `recursion no`, `listen-on { any; }`, `allow-query { any; }`, the reverse zone, the DHCP range, routers .1, and the PTRs are not faults.

## Part 3d: Verdict

- Where our human diagnosis beat the AIs:
  The firewall fault. It exists nowhere in the configuration — it is a live
  rule on a running system, visible only to someone who (a) notices the
  symptom is a *timeout* rather than SERVFAIL/REFUSED and (b) thinks to look
  at `iptables -L` on the actual machine. Diagnosing a live system starts
  from observing the running system; an AI that is handed files can only
  reason about the files. Our incident-log method — symptom → hypothesis →
  proof command — is what found it, and the same method caught the two
  "valid syntax, wrong value" faults (api address, DHCP resolver) that pure
  config-checking tools pass silently.
- Where an AI genuinely helped:
  Speed of comparison and completeness. Once pointed at params.py as the
  source of truth, the AI compared every value in every file against the
  scenario in seconds, predicted which checker items each fault would fail
  (including that the 8.8.8.8 fault breaks end-to-end resolution even when
  DNS itself is perfect), and explained the timeout-vs-SERVFAIL distinction
  clearly. As a cross-checker and explainer it was genuinely useful — after
  we had a hypothesis.
- What we learned about trusting AI to diagnose a live system it cannot see:
  An AI's confidence is not evidence. It can produce a perfect-looking config
  while missing the one fault that lives outside the files, and it cannot run
  the checker from the client's point of view — so its "fixed" is always a
  claim, not a result, until a human verifies it on the real system. The safe
  workflow is: diagnose from symptoms ourselves, use the AI to cross-check
  values and suggest proof commands, and accept a fix only when a tool on the
  real system (named-checkconf, dig from the client, check\_lab.py) confirms it.
