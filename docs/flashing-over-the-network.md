# Flashing over the network, without opening the device

Every documented conversion in this project flashes over UART, which means taking
the device apart. On ESP32 Yeelights running stock firmware there is another
route: the device's own `miIO.ota` update mechanism can be pointed at a file on
your LAN.

Confirmed here on two units of different firmware generations:
`yeelink.light.lamp9` on `2.1.7_0031` (`miio_ver 0.0.9`) and
`yeelink.light.ceiling10` on `2.0.6_0049` (`miio_ver 0.0.6`). The requirements
differ between them - see the table further down. Third parties report the same
mechanism working on `yeelink.light.ceiling22`; that was not verified here.

> **Read the limits first.** This writes only the application partition. The
> stock bootloader and partition table remain, so it cannot recover a device that
> will not boot - that still needs UART.
>
> It is worth knowing what recovery you do have. The vendor's current stock image
> for a given model can be fetched from Xiaomi's cloud - see
> `docs/xiaomi-cloud-firmware.md` - so a way back exists without a UART backup,
> though only to the *current* build, since older versions are not downloadable.
> Separately, the updater writes to the inactive OTA slot, so the stock app
> normally survives a conversion in the other one.

---

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

## What is safe and what is not

Confirmed non-destructive on the reference unit:

- An OTA pointed at a URL returning **404** - the device fetches, fails, returns
  to `idle`, unharmed. A useful way to prove the path end to end without
  installing anything.
- Aborted transfers, including the HTTP/1.0 resets described in requirement 2.
- Unknown or malformed miIO methods, which are ignored without a reboot.

Not verified, and where the real risk lies:

- The **install** stage. Once a valid image downloads it is written and booted.
  A third-party report describes an official Yeelight OTA bricking a ceiling
  light, so this firmware's install path can write something unbootable.
- The OTA call itself carries no checksum, so nothing at the protocol level
  distinguishes a correct image from one built for the wrong model. What the
  firmware checks beyond the CRC trailer, and how it behaves on a bad image, was
  not tested - deliberately.

## Five requirements that are easy to miss

Not all five apply to every device, and the differences track the firmware
generation. Two units were converted, and this is what each one actually showed:

| | `yeelink.light.lamp9`<br>`miio_ver 0.0.9`, stock 2.1.7_0031 | `yeelink.light.ceiling10`<br>`miio_ver 0.0.6`, stock 2.0.6_0049 |
| --- | --- | --- |
| 1. CRC trailer | appended; necessity not tested | appended; necessity not tested |
| 2. HTTP/1.1 | **observed necessary** | served over 1.1 throughout, so never retested |
| 3. no port in the URL | **not needed** - flashed successfully on port 8000 | **required** |
| 4. `PARTITION_TABLE_MD5: n` | **not needed** - booted fine without it | **required** |
| 5. `FREERTOS_UNICORE: y` | set; necessity not tested | **required**; die confirmed single-core |

So a newer unit may well flash without 3 or 4 at all. An older one needs both, and
each failure looks like something else entirely - which is why they are written up
in detail below rather than as a checklist.


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
against five unrelated Xiaomi images which all reproduce exactly: one MT7697, and
the stock ESP32 images for `yeelink.light.ceiling10`, `lamp9`, `ceilb` and one
further ESP32 product.

`tools/append_crc.py` implements it, and can verify itself against any genuine
Xiaomi image you have.

### 2. The HTTP server must speak HTTP/1.1

Observed on `lamp9` (`miio_ver 0.0.9`). The device's downloader identifies itself
as `User-Agent: MIoT`. Against an HTTP/1.0 server it connects, begins reading,
then resets the connection - three times in quick succession - and returns to
`idle` with nothing written:

```
"GET /fw_crc.bin HTTP/1.1" 200 -
ConnectionResetError: [Errno 104] Connection reset by peer   (x3)
```

Python's `http.server` answers HTTP/1.0 and ignores `Range`, so it fails here.
Serving byte-identical content over HTTP/1.1 with keep-alive and range support
works first time. `tools/ota_server.py` is a minimal server that does this and
logs what the device actually requests.

Every later transfer, `ceiling10` included, was served over HTTP/1.1 by that tool,
so the failure was never reproduced on the older generation - it simply never had
the chance to occur.

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

**This is generation-specific.** Observed on `ceiling10` (`miio_ver 0.0.6`). The
`lamp9` on `0.0.9` does **not** have the bug: it was flashed successfully with
`--url http://<ip>:8000/fw_crc.bin`, which is why the procedure below originally
specified port 8000.

Port 80 is nevertheless the right default, because it works on both and costs
only a `sudo`.

#### Corollary: what counts as success

`["ok"]` is the cloud acknowledging the relay, not the device agreeing to do
anything. On `ceiling10` (`0.0.6`), `miIO.ota` also reboots the device about eight
seconds after it is accepted whether or not a download follows, so a failed
attempt power-cycles the light. **Watch the HTTP server log for a GET from the
device**; that is the only evidence the transfer started, and
`miIO.get_ota_state` moving `idle -> downloading -> installed` confirms it.

`tools/cloud_ota.py --sweep` tries several documented payload shapes in one login
and stops as soon as the device leaves `idle`, which is useful when a firmware
generation wants a different payload - though note that no payload shape helps if
the URL carries a port.

### 4. The app must tolerate a partition table with no MD5

**Also generation-specific**, and observed on `ceiling10` (`miio_ver 0.0.6`); the
`lamp9` on `0.0.9` booted an image built without this option, so its stock table
does carry the MD5 record.

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

Setting the option is harmless on a device that does not need it, so it is worth
having on any config intended for this route.

### 5. Build for a single core if the die has one

The `ceiling10`'s module is marked `ESP32-WROOM-32D`, which is normally the
dual-core `D0WD`. The die in it is not:

```
Chip type: Unknown ESP32 (revision v1.0)
Features:  Wi-Fi, BT, Single Core + LP Core, 240MHz
```

An image built for two cores tries to bring up an APP CPU that is not there, and
faults during startup - before Wi-Fi, so it looks exactly like requirement 4.

```yaml
esp32:
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_FREERTOS_UNICORE: y
```

Every config in this repository already sets this, which makes it easy to overlook
when writing a new one from scratch. It was set for the `lamp9` too, so whether
that unit strictly needs it is unknown - its die was never read.

The lesson is narrower than "check the die": the module marking does not tell you,
so set the option rather than trusting the label.

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

## Procedure

1. **Recover the device token and `did`** from the Xiaomi cloud. Existing tools
   cover this; note the Yeelight app account and the Xiaomi account may be the
   same identity, in which case no re-pairing is needed.

2. **Build the ESPHome image** for your model. Include `ap:` and
   `captive_portal:` - see "Always include a fallback AP" above, which explains
   why this is not optional.

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

   Port **80**, not a high port. Older firmware cannot parse a URL with an
   explicit port (requirement 3), and port 80 works on every unit tested, so it is
   the safe default - at the cost of needing root to bind:

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

## What the flash looks like underneath

Worth knowing before starting, because it determines what recovery is available.
Read the table with `esptool read-flash 0x8000 0xc00 ptable.bin`. Layout below is
from a `ceiling10`; other models are likely similar but were not dumped.

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

## Appendix: why the obvious approach fails

Background, kept for the record. Nothing below is needed to follow
the procedure above.

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
