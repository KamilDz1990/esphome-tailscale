# Fork changes — KAM-TECH ESP32-S3 Tailscale Gateway

This document summarizes the functionality added in this fork compared with the upstream [`Csontikka/esphome-tailscale`](https://github.com/Csontikka/esphome-tailscale) project.

## 2026-08 — ESP32-S3 subnet gateway extension

### Added

- Tailscale subnet route advertisement through `advertise_routes`.
- Optional gateway SNAT / NAPT through `gateway_snat`.
- ESP-IDF lwIP forwarding support for routed LAN traffic.
- IPv4 NAPT support for traffic between the Tailscale WireGuard interface and the upstream LAN.
- MicroLink helper for enabling and disabling NAPT on the WireGuard netif.
- NAPT lifecycle management:
  - enable after a successful Tailscale connection,
  - disable when the VPN disconnects,
  - disable before MicroLink / WireGuard shutdown.
- `Subnet Router` binary sensor that reflects the real routing state instead of only Tailscale connection state.
- Optional YAML `auth_key` configuration.
- Safe startup without an auth key: MicroLink is not started until a usable key is available.
- Runtime Tailscale auth key entry through an ESPHome `text` entity.
- Persistent auth-key override stored in ESP32 NVS.
- Masked auth-key entity state after the runtime key has been loaded.
- Generic first-run deployment model using ESPHome fallback AP / Captive Portal for Wi-Fi setup and the ESPHome web server for Tailscale key setup.

### Changed

- The component can now be used as a lightweight subnet gateway for an existing LAN instead of only as a native Tailscale endpoint.
- The Tailscale auth key no longer has to be compiled into firmware.
- A single generic firmware image can be flashed to multiple ESP32-S3 devices and configured after first boot.

### Tested

Reference environment:

- ESP32-S3-DevKitC-1
- ESPHome 2026.7.x
- ESP-IDF
- PSRAM enabled
- Wi-Fi uplink
- advertised LAN `192.168.0.0/24`

Observed test result as of 2026-08-28:

- more than 2 days and 17 hours of continuous operation,
- no observed stability or connectivity problems,
- successful remote LAN access over LTE / mobile data,
- successful remote LAN access from external Wi-Fi networks.

## Credits

The original ESPHome Tailscale component, protocol integration and most of the project structure originate from Csontikka's upstream project.

This fork focuses specifically on adding the ESP32-S3 subnet-gateway use case while retaining ESPHome integration.
