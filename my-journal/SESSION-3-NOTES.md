# Session 3 — 2026-07-15

## The gist

Big one today — flashed real firmware onto the actual ESP32-S3 board for the first time (not just fake/simulated data anymore) and watched it boot up successfully. It's alive! It just doesn't know our WiFi password yet, so that's the very next thing to do.

I also got a little paranoid partway through about handing my home WiFi password to a random dev board and having a sensing device on my network in general, so we paused to sanity-check that before going further. Notes on that below.

## The technical play-by-play

1. **Found the right firmware files.**
   Checked `firmware/esp32-csi-node/release_bins/` for a build matching the board's chip — an **N16R8** (16MB flash, 8MB PSRAM). Turns out the repo doesn't ship a firmware build specifically configured for 16MB flash; the closest match is the standard build (configured for 8MB). That's fine — flashing an 8MB-sized firmware onto a 16MB chip is safe, it just leaves roughly half the flash chip unused for now. Used `--flash_size detect` (instead of hardcoding a size) so `esptool` reports the true physical 16MB size accurately rather than pretending it's 8MB.

2. **Flashed the board.**
   Board connected at `/dev/tty.usbmodem5C390217221`. Ran:
   ```
   python3 -m esptool --chip esp32s3 --port /dev/tty.usbmodem5C390217221 --baud 460800 \
     write_flash --flash_mode dio --flash_size detect \
     0x0     firmware/esp32-csi-node/release_bins/bootloader.bin \
     0x8000  firmware/esp32-csi-node/release_bins/partition-table.bin \
     0xf000  firmware/esp32-csi-node/release_bins/ota_data_initial.bin \
     0x20000 firmware/esp32-csi-node/release_bins/esp32-csi-node.bin
   ```
   `esptool` correctly identified the chip (ESP32-S3, 8MB PSRAM), auto-detected the real 16MB flash, wrote all four firmware pieces, and verified every write against a hash — no corruption. No BOOT button press was needed; the board reset into the new firmware automatically.

3. **Watched it boot over the serial connection.**
   A true "live, interactive" serial monitor needs a real keyboard/terminal, which isn't something this AI tool session can give directly — so instead we reset the board and captured ~10 seconds of its startup log as a snapshot. That confirmed:
   - The new firmware is running: **`esp32-csi-node` v0.7.0**.
   - The chip and flash info line up with what `esptool` reported (same 16MB-vs-8MB note appeared here too — harmless, just the firmware confirming it only uses the 8MB it was built for).
   - It's trying to connect to WiFi using a placeholder network name (`wifi-densepose`) that's baked into the firmware by default, and failing to find it — completely expected, since the board hasn't been told our actual home WiFi name/password yet.

## Network privacy check-in

Before going further and actually giving this thing our WiFi credentials next session, I wanted to pause and think through what I was actually nervous about:

- **Handing over the WiFi password.** Pretty standard IoT-device concern — any device that joins your home network needs the password. Mitigating factor: it's just joining our own local network like a phone or laptop would, not sending the password anywhere external.
- **The device "phoning home."** Worth knowing: the *sensing server software* (the part that runs on the Mac, not the ESP32 itself) does have one built-in external call by default — it checks an online "edge module registry" URL (`storage.googleapis.com/cognitum-apps/app-registry.json`) for optional add-on modules. That can be turned off entirely with a `--no-edge-registry` flag if we want the whole setup fully offline. As for the ESP32 firmware itself — from what we've looked at so far, it just streams sensing data to the Mac over the local network (UDP), nothing pointing externally that we've seen, though we haven't done a full line-by-line audit of the firmware source.
- **The dashboard being exposed to other devices on the network.** Checked this — by default the sensing server only listens on `127.0.0.1` (loopback), meaning only this Mac can reach it. It would need to be explicitly reconfigured (`--bind-addr 0.0.0.0`) to be reachable from other devices, which we have not done.

Net takeaway: comfortable moving forward with provisioning next session, with the external registry endpoint disabled if we want a fully local setup.

## Result

✅ Board flashed successfully and boots cleanly. It's alive, running the right firmware version, and just waiting to be told which WiFi network to join.

## Next session
- Provision the board with our real home WiFi credentials using `firmware/esp32-csi-node/provision.py`.
- Once it's on WiFi, point the sensing server at the real board (instead of `--source simulate`) and see genuine, live CSI sensing data in the dashboard for the first time.
