---
title: "BloodHound and SharpHound: Mapping Active Directory Attack Paths"
date: 2021-07-22
draft: false
tags: ["Active Directory", "BloodHound", "SharpHound", "Neo4j", "Privilege Escalation"]
categories: ["Red Team"]
summary: "How to collect the right AD data once, then query it as a graph instead of guessing your way to Domain Admin."
---

Before BloodHound, finding a path to Domain Admin in a mid-size domain was mostly guesswork: you enumerated ACLs by hand, eyeballed group memberships, and built the graph in your head. BloodHound moved that graph into Neo4j, where you can actually query it.

The workflow is three steps: collect, ingest, query. Most people get the first one wrong.

## Collection

`SharpHound` is the collector. Run it from a domain-joined Windows host with the credentials of a normal user:

```powershell
.\SharpHound.exe -c All --outputdirectory C:\temp\loot
```

`-c All` collects users, groups, computers, sessions, ACLs, trusts, GPOs, and OUs. It writes a zip. That collection is enough for most engagements.

Things that matter at this stage:

- **Sessions are time-sensitive.** `AdminCount` and session data go stale in hours. Collect, then analyse immediately — don't collect on Monday and query on Thursday.
- **Stealth matters on real engagements.** Session collection touches every machine. `--stealth` and `--excludedcs` reduce the footprint. For a first pass, `-c DCOnly` collects everything from the domain controllers alone: no host is touched except the DC, and you still get the full ACL picture.
- **Output format.** `--outputdirectory` keeps the zip somewhere predictable; the default drops it in the current working directory and people lose track of it.

## Ingestion

Start Neo4j, then import the zip via the BloodHound UI or the CLI:

```bash
neo4j console
```

Once imported, two built-in queries answer most of the questions:

- **Shortest Paths from Owned Principals** — set your controlled user as owned, then look at what is reachable.
- **Shortest Paths to Domain Admins** — the inverse view.

## Reading the graph

The useful primitive is not "who is a Domain Admin". It is "which edge can I traverse". The edges worth memorising:

| Edge | What it means |
| --- | --- |
| `MemberOf` | Direct group nesting |
| `AdminTo` | Local admin on that host |
| `CanRDP` / `CanPSRemote` | Remote access rights |
| `GenericAll` / `WriteDacl` | Full or ACL-writing control over the object |
| `ForceChangePassword` | Exactly what it says |
| `AllowedToDelegate` | Constrained delegation target |

An `AdminTo` edge to a workstation that a Domain Admin logs into is a path. So is `GenericAll` on a group that is nested into `Domain Admins`.

## The pattern that keeps working

The most common real-world path I find is mundane:

1. A helpdesk group has `ForceChangePassword` on a broad OU of users.
2. One of those users has an active session on a server where a tier-2 admin also has a session.
3. That admin has `AdminTo` on the domain controller.

Nothing exotic, three edges, and it's reachable from a starter account. BloodHound makes it visible in a way that scrolling through ADUC never did.

## Remediation is the hard part

Remediation is where the graph stops being a red-team toy and becomes useful to the defenders running it. Cutting one edge usually breaks the path:

- Remove `AdminTo` where it isn't operationally required.
- Break the tier model so tier-2 admins never have sessions on tier-0 assets.
- Rotate credentials for any account that appeared in a discovered path — assume compromise.

Re-run collection after remediation and confirm the path is actually gone. A surprising number of "fixed" paths survive because the change was made to a group the tool reads through a different route.
