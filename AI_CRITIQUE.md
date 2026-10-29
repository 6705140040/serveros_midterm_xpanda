## AI Usage and Critique

Group: Team X-Panda
Members: Aung Myo Hlaing (Olary) — 6705140040, Hein Zaw Lin (Hubert) — 6705140038, Hein Thura Naung (Ivan) — 6705140056, Chan Myae Thaw Tar (Chan) — 6705140048, Aye Than Tin (Aye) — 6705140074

## Part 3a: Declaration (required)

| AI tool | What we used it for |
| --- | --- |
| Muse (Muse Spark, via Muse) | We used it to read the handout, compare the broken configs with `params.py`, and suggest commands and fixes. We checked every suggestion on the live system before we used it. See 3c. |
| ChatGPT, run by the team on 2026-10-27 | We gave it only the broken config files and the symptom, and asked it to find every fault and give a proof command for each. Its answer is summarised in 3c below. |

## Part 3b: Confirm you diagnosed first

We did our own diagnosis first, and wrote it in INCIDENT_LOG.md. These are the faults we found ourselves:

1. Firewall rule dropping inbound UDP/53. This rule is not in any config file.
2. `named.conf.local` points the forward zone at a file that does not exist (`db.MISSING`), so the zone never loads and we get SERVFAIL.
3. `api.novapeak.lab` is 192.168.19.1 (the gateway) instead of 192.168.19.40.
4. `dhcpd.conf` gives clients 8.8.8.8 instead of our DNS, 192.168.19.10.

## Part 3c: Putting the AIs to the test

We gave both AIs the broken config files (named.conf.local, named.conf.options, both zone files, dhcpd.conf, isc-dhcp-server) and the symptom: a machine joining the network cannot resolve the company's own host names. We asked each one to diagnose and fix.

| Fault (in our words) | We found it? | AI #1 (Muse) | AI #2 (ChatGPT) | Was each AI actually right? |
| --- | --- | --- | --- | --- |
| Firewall drops UDP/53 | Yes — `iptables -L INPUT` | Found it, but only because it read the lab's setup.sh, which has the iptables command. The config files alone show nothing about this fault. | Did not find the exact iptables DROP rule. It only said to check the firewall on the live system, and that a timeout fits a firewall drop but does not prove it. | AI #1 got the fix right (`iptables -D ...`), but it did not really diagnose it — it read the answer in a script. On a real server with no setup script, it could only have found this by checking the live firewall. AI #2 was not right on this fault — it only said to check the firewall, it did not find the rule. |
| Forward zone file = db.MISSING | Yes — `named-checkconf -z` | Found it straight from named.conf.local. | Found it straight from named.conf.local. Proof: `named-checkconf -z`. | AI #1 was right, and it correctly said the symptom would be SERVFAIL. We checked live: before the fix, named-checkconf said "file not found / not loaded", after the fix it said "loaded serial". AI #2 was right too. |
| api A record .1 instead of .40 | Yes — compared zone vs params.py | Found it by comparing the zone with params.py. | Found it by comparing the zone with the scenario. Proof: grep, `named-checkzone`, and dig. It also said a syntax check cannot catch a wrong-but-valid IP. | AI #1 was right. We checked live with dig and with checker C2. AI #2 was right too. |
| DHCP advertises 8.8.8.8 | Yes — compared dhcpd.conf vs params.py | Found it, and said it was the key fault for C9/C11. | Found it. Proof: grep, `dhcpd -t`, and checking what a fresh lease actually gives the client. | AI #1 was right. We checked live: `dhcpd -t` passes both before and after, so the file is not broken in syntax — it is just the wrong value. After the fix, C9 and C11 pass. AI #2 was right too. |

For each AI suggestion that was WRONG or misleading, here is what it was and how we checked it:

- **AI #1 first said named.conf.options and isc-dhcp-server might be broken.** They are named in the handout as places faults "can" live. We checked them line by line against params.py and both were already right (`listen-on/allow-query { any; }`, `INTERFACESv4="lan0"`). An AI working from a symptom alone tends to rewrite files that are not broken. We checked instead of changing them, and the checker confirms they needed no change.
- **AI #1 could not prove its own fix end-to-end at first.** In its sandbox, network namespaces were blocked and UDP/53 was intercepted, so it checked with named-checkconf, named-checkzone, dhcpd -t, and dig over TCP. That is strong evidence, but it is not the client's test. That claim only became true when we ran check_lab.py in KillerCoda: baseline 11/25, final 25.0/25.
- **ChatGPT made up a fault that does not exist.** It said `20 IN PTR www\.novapeak.lab.` in the reverse zone, with a backslash before the dot. The file we gave it actually says `20 IN PTR www.novapeak.lab.`, with no backslash. The reverse PTRs are correct, and checker C3 passes. We found this by reading the file again, not by trusting what the AI quoted. This is the clearest example in this project of an AI sounding sure while quoting a file wrong.
- **ChatGPT counted four config faults, but that is not right.** It found db.MISSING, api .1, and DHCP 8.8.8.8 correctly, then added the made-up PTR fault to make four, and it missed the live firewall fault. The real total is three faults in files plus one live firewall fault, which is four faults in total.
- **ChatGPT's serial suggestion was different, not wrong.** It said to raise the SOA serial to 2026102701 when fixing api. We raised it to 2024010102 instead. Raising the serial when you change a zone is good practice. The exact new number does not matter, so we did not count this as a mistake.
- **ChatGPT got the other settings right.** It said `recursion no`, `listen-on { any; }`, `allow-query { any; }`, the DHCP pool, routers .1, lease times, and `INTERFACESv4="lan0"` are not faults. It also said a missing zone file alone would give SERVFAIL, not a timeout. That matches what we saw live: SERVFAIL only showed up after we removed the firewall rule.
- Peer note by Chan: ChatGPT quoting the PTR record with a backslash shows why an AI quote must be checked against the actual file.

## Part 3d: Verdict

- Where our human diagnosis beat the AIs:
The firewall fault. It is nowhere in the config files. It is a live rule on a running system. You only find it if you notice the symptom is a timeout, not SERVFAIL or REFUSED, and you think to look at `iptables -L` on the real machine. An AI that is only given files can only reason about files. Our method in the incident log — symptom, then hypothesis, then proof command — is what found it. The same method also caught the two "valid syntax, wrong value" faults (api address and DHCP resolver) that config-checking tools pass without complaint.

- Where an AI genuinely helped:
Speed and completeness. Once we pointed it at params.py as the source of truth, it compared every value in every file against the scenario in seconds, said which checker items each fault would fail, and explained the timeout vs SERVFAIL difference clearly. As a cross-checker and explainer, it was genuinely useful — after we already had a hypothesis.

- What we learned about trusting AI to diagnose a live system it cannot see:
An AI sounding confident is not evidence. It can give you a perfect-looking config while missing the one fault that lives outside the files, and it can even quote a file wrong, like ChatGPT did with the PTR record. It also cannot run the checker from the client's point of view. So its "fixed" is always just a claim until a human checks it on the real system. The safe way to work is: diagnose from symptoms ourselves first, use the AI to cross-check values and suggest proof commands, and only accept a fix when a tool on the real system (named-checkconf, dig from the client, check_lab.py) confirms it.
