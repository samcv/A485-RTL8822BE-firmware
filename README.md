# Bluetooth fix for Lenovo A485 (and possibly others with RTL8822BE wifi)

## Steps

1. Download `rtl8822befw.bin`
2. Backup old firmware ` sudo cp /lib64/firmware/rtlwifi/rtl8822befw.bin /lib64/firmware/rtlwifi/rtl8822befw.bin.bak`
3. Copy new firmware: `sudo cp rtl8822befw.bin /lib64/firmware/rtlwifi/rtl8822befw.bin`
4. Reboot (though putting it to sleep may work too).

## Caveats
For me the bluetooth will only work starting the first time you put the laptop to sleep.
I haven't tested yet but it may be possible to blacklist the wifi module, then load
it later on boot, but I haven't tested this yet.


## Where is this firmware from? How did you get it?

With a fair amount of effort I was able to extract the Lenovo provided firmware
for this wifi card.

How did I do this?
After much debugging trying to fix the bluetooth on Linux I decided to try another approach. I downloaded the Windows wifi driver from Lenovo
[`801373fb20311f5acc45476c7caeeb45b7fc39fb r0wrw05w.exe`](https://support.lenovo.com/us/en/downloads/ds504117)

After installing it you will find a binary blob located at `C:/DRIVERS/WLAN Driver/Source/rtwlane.sys`. The problem was to
find where in this large file the firmware was (where it started and where it ended).
I ended up spending a lot of time in a hex editor viewing this binary blob and the current
linux-firmware provided one. After identifying where I thought the start of the file was (at bytes `22 88 00`),
the next issue was where it ended. Not finding any real identifier at the end of the
linux-firmware version I looked to the Linux source code where the firmware is loaded.
You want `drivers/staging/rtlwifi/halmac/halmac_88xx/halmac_api_88xx.c` in function
halmac_download_firmware_88xx(). Here I was able to find the validation code that is used to validate
firmware prior to loading. Lucky for me there are 4 or so locations toward the start of the firmware
which after some operations are applied to it allow it to validate the firmware has the correct length.
By using the hex editor and finding these locations I was able to find the correct length of the firmware.

In case anybody doesn't trust the firmware I provide you can download Lenovo's firmware (check that the sha1sum is the
same) and do:

```
dd if=rtwlane.sys of=rtl8822befw.bin skip=4932176 bs=1 count=78320
```

Final result should have the following checksum:
`0017b27df0dcd9b944f40fce873487b80c3d9eb4  rtl8822befw.bin`

What | offset | width | Description|Hex Data|Value
-----|--------|-------|-------------|
HALMAC_FWHDR_OFFSET_VERSION_88XX | 4 | 2 bytes | Major firmware version number|14|20
HALMAC_FWHDR_OFFSET_SUBVERSION_88XX | 6 | 1 byte | Minor firmware version number|04|4
HALMAC_FWHDR_OFFSET_SUBINDEX_88XX | 7 | 1 byte | Patch firmware version number|00|0
HALMAC_FWHDR_OFFSET_MEM_USAGE_88XX | 24 | 1 byte || 08 | 8
HALMAC_FWHDR_OFFSET_DMEM_SIZE_88XX | 36 | 4 bytes ||D8 0B 00 |3032
HALMAC_FWHDR_OFFSET_IRAM_SIZE_88XX |48 | 4 bytes| |C8 25 01|75208
HALMAC_FWHDR_OFFSET_ERAM_SIZE_88XX| 52|4 bytes| |00 00 00 | 0

Static Values:
* HALMAC_FW_CHKSUM_DUMMY_SIZE_88XX = 8
* HALMAC_FWHDR_SIZE_88XX = 64

Simplified

```
/* Simplified algorithm based on code in halmac_api_88xx.c */
    dmem_pkt_size = DMEM_SIZE + HALMAC_FW_CHKSUM_DUMMY_SIZE_88XX;
	iram_pkt_size = IRAM_SIZE + HALMAC_FW_CHKSUM_DUMMY_SIZE_88XX;
	if ((MEM_USAGE & 4) != 0) {
		eram_pkt_size = ERAM_SIZE;
    }
	if (eram_pkt_size != 0)
		eram_pkt_size += HALMAC_FW_CHKSUM_DUMMY_SIZE_88XX;
halmac_fw_size = (HALMAC_FWHDR_SIZE_88XX + dmem_pkt_size + iram_pkt_size + eram_pkt_size);
/* Now solving for halmac_fw_size by using our calculated values: */
78320                       64           +   (3032+8)    + (75208+8)     + (0)

```

The Lenovo provided firmware shows this is version 20.4.0 based on the above values stored in those offsets. All values
are little endian by the way.

# Hardware info
USB shows up as:
`Bus 004 Device 008: ID 0bda:b023 Realtek Semiconductor Corp. `
Wifi shows up as:
`02:00.0 Network controller [0280]: Realtek Semiconductor Co., Ltd. RTL8822BE 802.11a/b/g/n/ac WiFi adapter [10ec:b822]`
