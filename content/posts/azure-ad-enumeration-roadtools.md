---
title: "Enumerating Azure AD with ROADtools and AzureHound"
date: 2023-02-18
draft: false
tags: ["Azure AD", "Entra ID", "ROADtools", "AzureHound", "Cloud"]
categories: ["Red Team"]
summary: "Mapping a Microsoft cloud tenant from a single set of valid credentials, and reading the result as an attack graph."
---

Once an organisation moves identity to Azure AD, the reconnaissance model changes. There is no LLMNR to poison and no SMB to relay. What you get instead is a Graph API that is generous about what an authenticated user can read.

If you have any valid credential — a low-privileged user, or a refresh token from a compromised laptop — you can map a surprising amount of the tenant.

## ROADrecon: the dump

`ROADrecon` (`roadrecon`) authenticates, then pulls a full snapshot of the directory via Graph into a local database.

```bash
pipx install roadrecon

# Device code flow — works without a password, ideal with a stolen session
roadrecon auth --device-code

# Password flow
roadrecon auth -u jdoe@corp.onmicrosoft.com -p 'Password123!'

# Gather everything the account can read
roadrecon gather

# Browse it
roadrecon gui
```

The GUI is a local Flask app that lets you pivot by user, group, application, and role. What I look for in the dump:

- **Global Administrators.** Obvious, but note *which* accounts are cloud-only versus synced, and whether any are service principals.
- **Applications with high-privilege Graph permissions.** An app registration with `Directory.ReadWrite.All` and a password credential that never expires is a permanent backdoor if you can read the secret.
- **Consent grants.** A user consenting to a third-party app can hand over mailbox or file access.
- **Conditional Access policies** — they define what you *can't* do, so read them early rather than discovering them by getting blocked.

## AzureHound: the graph

`AzureHound` collects specifically to feed BloodHound, which is where the value compounds: you get the same graph queries for cloud as for on-prem AD.

```bash
azurehound -u "jdoe@corp.onmicrosoft.com" -p 'Password123!' \
  list --tenant "contoso.onmicrosoft.com" -o azure.json
```

Import `azure.json` into BloodHound (or BloodHound CE). The edges that matter:

| Edge | Meaning |
| --- | --- |
| `AZGlobalAdmin` | Global Administrator |
| `AZPrivilegedRoleAdmin` | Can assign any directory role |
| `AZOwns` | Full control over the object |
| `AZResetPassword` | Can reset the target's password |
| `AZAddSecret` | Can add a credential to an app registration |
| `AZMGAddSecret` | Can add a secret to a service principal |

That last pair is the cloud equivalent of a DACL write. It doesn't look like privilege escalation in the UI, which is why it survives review.

## The hybrid bridge

The most valuable finding in a hybrid tenant is the path from cloud to on-premises. Two common ones:

1. **Azure AD Connect.** The sync account has `Replicate Directory Changes` on-premises. Compromising it, or the server running it, gives you on-prem AD.
2. **A synced Global Admin.** If an account is both a cloud Global Admin and privileged on-premises, cloud compromise is domain compromise.

Read the sync configuration where you can:

```bash
roadrecon plugin aadconnect
```

The plugin extracts the AAD Connect configuration including the service account name — a direct pointer at the on-premises target.

## Defensive notes

- **Separate cloud and on-prem privileged accounts.** The hybrid bridge is the whole ballgame.
- **Review app registrations on a schedule.** Credential-bearing apps with broad Graph scopes and multi-year expiry are the cloud equivalent of a service account with a password from 2016.
- **Restrict who can consent to applications**, or at minimum require admin consent for anything requesting mail or file scopes.
- **Monitor for enumeration.** A single account issuing thousands of Graph reads in a short window is unusual and easy to alert on — it is the cloud counterpart to the LDAP enumeration burst you'd catch on-premises.
