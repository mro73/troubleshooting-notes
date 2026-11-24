# Troubleshooting: Lubuntu 26.04 live session freezes on a Lamina tablet (Cherry Trail)

Nov 24, 2026 · @Klaudia

## Summary

A plain Lubuntu 26.04.1 image written 1:1 to a pendrive freezes on the Lamina tablet or ends in a kernel panic within a few minutes of starting the live system.

The most likely cause is a race condition between putting the PCIe link to the Intel 3165 Wi-Fi card to sleep (ASPM L1) and the deep sleep states of the Cherry Trail processor.

Workaround for the plain image: setting the kernel parameter `pcie_aspm=off` (3 boots out of 3 succeeded) or `intel_idle.max_cstate=1` (also 3 out of 3). A pendrive made with the bootia32-usb app (a FAT32 partition with the ISO files copied onto it) boots the same image with no parameters, 3 times out of 3. The system installed from a pendrive created with this app works stably, including after 2 hours of suspend.

## Environment and symptoms

The problem concerns a single tablet, but its hardware is typical of the whole Intel Cherry Trail family.

| Item | Value |
| --- | --- |
| Tablet | Lamina LAMINALTPRO1024 / SMB-H1005F |
| Firmware | Insyde EFI v2.4, 64-bit UEFI (`fw_platform_size` = 64), BIOS LTPRO1024.I22K6.9.01 from 2019 |
| CPU / GPU | Intel Cherry Trail (Cherry View, GPU id 22b0) |
| Wi-Fi / Bluetooth | Intel Wireless-AC 3165 (rev 91), PCIe `0000:01:00.0`, iwlwifi driver, plus Intel BT |
| Storage | eMMC 58 GB (mmcblk0) |
| Image | Lubuntu 26.04.1; kernel in live 7.0.0-30-generic, in the installed system 7.0.0-43-generic |

Symptoms of the plain image written to a pendrive:

- The boot splash screen hangs for a very long time, even around 10 minutes. Rarely a kernel panic also appears: `Timeout: Not all CPUs entered broadcast exception handler`. This is a hardware error of the machine check type.
- Choosing safe graphics (`nomodeset`) in GRUB lets the live system start.
- Changing the kernel parameters to `pcie_aspm=off` or `intel_idle.max_cstate=1` before `---` also lets the system start.

## Diagnosis timeline

Twelve steps, from the first tests on Fedora to comparing the ASPM settings. Each step changes one thing, and the result shows what can be ruled out or narrowed down.

| No. | Test | Result | What follows |
| --- | --- | --- | --- |
| 1 | Fedora LXQt Live 45, plain image | Boots to the desktop | Fedora ships its own ia32 files in the image and does not need the bootia32.efi blob |
| 2 | Output of `fw_platform_size`, `dmesg` (EFI) | 64, Insyde EFI v2.4 | The firmware is 64-bit, so a 32-bit blob is not needed here |
| 3 | Plain Lubuntu 26.04.1, no parameters | Long splash, kernel panic or freeze, 3 attempts out of 3 | The problem is reproducible |
| 4 | Same plus `intel_idle.max_cstate=1` | Boots, parameter confirmed in `/proc/cmdline` | Lead towards the deep C-states of the CPU (a known Cherry Trail bug, kernel bugzilla 109051) |
| 5 | Safe graphics (`nomodeset`) | Boots, stable for over 20 min | Changing the order and pace of the boot also avoids the problem |
| 6 | Pendrive from the bootia32-usb app, same ISO | Boots 3 times out of 3 with no parameters | The file and partition layout on the pendrive affects the result |
| 7 | Installed system, s2idle suspend for 2 hours | Resumes, Wi-Fi and touch work | The installed system is stable |
| 8 | `casper.log` and `boot.log` from the pendrive after the freezes | Last entries: start of `systemd-rfkill` (session .0), start of iwlwifi and Bluetooth (session .3) | First lead towards the radio; the `writable` partition is created in only one session, so it is not the cause |
| 9 | Plain image plus `modprobe.blacklist=iwlwifi,btusb` | Froze (1 attempt) | The Wi-Fi and Bluetooth driver itself is not the culprit |
| 10 | Plain image plus `pcie_aspm=off` | Boots 3 times out of 3 | Turning off power saving on the PCIe bus removes the problem |
| 11 | Plain image plus `pcie_aspm.policy=performance` | Boots (3 attempts) | Consistent with step 10 |
| 12 | Installed system: `lspci -vv -s 01:00.0`, ASPM policy | `01:00.0` is the Wi-Fi 3165; ASPM L1 enabled, L1.1 and L1.2 substates disabled, policy `default` | ASPM is enabled there and the system works, so L1 alone is not enough to cause the failure; the moment it kicks in matters |

