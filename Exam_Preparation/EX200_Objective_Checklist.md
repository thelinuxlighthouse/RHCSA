# EX200 RHEL 10 Objective Coverage Checklist

Baseline checked against the official Red Hat EX200 page on July 19, 2026. The wording below is condensed for study. Recheck the official page when booking the exam.

## Essential tools

- [ ] Enter shell commands correctly — Chapter 1, Sections 1–2.
- [ ] Redirect stdin, stdout, and stderr and build pipelines — Chapter 1, Section 3.
- [ ] Analyze text with grep and regular expressions — Chapters 1 and 12.
- [ ] Connect to remote systems through SSH — Chapter 1, Section 9.
- [ ] Log in and switch identities on a multi-user system — Chapter 1, Section 9.
- [ ] Create and extract tar archives with gzip and bzip2 — Chapter 1, Section 8.
- [ ] Create and edit text files — Chapter 1, Sections 2 and 5.
- [ ] Create, delete, copy, and move files and directories — Chapter 1, Section 2.
- [ ] Create hard and symbolic links — Chapter 1, Section 6.
- [ ] Read and change ordinary owner, group, other, read, write, and execute modes — Chapter 1, Section 7.
- [ ] Find and use man, info, and package documentation — Chapter 1, Section 10.

## Software

- [ ] Configure RPM repository access — Chapter 2, Section 3.
- [ ] Install and remove RPM packages — Chapter 2, Sections 4–5.
- [ ] Configure Flatpak remote access — Chapter 2, Sections 7–8.
- [ ] Install and remove Flatpak applications — Chapter 2, Sections 8–10.

## Shell scripts

- [ ] Make decisions with if, test, and brackets — Chapter 3, Section 4.
- [ ] Loop over lists, arguments, files, and command data — Chapter 3, Section 5.
- [ ] Process positional arguments — Chapter 3, Section 3.
- [ ] Use command output inside a script — Chapter 3, Sections 2 and 9.

## Running systems

- [ ] Boot, reboot, shut down, halt, and power off normally — Chapter 4, Section 1.
- [ ] Boot or isolate to different targets — Chapter 4, Section 2.
- [ ] Interrupt boot for authorized recovery — Chapter 4, Section 3.
- [ ] Identify CPU- and memory-heavy processes and end them — Chapter 4, Sections 4–5.
- [ ] Adjust process scheduling preference — Chapter 4, Section 6.
- [ ] Select and verify TuneD profiles — Chapter 4, Section 7.
- [ ] Find and interpret journal and log data — Chapter 4, Section 9.
- [ ] Preserve system journal data across reboot — Chapter 4, Section 10.
- [ ] Start, stop, restart, reload, and inspect network services — Chapter 4, Section 8.
- [ ] Transfer files securely — Chapter 4, Section 11.

## Local storage

- [ ] List, create, and remove GPT partitions — Chapter 5, Sections 2–3.
- [ ] Create and remove LVM physical volumes — Chapter 5, Sections 5 and 9.
- [ ] Create, extend, and remove volume groups — Chapter 5, Sections 5 and 9.
- [ ] Create and remove logical volumes — Chapter 5, Sections 5 and 9.
- [ ] Persist mounts by UUID or label — Chapter 5, Section 6.
- [ ] Add partitions, logical volumes, and swap without damaging existing storage — Chapter 5, Sections 7–8.

## File systems

- [ ] Create, mount, unmount, and use VFAT, ext4, and XFS — Chapter 6, Sections 1–5.
- [ ] Mount and unmount NFS — Chapter 6, Section 6.
- [ ] Configure autofs — Chapter 6, Section 7.
- [ ] Extend an LV and its file system — Chapter 6, Section 8.
- [ ] Diagnose and correct file access problems — Chapter 6, Section 9.

## Deployment and maintenance

- [ ] Schedule work with at, cron, and systemd timers — Chapter 7, Sections 1–4.
- [ ] Manage service runtime and boot state — Chapter 7, Section 5.
- [ ] Choose a persistent default boot target — Chapter 7, Section 6.
- [ ] Configure and verify a time-synchronization client — Chapter 7, Section 7.
- [ ] Install and update packages from CDN, remote repositories, and local RPM files — Chapters 2 and 7.
- [ ] Inspect and modify the boot loader and persistent kernel arguments — Chapter 7, Sections 9–12.

## Basic networking

- [ ] Configure persistent IPv4 and IPv6 — Chapter 8, Sections 1–6.
- [ ] Configure hostname and name resolution — Chapter 8, Section 7.
- [ ] Make network profiles and services start automatically — Chapter 8, Section 8.
- [ ] Restrict network access with firewalld — Chapter 8, Sections 9–13.

## Users and groups

- [ ] Create, delete, and modify local users — Chapter 9, Sections 1–5.
- [ ] Change passwords and password aging — Chapter 9, Section 6.
- [ ] Create, delete, modify, and populate local groups — Chapter 9, Sections 7–8.
- [ ] Configure privileged access with sudo — Chapter 9, Section 9.

## Security

- [ ] Configure firewalld rules — Chapters 8 and 10.
- [ ] Manage default permissions with umask and special directory modes — Chapter 10, Sections 2–3.
- [ ] Configure SSH public-key authentication — Chapter 10, Section 4.
- [ ] Set SELinux enforcing or permissive mode now and at boot — Chapter 10, Section 7.
- [ ] Identify SELinux file and process contexts — Chapter 10, Section 8.
- [ ] Restore default contexts and define persistent mappings — Chapter 10, Sections 9–10.
- [ ] Manage SELinux network port types — Chapter 10, Section 11.
- [ ] Inspect and persist SELinux booleans — Chapter 10, Section 12.

## Persistence rule

- [ ] Reboot a practice machine and prove every requested setting remains correct — Chapter 12, Sections 15–17.

## Supplementary material

Podman administration is in Chapter 11. It is useful RHEL 10 knowledge but is not listed in the current published EX200 objective set.

Official baseline: [Red Hat EX200 exam page](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam)

