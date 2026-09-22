---
title: "Subdomain Reconnaissance at Scale with Amass, Subfinder, and massdns"
date: 2020-09-20
draft: false
tags: ["Recon", "OSINT", "Amass", "Subfinder", "DNS"]
categories: ["Pentesting"]
summary: "A practical pipeline for building and maintaining an accurate external asset inventory before you touch a single target."
---

Every engagement starts with the same problem: you cannot test what you cannot find. Certificate transparency logs have made subdomain enumeration dramatically easier than it was five years ago, but the tooling only produces a list — turning that list into a reliable asset inventory is the actual work.

## Why passive enumeration first

Active brute force against a client's DNS is noisy and, depending on the contract, may not be authorised. Start passive:

- **Certificate Transparency** — every publicly trusted certificate is logged. `crt.sh` is the fastest source.
- **Passive DNS** — datasets like SecurityTrails and VirusTotal expose historical resolutions.
- **Search engines and code** — API keys and hostnames leak into GitHub, Pastebin, and JS bundles.

Passive sources give you breadth with zero packets sent to the target.

## The pipeline

I run two enumerators in parallel and merge, because they draw from overlapping but not identical sources.

```bash
# Passive collection
subfinder -d example.com -all -silent -o subfinder.txt
amass enum -passive -d example.com -o amass.txt

# Merge and deduplicate
cat subfinder.txt amass.txt | tr 'A-Z' 'a-z' | sort -u > all-subs.txt
wc -l all-subs.txt
```

Typical output for a mid-size organisation lands between 500 and 5,000 names. Most are dead.

## Resolving quickly

`massdns` resolves tens of thousands of names per second if you give it a decent resolver list:

```bash
# Public resolvers, one per line
curl -s https://public-dns.info/nameservers.txt | head -2000 > resolvers.txt

massdns -r resolvers.txt -t A -o S -w resolved.txt all-subs.txt
```

Then filter to the ones that actually answer:

```bash
grep -E ' A ' resolved.txt | awk '{print $1}' | sed 's/\.$//' | sort -u > live.txt
```

Any name in `resolved.txt` that only ever returns `CNAME` is worth a second look — dangling CNAMEs pointing at deprovisioned cloud resources are a classic subdomain takeover.

## HTTP probing

Resolution is not the same as serving. Probe for HTTP(S) and capture titles and status codes:

```bash
cat live.txt | httpx -silent -status-code -title -tech-detect -o http.txt
```

This is where the inventory becomes actionable: you end up with a list of hosts, their frameworks, and a status code. Login panels and default installs jump out immediately.

## Keeping it current

Point-in-time enumeration goes stale within weeks. Schedule the whole pipeline weekly, diff against the previous run, and alert on new names:

```bash
comm -13 <(sort yesterday.txt) <(sort today.txt)
```

Newly appeared subdomains are the highest-value recon finding there is — they're often staging environments that were never hardened because nobody remembers they exist.

## Scope discipline

Two rules I hold to regardless of how interesting a finding looks:

1. **Everything found must be in scope.** A subdomain resolving to a third-party SaaS provider is not your target, it's their tenant.
2. **Record where each name came from.** When a client asks how you found an internal hostname, "crt.sh, 2020-09-18" is an answer; "the tool found it" is not.