### Sessions saved on the pendrive (steps 3 and 8)

After every boot of the plain image, the live system saved its logs on the pendrive's `writable` partition, in a folder named `install-logs-2026-11-16.N`. In the text, "session .0", ".3" and so on mean the `N` at the end of that name. The pendrive made with the app had no such logs.

| Folder | Time by the tablet clock (start → last change in `log`) | Kernel parameters (from how the tests went, to be confirmed in the log) | `writable` partition created in this session | Last traces in the logs | Result |
| --- | --- | --- | --- | --- | --- |
| `.0` | 01:23 → 01:27 (about 4 min) | None (first attempt, plain image) | Yes (6 `GPT` entries in `casper.log`: `GPT PMBR size mismatch`, `backup GPT table is not on the end of the device`) | Last lines of `boot.log`: `Listening on systemd-rfkill.socket`, `Starting systemd-rfkill.service` | Freeze / kernel panic |
| `.1` | 12:29 → 13:15 (about 46 min) | `intel_idle.max_cstate=1` | No (0 `GPT` entries) | Full logs (`syslog`, `kern.log`, `Xorg.0.log`); `boot.log` ends with `Finished systemd-user-sessions`, `Starting plymouth-quit`, `Starting setvtrgb` | Full boot to the desktop |
| `.2` | 15:20 → 15:22 (about 2 min) | None | No (0 `GPT` entries) | `boot.log` empty (0 bytes) | Freeze |
| `.3` | 15:57 → 15:58 (about 1 min) | None | No (0 `GPT` entries) | `boot.log` empty; last kernel messages in the journal: start of iwlwifi and Bluetooth | Freeze |
| `.4` | 16:31 → 16:45 (about 14 min) | Safe graphics (`nomodeset`) | No (0 `GPT` entries) | Full logs; `boot.log` ends the same way as in `.1` (`Starting setvtrgb`) | Full boot to the desktop |

The times are a rough lower bound: the modification time of the `log` folder changes only when an entry is created or removed in it, not on every write, so a session may have lasted longer.

Conclusions from the table: all three boots without parameters (`.0`, `.2`, `.3`) ended in a freeze, while both full boots (`.1`, `.4`) had a parameter or safe graphics. The `writable` partition was created only in session `.0`, so creating it is not the cause: sessions `.2` and `.3` froze without it. The freezes of `.0` and `.3` end at the start of the radio, which was the first lead towards the Wi-Fi card. The tests with `modprobe.blacklist` and `pcie_aspm` (steps 9–11) were done on a different pendrive, so they are not in this table.

## Rejected or weakened hypotheses

Some of the first suspicions turned out to be wrong; each rejection narrowed the search.

- **The bootia32.efi blob solves the problem.** Rejected: the firmware is 64-bit, and a pendrive with just the plain image on Fedora boots without the blob.
- **USB load or a particular pendrive is to blame.** Rejected: the same pendrive worked in both modes, plain and from the app.
- **The kernel version is to blame (-30 in live, -43 after installation).** Weakened: the same ISO with kernel -30 boots from the pendrive made with the app.
- **Creating the `writable` partition on first boot is to blame.** Rejected: sessions .2 and .3 did not have that step and still froze.
- **The Wi-Fi or Bluetooth driver is to blame.** Weakened: blacklisting `iwlwifi` and `btusb` did not help. The radio lead survived in another form: what matters is the power state of the PCIe link to the card, which the kernel sets before the driver is loaded, not the driver itself.

