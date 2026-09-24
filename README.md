# win11-iot-ltsc-libvirt

Unattended install of **Windows 11 IoT Enterprise LTSC (Evaluation)** as a
libvirt/QEMU/KVM virtual machine, with working virtio disk/network, TPM 2.0,
UEFI Secure Boot, and a functioning QEMU Guest Agent.

Managed entirely through `virsh`/`virt-install` — no manual
`qemu-system-x86_64` invocations.

## Why this edition

- **IoT Enterprise LTSC** is built on the Enterprise SKU, so it supports
  domain-join and Group Policy (unlike Home/Pro-consumer builds).
- **LTSC** ships without Store, Cortana, Copilot, or forced feature updates —
  a stable, minimal, reproducible base without needing a third-party
  "debloated" image.
- It runs on the same NT kernel as regular Windows 11, so the same TPM 2.0 /
  Secure Boot install requirements apply.

## Prerequisites

Tested on Arch Linux (CachyOS). Package names below are for `pacman`; the
underlying tools exist on most distros.

| Tool | Package | Purpose |
|---|---|---|
| `qemu-system-x86_64`, `qemu-img` | `qemu-full` (or `qemu-desktop`) | VM emulation, KVM acceleration |
| `virsh`, `virt-install`, `virt-xml` | `libvirt` | Domain/storage management |
| OVMF firmware (incl. Secure Boot variant) | `edk2-ovmf` | UEFI firmware for the guest |
| `swtpm` | `swtpm` | Software TPM 2.0, spawned automatically by libvirt per-VM |
| `xorriso` | `libisoburn` (usually preinstalled) | Building the `autounattend.iso` |
| `ntfs-3g` | `ntfs-3g` | Needed only for the offline guest-agent fix below |
| `guestfish`, `hivexregedit` | `libguestfs` | Needed only for the offline guest-agent fix below |
| `msiextract` | `msitools` | Needed only for the offline guest-agent fix below |

You also need `libvirtd` running and your user in the `libvirt` group (so
`virsh`/`virt-install` work against `qemu:///system` without `sudo`).

## 1. Get the install media

- **Windows 11 IoT Enterprise LTSC (Evaluation) ISO** — official Microsoft
  Evaluation Center:
  https://www.microsoft.com/en-us/evalcenter/download-windows-11-iot-enterprise-ltsc-eval
- **virtio-win driver ISO** (official upstream, Fedora/Red&nbsp;Hat project):
  https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

Both are single-edition images (`wimlib-imagex info install.wim` will show
exactly one image, `IoTEnterpriseSEval`), so no product key is needed anywhere
in the answer file.

## 2. Build the answer file

`autounattend.xml` in this repo drives a fully unattended install:

- GPT/UEFI partitioning (EFI System Partition + MSR + Windows partition —
  **not** the legacy MBR/single-partition layout some older unattend
  templates use, which fails on a UEFI target)
- injects the `viostor` (disk) and `NetKVM` (network) virtio drivers during
  WinPE, from the second attached CD-ROM (`E:` — see note in the XML about
  drive-letter ordering)
- creates a local administrator account, skips all OOBE/EULA/MSA screens
- installs the QEMU Guest Agent MSI as a first-logon command (see the
  **Guest agent doesn't come up** section below for why this alone isn't
  enough)

**Before building the ISO, open `autounattend.xml` and replace the
placeholder password** (`CHANGE-ME-Str0ng-Pa55w0rd!`) with your own. Never
commit a real password back into this file.

```sh
xorriso -as mkisofs -o autounattend.iso -V AUTOUNATTEND -J -R autounattend.xml
```

(A prebuilt `autounattend.iso` is included in this repo for convenience, but
it was built from the placeholder password above — rebuild it after editing
the XML.)

## 3. Create the VM

```sh
virt-install \
  --connect qemu:///system \
  --name win11-iot-ltsc \
  --os-variant win11 \
  --memory 4096 \
  --vcpus 2 \
  --cpu host-passthrough \
  --disk path=/var/lib/libvirt/images/win11-iot-ltsc.qcow2,size=80,bus=virtio,format=qcow2 \
  --disk path=/path/to/win11-iot-ltsc-eval.iso,device=cdrom,bus=sata,boot.order=1 \
  --disk path=/path/to/virtio-win.iso,device=cdrom,bus=sata \
  --disk path=/path/to/autounattend.iso,device=cdrom,bus=sata \
  --network network=default,model=virtio \
  --graphics spice \
  --video qxl \
  --tpm backend.type=emulator,backend.version=2.0,model=tpm-crb \
  --boot uefi \
  --features smm.state=on \
  --noautoconsole
```

`--os-variant win11` (via `osinfo-db`) makes `virt-install` pick a
Secure-Boot-capable OVMF firmware descriptor and `q35` machine type
automatically — you don't need to hand-select OVMF files.

**Always pass `--connect qemu:///system` explicitly.** Without it,
`virt-install`/`virsh` can silently fall back to a per-user `qemu:///session`
instance that lacks permission to write into `/var/lib/libvirt/images/`,
producing a confusing "Permission denied" from `qemu-img create`.

### Gotcha: stuck at "Please select boot device"

