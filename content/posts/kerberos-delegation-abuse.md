---
title: "Kerberos Delegation Abuse: Unconstrained, Constrained, and RBCD"
date: 2026-01-20
draft: false
tags: ["Kerberos", "Delegation", "Active Directory", "RBCD", "Rubeus"]
categories: ["Red Team"]
summary: "The three flavours of Kerberos delegation, what each one hands an attacker, and why resource-based delegation is both the safest and the most abusable."
---

Delegation exists because users need to access services that need to access other services on their behalf. Implementing that securely is genuinely hard, and AD has offered three different mechanisms over the years — each with a distinct failure mode.

Understanding which one is configured, and what it gives away, is a core AD assessment skill.

## 1. Unconstrained delegation

A computer or user with `TRUSTED_FOR_DELEGATION` in its UAC flags caches the TGT of anyone who authenticates to it. That cached ticket can be reused as that user, anywhere.

The practical attack: find a machine with unconstrained delegation, coerce a domain controller into authenticating to it, and harvest the DC's TGT from memory.

```powershell
# Find delegation-trusted objects
Get-ADObject -Filter { userAccountControl -band 0x80000 } -Properties userAccountControl,servicePrincipalName

# Or with PowerView
Get-DomainComputer -Unconstrained
```

Then coerce and collect:

```bash
# Force the DC to authenticate to your delegation-trusted host
python3 PetitPotam.py -d lab.local -u jdoe -p 'Password123!' deleg01.lab.local dc01.lab.local
```

```powershell
# On deleg01, as admin — dump cached tickets
Rubeus.exe dump /nowrap
```

Any domain controller TGT in that output is domain compromise. The `krbtgt` ticket lets you forge golden tickets.

**The mitigation is not configuration tuning.** Unconstrained delegation is a legacy design. Remove the flag everywhere it isn't absolutely required, and put any host that genuinely needs it in a dedicated tier.

## 2. Constrained delegation

Objects with `msDS-AllowedToDelegateTo` can delegate only to the listed SPNs. More scoped, but the classic abuse is the **S4U2Self/S4U2Proxy** combination: a service account configured to delegate to itself can obtain a service ticket *as any user* for its own SPN — effectively impersonating arbitrary users to itself.

If that account also has an SPN that accepts Kerberos authentication on a sensitive service, the delegation target list becomes an impersonation list:

```powershell
Rubeus.exe s4u /user:svc_web /rc4:<hash> /impersonateuser:administrator \
  /msdsspn:"cifs/dc01.lab.local" /altservice:ldap /ptt
```

The `/altservice` parameter is the important part: SPN suffix substitution lets a ticket issued for one service be repurposed for another on the same host. The delegation list says `cifs`; the ticket works for `ldap`.

**Detection:** look for `msDS-AllowedToDelegateTo` values that include services on tier-0 hosts. The real finding is usually a legacy service account from an era when the tier model didn't exist.

## 3. Resource-based constrained delegation (RBCD)

The newest and the most interesting. Instead of the delegating account listing its targets, the *target resource* lists who may act on its behalf, via `msDS-AllowedToActOnBehalfOfOtherIdentity`.

The security model inverts: to abuse it, you need write access to the resource's attribute, not control of the delegating account.

That means the abuse path is an ACL finding, not a delegation finding:

```powershell
# If you can write msDS-AllowedToActOnBehalfOfOtherIdentity on a computer object:
# 1. Create or use a machine account you control
# 2. Set the target's RBCD attribute to point at your controlled account
# 3. S4U2Self + S4U2Proxy to obtain a ticket as any user to that computer
```

The reason this matters: **any account with `GenericWrite` on a computer object can establish RBCD.** And `GenericWrite` on machine objects is extremely common — for helpdesk groups, for automation accounts, for anyone who was ever granted "manage this server" without anyone thinking through what that means.

It also works with a machine account you can create yourself. The default `MachineAccountQuota` of 10 lets any domain user create up to ten computer objects, which is exactly what you need:

```bash
impacket-addcomputer lab.local/jdoe:'Password123!' -computer-name 'FAKE$' -computer-pass 'Passw0rd123!'
```

Then configure RBCD from your controlled machine account onto the target and request the ticket.

**Mitigation:** set `MachineAccountQuota` to 0, and audit `GenericWrite`/`WriteDacl` on computer objects as seriously as you audit them on user objects. The attribute is an ACL surface, so it needs the same review cadence.

## Reading the environment

A practical first pass on any AD assessment:

```powershell
Get-DomainComputer -Unconstrained | select name
Get-DomainUser -TrustedToAuth | select name, msds-allowedtodelegateto
Get-DomainComputer -TrustedToAuth | select name, msds-allowedtodelegateto
```

Then, separately, search for `GenericWrite` on computer objects in BloodHound. The RBCD path rarely shows up in a delegation report, because nothing in the delegation configuration is wrong — the vulnerability is the ACL.

## The pattern

Across all three mechanisms the finding is the same shape: a legitimate feature granting a broad, durable impersonation capability, configured by someone who needed it to work and had no reason to think about the blast radius.

The mitigations differ, but the review question doesn't. For every object that can act as, or on behalf of, another identity — who can modify that configuration, and what does the impersonation actually reach?
