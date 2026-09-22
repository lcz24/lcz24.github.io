---
title: "Building a Home Detection Lab with Sysmon, Zeek, and Sigma"
date: 2025-08-11
draft: false
tags: ["Detection Engineering", "Sysmon", "Zeek", "Sigma", "Blue Team"]
categories: ["Blue Team"]
summary: "A practical blueprint for a lab that lets you write a detection, generate the telemetry, and prove the rule fires — all on one machine."
---

Detection engineering is learned by iteration: write a rule, generate the behaviour, check the alert, fix the rule. Without a lab, that loop involves production systems, change control, and other people's weekends.

The goal of this build is a lab where you can go from hypothesis to validated detection in under an hour.

## What the lab needs

Four capabilities, nothing more:

1. **Telemetry collection** — endpoint and network.
2. **Storage and query** — something that can answer ad-hoc questions.
3. **A source of behaviour** — either your own tools or an emulation framework.
4. **Rule format decoupled from the platform** — so rules are portable.

## Endpoint telemetry: Sysmon

Sysmon is still the highest signal-to-noise endpoint sensor available for free. Install with a configuration that actually enables what you need:

```powershell
sysmon64.exe -accepteula -i sysmonconfig.xml
```

For a lab, be generous with collection. You are not paying for storage, and an event you didn't collect is an event you can't write a rule against:

| Event ID | Why it matters |
| --- | --- |
| 1 | Process creation — lineage and arguments |
| 3 | Network connection — process-attributed egress |
| 7 | Image load — DLLs into sensitive processes |
| 8 | CreateRemoteThread — injection patterns |
| 11 | File create — dropped payloads |
| 12/13/14 | Registry — persistence |
| 22 | DNS query — with process attribution |

Event ID 22 is underrated. DNS with process attribution connects a network indicator back to the process that caused it, which is often the missing link in a detection.

## Network telemetry: Zeek

Zeek turns a packet capture into structured logs you can query. On a single lab host it works fine against a virtual interface:

```bash
zeek -i eth1 local "Site::local_nets = { 10.10.10.0/24 }"
```

The logs worth knowing:

- **`conn.log`** — every connection, with bytes and duration
- **`dns.log`** — queries and responses
- **`http.log`** — methods, URIs, user agents
- **`ssl.log`** — JA3 fingerprints and certificate details
- **`files.log`** — extracted file metadata

`ssl.log` is where JA3/JA3S earns its place: a TLS fingerprint is often more stable across tooling changes than any user-agent string.

## Storage: keep it boring

For a single-machine lab, a text index is enough:

```bash
# Ship everything into a simple daily index
zeek-cut < conn.log | head
```

If you want to go further, Elastic with a modest resource allocation is the common choice — but do not let the platform become the project. A lab that you query with `grep` and `jq` beats a lab you never finished building.

## Rules: Sigma

Sigma gives you a vendor-neutral rule format that compiles to whatever backend you use. A rule for a suspicious parent-child relationship:

```yaml
title: Office Application Spawning Shell
status: experimental
logsource:
  category: process_creation
  product: windows
detection:
  selection_parent:
    ParentImage|endswith:
      - '\winword.exe'
      - '\excel.exe'
      - '\outlook.exe'
  selection_child:
    Image|endswith:
      - '\cmd.exe'
      - '\powershell.exe'
      - '\wscript.exe'
  condition: selection_parent and selection_child
level: high
```

Convert with `sigma-cli`:

```bash
sigma convert -t splunk rules/office_spawn_shell.yml
sigma convert -t elastalert rules/office_spawn_shell.yml
```

The rule is the artefact worth keeping; the query syntax is an implementation detail.

## Proving the rule fires

This is the step most people skip. A detection you have never triggered is a hypothesis, not a detection.

Generate the behaviour deliberately. Either run the technique yourself in a contained way, or use an emulation framework that maps to ATT&CK techniques:

```bash
# After emulating the technique, check the alert
grep -i "winword.exe" /var/log/lab/sysmon.json | jq '.EventID, .EventData.Image'
```

If nothing fires, you have found either a rule bug or a coverage gap. Both are worth knowing before an incident tells you instead.

## The loop that matters

The value of this lab is not the components. It is that it makes the loop short:

1. Write a hypothesis as a Sigma rule.
2. Generate the behaviour.
3. Check whether the alert fired.
4. Fix the rule; go to 2.

Run that loop a few dozen times and you have a rule set you have actually validated, plus an accurate mental model of where your blind spots are. That model is worth more than any individual rule.
