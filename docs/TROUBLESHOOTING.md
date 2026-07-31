# Troubleshooting Log

Real issues encountered while running this homelab, and how they were diagnosed.

## 1. Recurring ACPI Mutex kernel errors on boot / in `dmesg`

**Symptom:**

```
ACPI Error: Cannot release Mutex [PATM], not acquired (20230628/exmutex-357)
ACPI Error: Aborting method \_SB.PCI0.LPCB.ECDV._Q66 due to previous error (AE_AML_MUTEX_NOT_ACQUIRED) (20230628/psparse-529)
```

This appeared repeatedly in the TTY console log on the Dell Inspiron, timestamped at large uptime intervals (e.g. `[3776033.331320]`), meaning it was firing periodically rather than just at boot.

**Diagnosis:**
- This is a known class of issue on some Dell laptop firmware/BIOS ACPI tables, related to the embedded controller (`ECDV`) query method (`_Q66`) — commonly tied to laptop-specific hardware events (like lid switch, battery, or thermal polling) that don't fully apply once the machine is run headless as a server.
- It is **non-fatal** — the kernel aborts the specific ACPI method call and continues; it does not crash the system or affect the Docker services running on top.
- Confirmed non-impactful by checking `netdata` and `uptime` — the server maintained multi-week uptime (6+ weeks between reboots) with all containers healthy throughout.

**Mitigation considered:**
- Could be silenced with kernel boot parameters (e.g. `acpi_mask_gpe` / disabling exec of the offending method), but since it's cosmetic log noise on this hardware and doesn't affect stability, it was left as-is and simply documented rather than "fixed."

## 2. Running a laptop headless with the lid closed

**Symptom:** Laptop would suspend when the lid was closed, dropping the server offline.

**Fix:** Set `HandleLidSwitch=ignore` (and `HandleLidSwitchDocked=ignore`) in `/etc/systemd/logind.conf`, then restarted the `systemd-logind` service so lid state is ignored entirely.

## 3. Confirming static IP stability

**Symptom:** Needed to verify the DHCP reservation was actually holding across reboots (rather than relying on netplan).

**Fix:** Checked the assigned IPv4/IPv6 addresses on `wlp2s0` via the login MOTD banner after each reboot and cross-referenced against the router's DHCP client table to confirm consistency.
