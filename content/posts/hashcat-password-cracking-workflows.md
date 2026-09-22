---
title: "Password Cracking at Scale: Hashcat Modes, Rules, and Masks"
date: 2024-07-30
draft: false
tags: ["Hashcat", "Password Cracking", "Wordlists", "GPU"]
categories: ["Pentesting"]
summary: "How to pick the right attack mode for a given hash set, and why the wordlist matters far less than the rule file."
---

Cracking hashes is a resource allocation problem. Given a hash set and a time budget, you want the highest probability of recovery per GPU-hour. That means choosing an attack mode deliberately rather than running `rockyou.txt` and hoping.

## Know what you're cracking first

Hash type determines everything downstream. Two rules:

1. **Get the mode right.** RC4 Kerberos tickets (`-m 13100`) crack orders of magnitude faster than AES (`-m 19700`). Running the wrong mode wastes the entire budget.
2. **Check for the easy wins.** A dump often contains NTLM (`-m 1000`), NetNTLMv2 (`-m 5600`), and Kerberos tickets (`-m 13100`) mixed together. Sort and attack separately — the cheap formats first.

```bash
hashcat --example-hashes | grep -A3 -i 'kerberos 5 tgs'
```

## Attack mode priority

The ordering I use, cheapest first:

**1. Straight dictionary with the organisation's own vocabulary.** This beats generic lists more often than people expect. Build a list from:

```bash
# Company-specific terms, then mutate
cat site-scrape.txt employees.txt products.txt | tr 'A-Z' 'a-z' | sort -u > org.txt
hashcat -m 1000 ntlm.txt org.txt
```

**2. Dictionary + rules.** The rule file is where the value is. A single good wordlist with `best64.rule` outperforms ten wordlists with no rules.

```bash
hashcat -m 1000 ntlm.txt rockyou.txt -r rules/best64.rule
hashcat -m 1000 ntlm.txt rockyou.txt -r rules/dive.rule   # more aggressive
```

**3. Mask attack once you know the policy.** If AD enforces "12 characters, one of each class", brute-forcing the full space is pointless — but a policy-aware mask can be viable:

```bash
# Uppercase + lowercase + digits + symbol, 8 chars — exhaustive
hashcat -m 1000 ntlm.txt -a 3 ?u?l?l?l?l?l?l?d

# Incremental
hashcat -m 1000 ntlm.txt -a 3 --increment --increment-min 8 ?a?a?a?a?a?a?a?a
```

**4. Combinator and hybrid.** `-a 1` concatenates two dictionaries; `-a 6`/`-a 7` append or prepend a mask to a dictionary. Good for `Summer2024!`-style patterns.

## Building a rule that fits the target

Rather than picking from the packaged rule sets blindly, generate rules from any plaintext you already recovered. Once you crack one password in a dump, its structure tells you about the rest:

```bash
# Convert a cracked set back into rules
hashcat --stdout wordlist.txt -r rules/best64.rule | sort -u > expanded.txt
```

Then feed the expanded list back in. Iterating this way — crack some, learn the pattern, target the rest — recovers more than any single long run.

## Practical notes

- **Session management matters.** Cracking runs for days. Always use `-s`, and use `--restore` rather than restarting from zero.
- **Monitor the temperature before anything else.** `nvidia-smi -q -d TEMPERATURE`. A card at 85°C that throttles is slower than a card at 70°C running a smaller mask.
- **Watch the efficiency graph in real time.** `--status --status-timer=30`. If the candidate rate collapses partway through a run, the mask is too large for the remaining budget — kill it and re-plan.
- **Use `--username` when the dump has them.** Saves a surprising amount of time on `user:hash` formats.
- **Potfile discipline.** Keep the potfile between runs; it prevents redoing work and accumulates the vocabulary you actually need.

## What actually limits recovery

In real engagements, the binding constraint is almost never GPU throughput. It is:

- **Password policy.** A 14-character minimum with a breach-list check removes most of the low-hanging fruit.
- **Format.** Managed service accounts and modern Kerberos encryption remove the fast attack entirely.
- **Time.** The client will not wait a week for a mask run that might not succeed.

Which is the pragmatic version of the defensive advice: you don't need to defeat cracking, you need to make it not worth the attacker's time budget. Length plus a breach-list check does that more reliably than a symbol requirement.
