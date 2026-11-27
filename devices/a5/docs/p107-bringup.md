# iPad mini 1 Global (iPad2,7 / P107AP) RAM boot

The cellular iPad mini 1 runs the P105 port with only its boot chain and
firmware taken from its own IPSW. Hardware tested: iPad2,7, S5L8942, iOS 9.3.5
with the EverPwnage jailbreak, booted through `flash --kdfu`.

## Verified on hardware

- kDFU from iOS 9.3.5 → the P107 iBSS → the auto-go iBEC → staging bundle → Linux 6.12.
- A shell over USB (CDC ECM, 10.55.0.2) and ssh.
- The SGX543MP2 microkernel boots (`microkernel is up`), and Xorg runs on
  modesetting with glamor.
- The Z2 digitizer reaches `READY` with P107.mtprops, which is byte-for-byte the same as P105's.
- XFCE with touch, from the NFS root (`desktop`).

Not checked yet: Wi-Fi, Bluetooth, audio, battery, sensors. The baseband
(Mav5, on HSIC and uart4) has no driver.

## Why P105's kernel and DTS fit unchanged

- **iBoot.** P107's decrypted iBSS/iBEC (iBoot-2261.30.37) are the same size as
  P105's. They differ in about 80 bytes of board data and strings only, and
  none of those bytes are at the auto-go hook's addresses (`HOOK_OFF` 0x2384c,
  the trampoline at 0x43e0e, `CACHE_FLUSH`, `USB_CONTINUE`).
- **ADT.** P107's DeviceTree adds the `baseband`, `uart4` and
  `usb-ehci/hsic-baseband` nodes and a few `product` flags. Every other
  difference is a phandle that those nodes shifted, or a `function-*` property
  that refers to such a phandle. Register addresses, GPIO pins, the Wi-Fi
  module (Borg) and the touch, backlight and button wiring are the same.

## Building

```sh
# P105 firmware first: the GPU files come from it (see below)
./cascadia a5 firmware                       # iPad2,5_8.4.1_12H321_Restore.ipsw
cp -a devices/a5/build/firmware devices/a5/build/firmware-p105
# then P107's boot chain, touch, Wi-Fi and Bluetooth
BOARD=p107 IPSW=$PWD/iPad2,7_8.4.1_12H321_Restore.ipsw ./cascadia a5 firmware
./cascadia a5 build
```

With `BOARD=p107`, `build-firmware.sh` uses the Donner 12H321 (iPad2,7) keys
from Legacy-iOS-Kit-Keys; that database's iPad2,5 entry matches the keys
already in this tree. The script also differs from the P105 run in three ways:

- It skips the PC-less (NAND) iBEC patch. That patcher knows only P105's iBEC,
  and a RAM boot does not need it.
- It copies `sgx543.fw`, `gl-event.pds` and the kernelcache from `GPU_FW_DIR`
  (default `build/firmware-p105`). The SGX543MP2 is the same part, but the
  driver uses offsets into P105's exact kext bytes, and P107's kernelcache
  does not match them.
- It has no reference hashes for P107's iBSS and iBEC.

`ios_check_model` accepts both iPad2,5 and iPad2,7.

## Fedora host notes

- **OpenSSH on iOS 9 signs with SHA-1 `ssh-rsa`.** Fedora's crypto policy
  rejects that (`error in libcrypto: invalid digest`). Run `flash --kdfu` with
  `OPENSSL_ENABLE_SHA1_SIGNATURES=1` set for that one command; the
  system-wide policy can stay as it is.
- **Rootless podman instead of docker.** A `docker` shim on `PATH` works if it
  maps `run --user U:G` to `--userns=keep-id` and adds
  `--security-opt label=disable` for SELinux.
- **firewalld rejects NFSv3 on the link.** Put the link in the trusted zone:
  `nmcli connection modify cascadia-usb connection.zone trusted`.
- **rpc.mountd can die on the first `nfs on`** with
  `couldn't open /var/lib/nfs/etab`. Run `systemctl restart nfs-mountd`
  afterwards, then check `ss -ltn | grep 20048`.