## Conclusion and workarounds

The best-fitting explanation: the Wi-Fi 3165 card drops into the ASPM L1 power-saving state, and in the same fraction of a second the Cherry Trail processor is in a deep C-state. When both wake up in an unlucky order, the processor reports a machine check and the system stops. The live system starts a lot of parallel tasks (reading from USB, loading modules, starting Wi-Fi and Bluetooth), so the race is most likely to hit there.

This is consistent with all the tests: `pcie_aspm=off` blocks the link from sleeping, `intel_idle.max_cstate=1` blocks the deep sleep of the processor, and safe graphics changes the boot timing.

| Situation | What to do |
| --- | --- |
| A pendrive for this tablet | Use the bootia32-usb app: a GPT partition with FAT32 (`parted mkpart ESP fat32 1MiB 100%`), ISO files copied onto it; boots with no parameters |
| A plain image written with dd | Add `pcie_aspm=off` to the `linux` line, before `---` |
| An alternative for the plain image | `intel_idle.max_cstate=1` or safe graphics |
| System installed on the eMMC | No change needed, it works stably |

A note about the app: it helps thanks to the file and partition layout, not thanks to the bootia32.efi blob, because this tablet has 64-bit firmware. Why this layout avoids the race is a hypothesis (a different rhythm of reads from USB changes the boot timing), not a measured fact.

## Open questions

The result is solid in practice, but a few things remain unconfirmed.

- **Why the FAT32 layout avoids the problem.** The hypothesis about a different rhythm of reads from USB has not been measured.
- **The exact moment of the failure.** After a hard freeze the logs break off earlier than the failure itself, because they did not get written in time. A `pstore` record or a serial console could show it, but neither was tried.
- **Sample size.** The freezes are random and the number of attempts is small (3 per variant), so single results may be luck.

## Lessons for the future

The biggest gains came from changing one thing at a time and comparing against a working case.

- Start from a fact about the hardware (`fw_platform_size`) before assuming the problem lies in the firmware architecture.
- Repeat the test; with a random failure, one successful attempt proves nothing.
- Look for the difference between the working and the non-working case (same image, two layouts on the pendrive) instead of guessing in a vacuum.
- Collect the logs from the pendrive after every attempt, and remember that after a hard freeze they are incomplete.
- Openly drop your own hypotheses when the data contradicts them (the blob, the kernel version, the USB load).
- In the write-up, separate what was measured from what is an inference from circumstantial evidence.

## Appendix: commands and parameters

Commands used in the diagnosis, with a short note on what they check.

| Command or parameter | What it was used for |
| --- | --- |
| `cat /sys/firmware/efi/fw_platform_size` | Whether the UEFI firmware is 32-bit or 64-bit |
| `cat /proc/cmdline` | Which kernel parameters are really in effect |
| `journalctl -D <directory> -k` | Reading the kernel logs saved on the pendrive after a freeze |
| `lspci -s 01:00.0` | Name of the device on the PCIe bus (here the Wi-Fi 3165) |
| `sudo lspci -vv -s 01:00.0 \| grep -iE 'ASPM\|LnkCtl'` | ASPM and L1 substate status for the card |
| `cat /sys/module/pcie_aspm/parameters/policy` | Current ASPM policy in the kernel |
| `sudo dmesg \| grep -i aspm` | What the kernel reported about ASPM at boot |
| `pcie_aspm=off` | Turns off ASPM management on the kernel side |
| `pcie_aspm.policy=performance` | Forbids the use of L0s and L1 |
| `intel_idle.max_cstate=1` | Limits the deep sleep states of the processor |
| `modprobe.blacklist=iwlwifi,btusb` | Blocks the Wi-Fi and Bluetooth drivers |
| `nomodeset` | The equivalent of safe graphics |

How to add a parameter in GRUB on the pendrive: in the menu press `e`, add the parameter on the line starting with `linux`, **before** the `---` characters, then boot with F10 or Ctrl+X. The change applies to a single boot only.
