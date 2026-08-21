# Flashing over the network, without opening the device

Every documented conversion in this project flashes over UART, which means taking
the device apart. On ESP32 Yeelights running stock firmware there is another
route: the device's own `miIO.ota` update mechanism can be pointed at a file on
your LAN.

Confirmed here on `yeelink.light.lamp9` firmware `2.1.7_0031`
(`miio_ver 0.0.9`). Third parties report the same mechanism working on
`yeelink.light.ceiling22`; that was not verified as part of this work.

> **Read the limits first.** This writes only the application partition. The
> stock bootloader and partition table remain, so it cannot recover a device that
> will not boot - that still needs UART. Take a flash backup over UART first if
> you want any way back, because no stock firmware image for these models is
> archived anywhere public.

---

## Why the obvious approach fails

Sending `miIO.ota` **locally** over UDP 54321 with a valid token is refused:

```
{"code": -30020, "message": "service not available."}
```

The response is identical with no parameters at all and with a full payload,
which places the refusal above the parameter layer: five payload shapes were
tried and none made any difference. Local OTA is disabled on this firmware, and
the update command has to arrive from the Xiaomi cloud instead.

Useful error signatures when probing:

| Response | Meaning |
| -------- | ------- |
| `-9999 user ack timeout` | method not implemented; firmware never acknowledges |
| `-32602 Invalid param.` | method exists, parameter validation rejected the input |
| `-30020 service not available.` | method exists, the service refuses before parsing |

## The route that works

Xiaomi's cloud exposes an RPC relay that forwards a raw miIO call to a device you
own. Because the command then originates from the server, the OTA service accepts
it - and `app_url` may point anywhere the device can reach, including a private
LAN address over plain HTTP.

```
POST https://<region>.api.io.mi.com/app/home/rpc/<did>
data = {"id":N,"method":"miIO.ota","params":{"app_url":"http://<your-lan-ip>/fw_crc.bin"}}
```

Two things about that payload matter:

- `app_url` alone is what worked here (`mcu_url` is the equivalent for the
  companion MCU). The larger payload shown in `python-miio` - `mode`, `install`,
  `file_md5`, `proc` - was not tested over the cloud channel, so this is what is
  known to work rather than the only form that does.
- No checksum is passed in this form, so nothing in the protocol protects you
  from shipping a bad image.

The cloud only relays the instruction. The firmware file never leaves your LAN.

## Four requirements that are easy to miss

### 1. The image carries a 4-byte CRC trailer

Xiaomi's own ESP32 update images are a normal ESP-IDF application image - whose
internal SHA-256 verifies - followed by four extra bytes. A plain ESPHome
`firmware.bin` has no trailer, so one was appended here to match the vendor
format. Whether the device rejects an image without it was not tested.

The algorithm is **not** a standard CRC-32; none of the catalogued variants
reproduce it:

```
CRC-32, polynomial 0x04C11DB7 (reflected 0xEDB88320)
        init 0x00000000, refin/refout = true, xorout 0x00000000
        stored little-endian
```

That is standard CRC-32 without the customary pre- and post-inversion. Verified
against two unrelated Xiaomi images - one ESP32, one MT7697 - which both
reproduce exactly.

`tools/append_crc.py` implements it, and can verify itself against any genuine
Xiaomi image you have.

### 2. The HTTP server must speak HTTP/1.1

The device's downloader identifies itself as `User-Agent: MIoT`. Against an
HTTP/1.0 server it connects, begins reading, then resets the connection - three
times in quick succession - and returns to `idle` with nothing written:

```
"GET /fw_crc.bin HTTP/1.1" 200 -
ConnectionResetError: [Errno 104] Connection reset by peer   (x3)
```

Python's `http.server` answers HTTP/1.0 and ignores `Range`, so it fails here.
Serving byte-identical content over HTTP/1.1 with keep-alive and range support
works first time. `tools/ota_server.py` is a minimal server that does this and
logs what the device actually requests.

### 3. The URL must not contain a port

The updater does not strip the port from the URL authority. Given
`http://192.0.2.2:8000/fw.bin` it issues a DNS lookup for the literal string
`192.0.2.2:8000`:

```
192.0.2.10.24408 > 192.0.2.1.53: A? 192.0.2.2:8000.
192.0.2.1.53 > 192.0.2.10.24408: NXDomain
```