If the very first boot hangs at OVMF's boot menu instead of proceeding to
Windows Setup, it's almost always because a stray keypress (e.g. sent via
`virsh send-key` while probing the VM) landed during POST and triggered the
interactive picker. Select **UEFI QEMU DVD-ROM** and press Enter within a
couple of seconds — Windows Setup's own "Press any key to boot from CD or
DVD..." prompt has a short timeout too, so you may need to send Enter two or
three times in quick succession.

### Gotcha: disk paths under your home directory aren't visible to QEMU

If your `$HOME` is `700` (owner-only), the `libvirt-qemu` user QEMU runs as
cannot traverse into it — even if the ISO file itself is world-readable, the
whole path must be traversable. `virt-install` warns about this but still
lets you proceed, and the VM will then fail to start. Either:

- keep install media under `/var/lib/libvirt/images/` (or another
  libvirt-accessible pool) directly, or
- upload it there via `virsh vol-upload` — goes through the privileged
  libvirt daemon, so it works without `chmod`/`setfacl`-ing your home
  directory:
  ```sh
  virsh --connect qemu:///system vol-create-as default my.iso <size-in-bytes> --format raw
  virsh --connect qemu:///system vol-upload --pool default my.iso /path/to/my.iso
  ```

## 4. Guest agent doesn't come up — why, and the fix

After install, `virsh qemu-agent-command <vm> '{"execute":"guest-ping"}'`
will keep failing with "QEMU guest agent is not connected", for two
independent reasons:

1. **`virt-install` doesn't add a virtio-serial channel device by default.**
   Add it, then reboot the guest so Windows enumerates the new device:
   ```sh
   virt-xml --connect qemu:///system win11-iot-ltsc \
     --add-device --channel type=unix,target.type=virtio,target.name=org.qemu.guest_agent.0 --update
   ```
2. **The `vioserial` driver for that new device was never injected** — the
   `DriverPaths` in `autounattend.xml` only cover `viostor`/`NetKVM`
   (needed just to get Setup running), not `vioserial`. Windows shows the
   new device as an unresolved "PCI Simple Communications Controller" and,
   critically, **does not automatically retry the driver search on
   subsequent boots** once it's cached a "no driver found" result.
3. Even after fixing the driver, the **`FirstLogonCommands` step that
   installs the guest agent MSI never actually runs**, because Windows only
   processes `HKLM\...\RunOnce` entries at an *interactive logon* — and
   this answer file deliberately doesn't enable `AutoLogon`, so nothing
   ever triggers one.

The most reliable fix is entirely offline (VM powered off), using
`guestfish`+`hivexregedit` against the qcow2 disk directly — no need to
type a password through `virsh send-key` (fragile with non-US keyboard
layouts, and impossible to verify since the field is masked):

1. Extract `qemu-ga.exe` and its DLLs from `virtio-win.iso:\guest-agent\qemu-ga-x86_64.msi`
   with `msiextract`, and register it as a Windows service
   (`SYSTEM\ControlSet001\Services\QEMU-GA`, `Start=2` AUTO_START,
   `LocalSystem`) via `hivexregedit --merge`.
2. Copy `vioserial\w11\amd64\vioser.{inf,sys,cat}` from `virtio-win.iso`
   into a plain folder inside the guest filesystem (e.g. `C:\drivers\vioserial\`
   — **not** directly into `C:\Windows\INF\`, which `pnputil /add-driver`
   refuses as a source location).
3. Delete the stale device enumeration key so Windows treats the
   virtio-serial controller as new hardware again:
   `SYSTEM\ControlSet001\Enum\PCI\VEN_1AF4&DEV_1043&SUBSYS_...` (find the
   exact device ID via `hivexregedit --export ... '\ControlSet001\Enum\PCI'`
   — `DEV_1043` is virtio-console/serial in the modern virtio 1.0 ID scheme;
   `1041`=net, `1042`=block, `1045`=balloon).
4. Set a **temporary** `AutoAdminLogon=1` (+ `DefaultUserName`/`DefaultPassword`,
   `AutoLogonCount=1`) in the `SOFTWARE` hive, plus a `RunOnce` entry pointing
   at a small `.cmd` script that runs `pnputil /add-driver ... /install` and
   `net start QEMU-GA`, then deletes `AutoAdminLogon`/`DefaultPassword` itself
   as its last step (self-cleanup, no permanent passwordless login left behind).
5. Boot the VM once — it logs in automatically, the script fixes the driver
   and starts the service, then disables autologon again on its own.

Two more pitfalls you'll hit doing this:

- **After a `virsh destroy` (forced power-off), `ntfs-3g` refuses read-write
  mounts** ("the disk contains an unclean file system") until Windows has
  booted once and been shut down *cleanly*. Always prefer `virsh shutdown`;
  only force-destroy as a last resort, and boot the VM once more afterwards
  before attempting another offline edit.
- **`C:\Windows\INF` is a destination, not a valid `pnputil /add-driver`
  source.** Stage driver files somewhere else first.

## Security notes

- Never commit a real admin password. The XML in this repo ships with an
  obvious placeholder (`CHANGE-ME-Str0ng-Pa55w0rd!`) for exactly this reason.
- The offline autologon trick above is temporary by design — verify
  `AutoAdminLogon` is back to `0` and `DefaultPassword` is gone afterwards
  (`reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon`
  via `virsh qemu-agent-command`, once the agent is up).
