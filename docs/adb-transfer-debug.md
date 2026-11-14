# TAB-A05-BA1 recovery: debugging adb push/pull EOF

Compared against the stock `recovery` ramdisk, which uses `musb-hdrc`
for the USB device controller. The TWRP ramdisk previously used
`11201000.usb`, had overlapping gadget-binding property triggers,
and restarted adbd during boot.

Changes:
- Match the stock `musb-hdrc` UDC and controller mode
- Set `sys.usb.ffs.aio_compat=1` **before** starting Android 9 adbd to
  select synchronous FunctionFS read/write
- Remove overlapping generic USB gadget binding and legacy adbd restart
- Unbind before rebuilding the ADB and MTP USB function links

These are candidate fixes; EOF has **not yet been tested on hardware**.
Keep a copy of the stock recovery for rollback.

## Reproduction / verification

```sh
adb shell 'ls /sys/class/udc; getprop sys.usb.controller; getprop sys.usb.ffs.aio_compat; getprop sys.usb.config; getprop init.svc.adbd'
adb shell 'dd if=/dev/zero of=/tmp/adb-test.bin bs=1048576 count=32'
adb pull /tmp/adb-test.bin
adb push /tmp/adb-test.bin /tmp/adb-test-pushed.bin
adb shell 'sha256sum /tmp/adb-test.bin /tmp/adb-test-pushed.bin'
adb shell 'dmesg | grep -Ei "usb|musb|ffs|gadget|disconnect|reset|aio|error"'
```

Repeat multiple times, with MTP enabled and disabled, and also try
`/data` as the target to separate storage problems from USB problems.
If the new image does not enumerate, inspect logs and revert the UDC
change first. The synchronous-I/O option is experimental on this device.
