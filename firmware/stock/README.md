# Stock firmware images

Unmodified stock firmware as served by Xiaomi's own update endpoint, kept here so
a device can be put back the way it was found.

These are Xiaomi's binaries, not this project's. They are redistributed here for
recovery only; all rights remain with the vendor.

## Provenance

Each file came from `/home/latest_version` for its model, fetched with
`tools/cloud_fw_info.py`, and its md5 was verified against the value the endpoint
reported before the file was written. `docs/data/stock-firmware.json` carries the
version, size, md5 and publication date for every one, so any file here can be
re-verified:

```sh
md5sum firmware/stock/yeelink.light.ceiling10-stock-2.0.6_0049.bin
# compare against .["yeelink.light.ceiling10"].md5 in docs/data/stock-firmware.json
```

## What these files are

A complete ESP-IDF application image plus the 4-byte CRC trailer Xiaomi's updater
expects - byte for byte what the vendor serves, trailer included. That is also
what `tools/append_crc.py` reproduces for an ESPHome build.

50 are ESP32 images and 9 are ESP8266; `chip` in the catalogue says which, read
from the image header rather than inferred from the product.

## Only the current version exists

Xiaomi serves exactly one firmware per model. Older versions appear in the
version history but cannot be downloaded, so these are the current builds at the
time of collection and restoring one means restoring to current, not to whatever
a given device happened to be running.

## Why this is a separate branch

These images are not part of what was proposed upstream. `esphome-yeelight-ceiling-light`
exists to hold ESPHome configurations, and 73 MB of vendor binaries changes what
the repository is - a reasonable thing for its maintainer to decline, and
declined. The branch offered upstream carries the tools, the catalogue and the
checksums, none of which depend on these files being present.

They are kept here so the archive survives: Xiaomi serves exactly one build per
model, so every earlier version is already unobtainable, and a model that is
withdrawn takes its firmware with it.
