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

The XML ships with neutral defaults you'll likely want to change too — both
are called out with a comment at the relevant line:

- **Keyboard layout** (`InputLocale`, two places): `en-US` by default. Set to
  e.g. `da-DK` for Danish, `de-DE` for German, matching your physical
  keyboard.
- **Timezone** (`TimeZone`): `UTC` by default, which always works. Set to a
  real Windows timezone ID (e.g. `Romance Standard Time` for
  Copenhagen/Paris/Brussels, `Pacific Standard Time` for US West Coast) if
  you want the guest clock to show local time.

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

## 4. Start the VM and connect to it

`virt-install ... --noautoconsole` above only *creates and starts* the VM
without opening a display — you still need to connect separately to see
anything. `--graphics spice --video qxl` means any SPICE-capable viewer
works.

Start (or restart) it:

```sh
virsh --connect qemu:///system start win11-iot-ltsc
```

Check its state at any time:

```sh
virsh --connect qemu:///system list --all
```

Get a GUI, either way:

- **`virt-manager`** — the VM shows up under the `QEMU/KVM` connection
  (`qemu:///system`); double-click it to open the display.
- **`virt-viewer`** from a terminal:
  ```sh
  virt-viewer --connect qemu:///system win11-iot-ltsc
  ```

You'll land on the Windows login screen. Log in with the local administrator
account you set in `autounattend.xml` before building the ISO (username
`admin` by default — see the `UserAccounts` section of the XML for the exact
name, and whatever password you replaced the placeholder with).

**Always shut down with `virsh shutdown`, not `virsh destroy`.** `destroy` is
a hard power-off and can leave the NTFS filesystem "unclean," which blocks a
later offline `guestfish` edit (like the guest-agent fix below) until Windows
has booted once more and shut down cleanly.

```sh
virsh --connect qemu:///system shutdown win11-iot-ltsc
```

Once the guest agent (below) is working, a few more things become available
without needing the GUI at all:

```sh
# confirm the agent is alive
virsh --connect qemu:///system qemu-agent-command win11-iot-ltsc '{"execute":"guest-ping"}'

# get the guest's IP reliably (works even without a DHCP lease visible to the host)
virsh --connect qemu:///system domifaddr win11-iot-ltsc --source agent

# clean guest-initiated shutdown instead of an ACPI request
virsh --connect qemu:///system shutdown win11-iot-ltsc --mode agent
```

## 5. Guest agent doesn't come up — why, and the fix

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
layouts, and impossible to verify since the field is masked). The
[`scripts/`](scripts/) folder has the exact files this needs — each one has
usage notes in its header comment:

1. Extract `qemu-ga.exe` and its DLLs from `virtio-win.iso:\guest-agent\qemu-ga-x86_64.msi`
   with `msiextract`, copy them into the guest filesystem at
   `C:\Program Files\Qemu-ga\`, then register the service with
   [`scripts/qga-service.reg`](scripts/qga-service.reg) (`hivexregedit --merge`
   against the guest's `SYSTEM` hive).
2. Copy `vioserial\w11\amd64\vioser.{inf,sys,cat}` from `virtio-win.iso` into
   `C:\Windows\INF\` inside the guest filesystem (`fix-vioserial.cmd` below
   stages them from there into a proper source folder before installing —
   **not** directly into `C:\Windows\INF\`, which `pnputil /add-driver`
   refuses as a source location).
3. Delete the stale device enumeration key so Windows treats the
   virtio-serial controller as new hardware again, with
   [`scripts/delete-vioserial-enum.reg`](scripts/delete-vioserial-enum.reg)
   (confirm the exact device ID for your setup first — see the comment in
   that file; `1041`=virtio-net, `1042`=virtio-blk, `1043`=virtio-console/serial,
   `1045`=virtio-balloon).
4. Place [`scripts/fix-vioserial.cmd`](scripts/fix-vioserial.cmd) at
   `C:\Windows\Setup\fix-vioserial.cmd` inside the guest filesystem, then set
   a **temporary** autologon that triggers it once, with
   [`scripts/autologon-runonce.reg`](scripts/autologon-runonce.reg) (edit the
   password placeholder in that file to match your real one first — merge
   against the guest's `SOFTWARE` hive). The script installs the driver,
   starts the service, and deletes the autologon settings itself as its last
   step — no permanent passwordless login left behind.
5. Boot the VM once — it logs in automatically, the script fixes the driver
   and starts the service, then disables autologon again on its own. Verify
   with `virsh qemu-agent-command <vm> '{"execute":"guest-ping"}'`.

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
