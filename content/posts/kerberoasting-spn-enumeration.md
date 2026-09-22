---
title: "Kerberoasting: From SPN Enumeration to Offline Cracking"
date: 2020-12-08
draft: false
tags: ["Active Directory", "Kerberos", "Kerberoasting", "Impacket"]
categories: ["Pentesting"]
summary: "Why requesting service tickets for accounts with SPNs is still one of the most reliable ways to escalate from a domain user to plaintext credentials."
---

Kerberoasting has been public since 2016 and it still works, because the underlying design has not changed: any authenticated domain user can request a service ticket for any account that has a Service Principal Name, and that ticket is encrypted with the target account's password hash.

If the service account's password is weak, the hash comes off in minutes.

## What actually happens

1. You authenticate to the KDC as any domain user.
2. You request a ticket for `MSSQLSvc/db01.lab.local:1433`.
3. The KDC returns a `TGS-REP` encrypted with the service account's NTLM hash.
4. You take that ciphertext offline and crack it.

No elevated privileges anywhere. That is the whole trick.

## Finding accounts with SPNs

From a domain-joined Linux box with the credentials of a plain user:

```bash
impacket-GetUserSPNs lab.local/jdoe:'Password123!' -dc-ip 10.10.10.10 -request
```

Or without credentials at all, if you have any valid session and can use LDAP anonymously — many domains still permit it:

```bash
impacket-GetADUsers lab.local/ -dc-ip 10.10.10.10 -all
```

The output lists accounts and their service names. Two categories matter:

- **User accounts with SPNs** — the password is probably a human-chosen string. Excellent targets.
- **`krbtgt` and machine accounts** — 120-character random passwords. Ignore them.

## Requesting the tickets

`-request` makes `GetUserSPNs` fetch a crackable hash for every account it finds:

```
$krb5tgs$23$*svc_sql$LAB.LOCAL$MSSQLSvc/db01.lab.local:1433*$8f3a...
```

Save it and move to the cracking box.

## Cracking offline

Because it's Kerberos RC4 or AES, hashcat handles it directly:

```bash
# $krb5tgs$23$ = RC4 (etype 23)
hashcat -m 13100 tgs.txt rockyou.txt -r rules/best64.rule

# $krb5tgs$18$ = AES-256 (etype 18) — much slower, still worth running
hashcat -m 19700 tgs_aes.txt rockyou.txt
```

The RC4 variant is roughly 100x faster to crack than AES-256. If you only have AES tickets, mask attacks are usually a dead end — targeted wordlists built from the organisation's vocabulary work better than brute force.

A dictionary the client's own marketing site produced will beat `rockyou.txt` more often than people expect.

## Two things people get wrong

**You do not need to be an admin.** Every writeup that implies otherwise is wasting your time. Any domain user can do this.

**AES is not a fix by itself.** It raises the cracking cost, but a weak password in a wordlist still falls. The real fixes are:

- 25+ character, machine-generated service account passwords
- **Group Managed Service Accounts** — AD rotates a 240-character password every 30 days, and you cannot Kerberoast what you cannot crack
- Alert on anomalous `TGS-REQ` volume for SPN-bearing accounts; a single user requesting 40 service tickets in a minute is not normal

## Detection

From the defender's chair, the signal is in the ticket request pattern, not the tool. Event ID 4769 with encryption type `0x17` (RC4) against accounts that normally negotiate AES is the most common indicator I've seen in real environments.