Three attempts, three NXDOMAINs, then it gives up. `miIO.ota` still answers
`["ok"]`, `miIO.get_ota_state` still reads `idle`, and nothing arrives at the HTTP
server - so the symptom is indistinguishable from the command being ignored.

**Serve on port 80** so the authority is a bare address. It is then recognised as
an IP literal, never goes to DNS, and the transfer starts immediately.

This cost a long detour to find, because it is invisible from the machine running
the HTTP server: a switch does not forward the device's DNS queries to another
port, so a capture there shows nothing either way. It was only visible from the
router.

Observed on `yeelink.light.ceiling10` (`miio_ver 0.0.6`). Whether the newer
`0.0.9` firmware parses a port correctly was not tested - the working `lamp9`
flash happened to use port 80.

#### Corollary: what counts as success

`["ok"]` is the cloud acknowledging the relay, not the device agreeing to do
anything, and on some firmware `miIO.ota` reboots the device about eight seconds
later whether or not it downloads. **Watch the HTTP server log for a GET from the
device**; that is the only evidence the transfer started, and
`miIO.get_ota_state` moving `idle -> downloading -> installed` confirms it.

`tools/cloud_ota.py --sweep` tries several documented payload shapes in one login
and stops as soon as the device leaves `idle`, which is useful when a firmware
generation wants a different payload - though note that no payload shape helps if
the URL carries a port.

### 4. The app must tolerate a partition table with no MD5

This one does not stop the transfer. The image downloads, the updater reports
`installed`, and then the device is bricked in a reboot loop - **silent at every
layer**. No Wi-Fi, no fallback AP, and power-cycling cannot reach ESPHome's safe
mode either. Only UART reveals why:

```
E (344) partition: No MD5 found in partition table
E (345) partition: load_partitions returned 0x105
assert failed: esp_ota_get_running_partition esp_ota_ops.c:721 (it != NULL)
```

ESP-IDF's `CONFIG_PARTITION_TABLE_MD5` defaults to `y`, and
`components/esp_partition/partition.c` refuses a table with no MD5 record when it
is set. Its own Kconfig help says the generation "should be turned off for legacy
bootloaders which cannot recognize the MD5 checksum in the partition table" -
which is exactly the situation here, because the stock partition table on the
older firmware generation carries no such record.

`esp_ota_get_running_partition()` then finds nothing and the app asserts **before
Wi-Fi and before `safe_mode` set up**, which is why none of the usual recovery
paths exist. Recovery is UART.

The fix is one option:

```yaml
esp32:
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_PARTITION_TABLE_MD5: n
```

Nothing on flash changes - the stock bootloader, partition table and layout are
all left alone; the app simply stops requiring a record that was never there.

Confirm it applied before flashing, because a disabled bool is written as a
comment rather than `=n`:

```
grep PARTITION_TABLE_MD5 .esphome/build/<name>/sdkconfig.<name>
# CONFIG_PARTITION_TABLE_MD5 is not set

strings .pioenvs/<name>/firmware.bin | grep -c "No MD5 found in partition table"
0
```

Observed on `yeelink.light.ceiling10` (`miio_ver 0.0.6`); a `lamp9` on `0.0.9`
does not need it, so its stock table does carry the MD5. Setting the option is
harmless either way, so it is worth having on any config intended for this route.

## What the flash looks like underneath

Worth knowing before starting, because it determines what recovery is available.
Read the table with `esptool read-flash 0x8000 0xc00 ptable.bin`. On a
`ceiling10`:

| label | type | subtype | offset | size |
| ----- | ---- | ------- | ------ | ---- |
| nvs | data | nvs | 0x9000 | 16K |
| otadata | data | otadata | 0xD000 | 8K |
| phy_init | data | phy | 0xF000 | 4K |
| miio_fw1 | app | ota_0 | 0x10000 | 1920K |
| miio_fw2 | app | ota_1 | 0x1F0000 | 1920K |
| test | app | test | 0x3D0000 | 76K |
| mfi_p | data | spiffs | 0x3E3000 | 4K |
| factory_nvs | data | nvs | 0x3E4000 | 16K |
| coredump | data | coredump | 0x3E8000 | 64K |
| minvs | data | 0xfe | 0x3F8000 | 16K |

