---
title: "EDR Telemetry: What Your Defenses Actually See"
date: 2023-10-27
draft: false
tags: ["EDR", "Telemetry", "Detection Engineering", "Sysmon"]
categories: ["Blue Team"]
summary: "A look at the data EDR agents collect, and why understanding it is the difference between evasion research and guessing."
---

Most evasion content starts with a bypass technique and works backwards. That's the wrong order. The productive approach is to first understand what the sensor collects, and only then ask what it can't see.

This post is the map of the data, not a catalogue of bypasses.

## The sensor layers

An EDR agent typically sits in three places at once:

| Layer | Data source | What it captures |
| --- | --- | --- |
| Kernel | Callbacks (process, thread, image load, registry) | Process creation, module loads, driver activity |
| User mode | DLL injection, API hooking | Function arguments, command lines, .NET assembly loads |
| Network | WFP / filter driver | Connections with process attribution |

The kernel callbacks are the durable part. User-mode hooks can be removed or re-implemented; the kernel-side event registration is far harder to escape without a driver.

## The events that carry the signal

In practice, a small number of event types drive most detections:

- **Process creation (4688 / Sysmon 1)** with full command line and parent image. This is the highest-value event in the entire dataset. Lineage plus arguments answers most questions.
- **Image loads (Sysmon 7)** — unsigned or unusual DLLs, especially into sensitive processes.
- **Network connections (Sysmon 3)** — process-attributed egress.
- **Registry modifications (Sysmon 12/13)** — persistence, COM hijacking.
- **File creation (Sysmon 11)** — dropped payloads, with the option of hashing them.
- **WMI activity (Sysmon 19-21)** — a favourite for lateral movement and persistence precisely because it leaves fewer artefacts.

If you only instrument one thing, instrument process creation with arguments. Most real intrusions are legible in that stream alone.

## Why telemetry is not the same as detection

Collection without detection logic is just expensive storage. The gap shows up in three places:

**1. Baseline absence.** `powershell.exe -enc <base64>` is only suspicious if you know it isn't normal in your environment. In some shops it is the deployment mechanism.

**2. Volume.** Every sensor above produces millions of events per day on a mid-size estate. Without tuning, everything alerts and nothing is investigated.

**3. Correlation.** Individually boring events become interesting in sequence: a browser spawning a shell, which starts a PowerShell process, which makes an outbound connection to a new host. None of those three alone is a page-worthy alert.

## What this means from the red side

Knowing the data shapes how you'd approach an authorised test:

- **Process lineage is the strongest signal**, so anything that breaks parent-child relationships changes the picture. That's a real technique, but note it's a *detection-surface* change, not a magic bypass.
- **Command line arguments are logged.** Encoded or obfuscated arguments are themselves an indicator — they may dodge a string match while creating a better signal.
- **Module loads into LSASS or other sensitive processes** are heavily monitored. This is where a lot of older tooling dies.
- **The kernel callback remains.** User-mode unhooking doesn't remove the kernel event registration, which is why so much effort goes into drivers and BYOVD.

## Building the defender's version

If you're on the blue side and want to close the gap:

- **Start with Sysmon** and a maintained configuration. SwiftOnSecurity's baseline is a reasonable starting point; tune from there.
- **Shipping and retention matter more than the rule set.** A detection you can't query three weeks later isn't a detection.
- **Write detections against behaviour, not tools.** A rule matching a specific tool's filename has a shelf life measured in days.
- **Test them against the techniques you actually expect.** Adversary emulation is the cheapest way to find out that a rule never fires.

## The honest summary

Evasion is a study of what is not collected. Understanding the collection stack tells you where the blind spots plausibly are — and, more usefully for most people, tells you which of your own detections have never been validated.

The best red-teamers I know can describe the detection engineering trade-offs behind a technique before they describe the technique itself. That's not a coincidence.
