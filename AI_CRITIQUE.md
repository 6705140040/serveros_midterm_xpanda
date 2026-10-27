AI Usage and Critique
Group: [write your group name/number here, then delete these brackets]
Members: Aung Myo Hlaing — 6705140040, plus [teammate names — replace this line with the real list]
Part 3a: Declaration (required)
AI tool
What we used it for
Muse (Muse Spark, via Muse)
Reading the handout/repo, comparing the broken configs against params.py, suggesting diagnostic commands and fixes, drafting this submission's structure. All of its suggestions were verified against the live configs and tools before being used — see 3c.
[second AI — e.g. ChatGPT / Gemini]
[run the ready-made test in 3c, paste what it answered, then fill this row — do not leave this bracketed]
Part 3b: Confirm you diagnosed first
We completed our own diagnosis (INCIDENT_LOG.md) before the comparison below.
Briefly, the faults WE found on our own:
Firewall rule dropping inbound UDP/53 (not in any config file).
named.conf.local pointing the forward zone at a non-existent file (db.MISSING) — zone never loads, SERVFAIL.
api.novapeak.lab A record = 192.168.19.1 (the gateway) instead of 192.168.19.40.
dhcpd.conf advertising the public resolver 8.8.8.8 instead of the company DNS 192.168.19.10.
Part 3c: Putting the AIs to the test
We gave the following to at least two AI tools: the broken configuration files
(named.conf.local, named.conf.options, both zone files, dhcpd.conf,
isc-dhcp-server) plus the symptom "a machine joining this network cannot
resolve the company's own host names", and asked each AI to diagnose and fix.
Ready-made prompt used (paste the broken files after it for AI #2):
"You are a Linux sysadmin. A small company runs BIND9 DNS and ISC DHCP on
Ubuntu for the domain novapeak.lab (network 192.168.19.0/24, gateway .1,
DNS server .10, DHCP pool .104–.144, app host api should be .40). A machine
that joins the network with no manual configuration cannot resolve the
company's own host names. Here are the config files. Find every fault,
explain how you would prove each one on the live system, and give the fix."
Fault (in our words)
We found it?
AI #1 (Muse)
AI #2 ([name])
Was each AI actually right?
Firewall drops UDP/53
Yes — iptables -L INPUT
Found, but only because it read the lab's setup.sh, which contains the iptables command. From the config files alone there is no trace of this fault.
[paste what AI #2 said]
AI #1: right about the fix (iptables -D ...), but its "diagnosis" was reading the answer in a script, not reasoning from the live system. On a real server with no setup script it could only have found this by being told the symptom was a timeout and checking the live firewall — which it cannot do.
Forward zone file = db.MISSING
Yes — named-checkconf -z
Found immediately from named.conf.local
[paste what AI #2 said]
AI #1: correct, and correctly predicted SERVFAIL as the symptom. Verified live: named-checkconf -z printed "file not found / not loaded" before the fix and "loaded serial" after.
api A record .1 instead of .40
Yes — compared zone vs params.py
Found by comparing the zone file against params.py values
[paste what AI #2 said]
AI #1: correct. It also made the right subtle point that named-checkzone says "OK" for this file — a syntax check cannot catch a wrong-but-valid address. Verified live by dig and by checker C2.
DHCP advertises 8.8.8.8
Yes — compared dhcpd.conf vs params.py
Found, and flagged it as the key integration fault (C9/C11)
[paste what AI #2 said]
AI #1: correct. Verified live: dhcpd -t exits 0 both before and after, proving the fault is semantic; the fix (advertise 192.168.19.10) is what the checker tests from the client, and C9/C11 now pass.
For each AI suggestion that was WRONG or misleading, describe it and how you
verified it was wrong:
**AI #1 initially treated named.conf.options and /etc/default/isc-dhcp-server
  as suspects** (they are named in the handout as places faults "can" live).
  Checking them line-by-line against params.py showed both were already
  correct (listen-on/allow-query { any; }, INTERFACESv4="lan0"). An AI
  working from a symptom alone tends to rewrite files that are not broken;
  we verified instead of changing them, and the checker confirms they needed
  no change.
AI #1 could not prove its own fix end-to-end at first. In its sandbox,
  network namespaces were blocked and UDP/53 was intercepted, so it verified
  with named-checkconf -z, named-checkzone, dhcpd -t and dig over TCP — strong
  evidence, but NOT the client's-point-of-view test. That claim only became
  true when a human ran check_lab.py in KillerCoda: baseline 11/25 →
  final 25.0/25. We verified that distinction instead of trusting the claim.
[after testing AI #2, add anything it got wrong here — e.g. did it
  miss the firewall? did it suggest a public DNS "fix"? did it invent records
  (MX, extra hosts) the scenario does not have?]
Part 3d: Verdict
Where our human diagnosis beat the AIs:
  The firewall fault. It exists nowhere in the configuration — it is a live
  rule on a running system, visible only to someone who (a) notices the
  symptom is a timeout rather than SERVFAIL/REFUSED and (b) thinks to look
  at iptables -L on the actual machine. Diagnosing a live system starts
  from observing the running system; an AI that is handed files can only
  reason about the files. Our incident-log method — symptom → hypothesis →
  proof command — is what found it, and the same method caught the two
  "valid syntax, wrong value" faults (api address, DHCP resolver) that pure
  config-checking tools pass silently.
Where an AI genuinely helped:
  Speed of comparison and completeness. Once pointed at params.py as the
  source of truth, the AI compared every value in every file against the
  scenario in seconds, predicted which checker items each fault would fail
  (including that the 8.8.8.8 fault breaks end-to-end resolution even when
  DNS itself is perfect), and explained the timeout-vs-SERVFAIL distinction
  clearly. As a cross-checker and explainer it was genuinely useful — after
  we had a hypothesis.
What we learned about trusting AI to diagnose a live system it cannot see:
  An AI's confidence is not evidence. It can produce a perfect-looking config
  while missing the one fault that lives outside the files, and it cannot run
  the checker from the client's point of view — so its "fixed" is always a
  claim, not a result, until a human verifies it on the real system. The safe
  workflow is: diagnose from symptoms ourselves, use the AI to cross-check
  values and suggest proof commands, and accept a fix only when a tool on the
  real system (named-checkconf, dig from the client, check_lab.py) confirms it.