Three things follow from it.

**The app slots are 1920 KB**, so image size is a non-issue: stock itself is about
1.3 MB and a typical ESPHome build for one of these is well under a megabyte.

**There is no `factory` partition**, so `otadata` alone decides what boots.

**Stock survives the conversion.** The updater writes to the *inactive* slot, so
after flashing, the original vendor app is still sitting in the other one.

### Getting stock back without reflashing it

`otadata` holds one 32-byte record per 4K sector, at `0xD000` and `0xE000`. Each
carries a sequence number; the highest valid one wins and selects
`ota_[(seq-1) % 2]`. After a conversion you will typically find the older record
still pointing at the slot stock lives in.

So erase the newer record and let the older one win:

```
esptool erase-region 0xE000 0x1000
```

NVS survives, so Wi-Fi credentials and the vendor pairing stay intact and the
device rejoins its account by itself.

Do **not** erase the whole of `otadata`: with no valid record the bootloader falls
back to the *first* app partition, which is `ota_0` - not necessarily stock.

## Procedure

1. **Recover the device token and `did`** from the Xiaomi cloud. Existing tools
   cover this; note the Yeelight app account and the Xiaomi account may be the
   same identity, in which case no re-pairing is needed.

2. **Build the ESPHome image** for your model. Include `ap:` and
   `captive_portal:` - see the warning below.

3. **Check it fits.** Only the application partition is written and the stock
   partition table stays, so the image must fit the slot the stock firmware uses.
   The stock partition table has not been dumped, so the exact slot size is
   unknown. The `lamp9` ESPHome build used here was 810 KB and installed without
   trouble.

4. **Append the CRC trailer:**

   ```
   python3 tools/append_crc.py firmware.bin fw_crc.bin
   ```

5. **Serve it over HTTP/1.1** and confirm another host on the LAN can fetch the
   whole file before going further:

   Port **80**, not a high port - the updater cannot parse a URL with an
   explicit port (see requirement 3), so binding 80 needs root:

   ```
   sudo python3 tools/ota_server.py fw_crc.bin 80
   ```

6. **Relay the OTA command through the cloud**, with no port in the URL:

   ```
   python3 tools/cloud_ota.py --server de --ip <device-ip> \
       --url http://<your-lan-ip>/fw_crc.bin
   ```

7. **Watch it land.** `miIO.get_ota_state` walks `idle -> downloading ->
   installed`, the server logs one full-size `GET`, and the device reboots into
   ESPHome. The stock protocols (TCP 55443, UDP 54321) go silent.

## Always include a fallback AP

```yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "fallback-ap"

captive_portal:
```

A wrong credential is otherwise unrecoverable without UART - which defeats the
entire point of flashing over the network. This is not hypothetical: during this
work an SSID picked up a trailing `\r` from a CRLF file, the lamp came up on
ESPHome unable to join, and the fallback AP turned a teardown into a two-minute
captive-portal fix.

Worth knowing how portal-saved credentials behave, because it is easy to get
wrong. They are stored in NVS and **replace** the compiled-in ones (`set_sta`, not
`add_sta`) - so a device can keep running on them while the firmware carries a
wrong SSID. But the preference is keyed on `App.get_config_version_hash()`, so
**any configuration change orphans them** and the device falls back to whatever is
compiled in. They persist across a plain re-flash of the same config, not across
an edited one.

## What is safe and what is not

Confirmed non-destructive on the reference unit:

- An OTA pointed at a URL returning **404** - the device fetches, fails, returns
  to `idle`, unharmed. A useful way to prove the path end to end without
  installing anything.
- Aborted transfers, including the HTTP/1.0 resets above.
- Unknown or malformed miIO methods, which are ignored without a reboot.

Not verified, and where the real risk lies:

- The **install** stage. Once a valid image downloads it is written and booted.
  A third-party report describes an official Yeelight OTA bricking a ceiling
  light, so this firmware's install path can write something unbootable.
- The OTA call itself carries no checksum, so nothing at the protocol level
  distinguishes a correct image from one built for the wrong model. What the
  firmware checks beyond the CRC trailer, and how it behaves on a bad image, was
  not tested - deliberately.
