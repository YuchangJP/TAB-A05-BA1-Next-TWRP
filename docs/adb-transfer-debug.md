# TAB-A05-BA1 TWRP: adb transfer EOF

## Experimental baseline (2026-11-15)

A previous USB initialization change (commit `0879fe9`) prevented
USB enumeration on a real device despite a successful Actions build.
That approach simultaneously changed the UDC name, removed init/adbd
triggers, and modified gadget bind/unbind sequencing. It has been
**reverted**, including changes based on comparisons with stock recovery.

We now keep the previously enumerating TWRP USB gadget logic, including
`11201000.usb` and the original adbd lifecycle, and test only
`sys.usb.ffs.aio_compat=1` in the `on fs` action. Android 9 adbd
uses this property to select synchronous FunctionFS reads/writes.
This may improve adb push/pull EOF, but is **not hardware-verified**.

## Verification (test before flashing permanently)

- USB enumeration on the host: `lsusb` / Windows Device Manager
- `adb devices`, `adb shell getprop sys.usb.ffs.aio_compat`
- Run repeated `adb pull` / `adb push` with MTP on and off

```sh
adb shell 'dd if=/dev/zero of=/tmp/adb-test.bin bs=1048576 count=32'
adb pull /tmp/adb-test.bin
adb push /tmp/adb-test.bin /tmp/adb-test-pushed.bin
adb shell 'sha256sum /tmp/adb-test.bin /tmp/adb-test-pushed.bin'
```

If USB is still completely unrecognized, revert the single
`aio_compat` line and confirm the original TWRP image enumerates.
Before further changes collect the host-side USB descriptors
(`lsusb` / Device Manager) and device console/dmesg if available.

Stock recovery's configfs UDC string `musb-hdrc` does not establish
that it is valid in this separately built TWRP environment; do not
change it again without checking `/sys/class/udc` in the running kernel.
