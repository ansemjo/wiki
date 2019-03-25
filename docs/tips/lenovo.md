# Lenovo

## Updating BIOS on a Lenovo from Linux

Some modern Lenovo machines do not have an optical disc drive. The only option
for machines without Windows is a bootable .iso image though. What now?

Turns out inside that image there is another bootable format: an El Torito
image. You can extract that with a script called
[`geteltorito.pl`](/assets/geteltorito.pl) and flash it to a USB stick.

```
# ./geteltorito.pl -o n10ur17w-usb.img n10ur17w.iso
Booting catalog starts at sector: 20
Manufacturer of CD: NERO BURNING ROM
Image architecture: x86
Boot media type is: harddisk
El Torito image starts at sector 27 and has 47104 sector(s) of 512 Bytes

Image has been written to file "n10ur17w-usb.img".
# dd if=n10ur17w-usb.img of=/dev/sdb bs=1M
23+0 records in
23+0 records out
24117248 bytes (24 MB, 23 MiB) copied, 0.354471 s, 68.0 MB/s
```

!!!note
    This information and script is taken from
    [thinkwiki.de](http://thinkwiki.de/BIOS-Update_ohne_optisches_Laufwerk_unter_Linux)
