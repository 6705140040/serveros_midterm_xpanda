Submission — Server OS Midterm (192-442), Lab 01
Group name / number: [write your group name/number here, then delete these brackets]
Members (name + GitHub username, one per line):
Aung Myo Hlaing — 6705140040
[teammate name] — [teammate GitHub username]
[teammate name] — [teammate GitHub username]
Company / domain you were assigned: novapeak — novapeak.lab
Group id used with the checker: lab01
Contents
fixed-configs/ — the repaired configuration (named.conf.options,
  named.conf.local, db.novapeak.lab, db.novapeak.lab.rev, dhcpd.conf,
  isc-dhcp-server). The 4th fault (firewall DROP on UDP/53) is a live rule,
  not a file — its fix is documented in INCIDENT_LOG.md Fault 1.
INCIDENT_LOG.md — how each of the 4 faults was found and proved (Part 2).
AI_CRITIQUE.md — AI declaration, comparison and verdict (Part 3).
session.log — recorded terminal session of the diagnosis and fixes.
checker_result.txt — verification results.
Result summary
Faults fixed: (1) iptables DROP on UDP/53 removed, (2) forward zone file
db.MISSING → db.novapeak.lab, (3) api A record 192.168.19.1 → 192.168.19.40,
(4) DHCP resolver 8.8.8.8 → 192.168.19.10. Live checker in KillerCoda
2026-10-27: baseline 11/25 → final TOTAL: 25.0 / 25, all C1–C11 pass —
see checker_result.txt.