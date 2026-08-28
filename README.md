# ESP32-S3 Tailscale Gateway

This repository is a modified fork of the original [`Csontikka/esphome-tailscale`](https://github.com/Csontikka/esphome-tailscale) project.

The upstream project turns an ESPHome device into a native Tailscale node. This fork extends that functionality so an ESP32-S3 can also operate as a lightweight **Tailscale subnet gateway for an existing LAN**.

The main goal is remote access to low-bandwidth LAN and IoT devices through Tailscale without requiring a Raspberry Pi, PC, VPN server, port forwarding, or static routes on the local router.

## Architecture

```text
Remote phone / laptop
        │
        │ Tailscale
        ▼
ESP32-S3 Tailscale Gateway
        │
        │ SNAT / NAPT
        ▼
Existing LAN
192.168.0.0/24
        │
        ├── ESPHome devices
        ├── Home Assistant devices
        ├── ESPEMS
        └── other LAN services
```

The ESP32-S3 remains connected to the existing Wi-Fi network as a normal station. It does not need to become the main router for the LAN.

## Changes from upstream

### Tailscale subnet routing

New component options:

```yaml
tailscale:
  advertise_routes: "192.168.0.0/24"
  gateway_snat: true
```

- `advertise_routes` announces the selected LAN subnet to the Tailscale control plane.
- `gateway_snat` enables Source NAT / NAPT between the Tailscale WireGuard interface and the local network.

This allows remote Tailscale peers to reach devices on the LAN without adding static routes to the local router.

### ESP-IDF IP forwarding and NAPT

The component enables the lwIP functionality required for subnet routing:

```text
CONFIG_LWIP_IP_FORWARD
CONFIG_LWIP_IPV4_NAPT
```

A MicroLink helper was added to enable and disable NAPT directly on the WireGuard network interface.

NAPT follows the VPN lifecycle:

```text
Tailscale CONNECTED
        ↓
Gateway NAPT enabled
        ↓
Subnet Router active

Tailscale disconnected / stopped
        ↓
Gateway NAPT disabled
```

NAPT is also explicitly disabled before the MicroLink / WireGuard interface is destroyed.

### Subnet Router status entity

A new ESPHome binary sensor reports whether subnet routing is operational:

```yaml
binary_sensor:
  - platform: tailscale
    subnet_router:
      name: "Subnet Router"
```

It becomes active only when Tailscale is connected, subnet routing is configured, gateway SNAT is enabled, and NAPT has been activated successfully.

### Optional YAML authentication key

In upstream, the Tailscale auth key is mandatory in YAML:

```yaml
tailscale:
  auth_key: !secret tailscale_auth_key
```

In this fork the YAML auth key is optional.

The device can therefore boot with no Tailscale credentials compiled into the firmware:

```yaml
tailscale:
  hostname: "esp32-tailscale-gateway"
  advertise_routes: "192.168.0.0/24"
  gateway_snat: true
```

If no authentication key is available, Wi-Fi, ESPHome API and the web server continue to operate normally. MicroLink is not started with an empty or dummy key.

### Runtime Tailscale Auth Key configuration

The auth key can be entered through an ESPHome `text` entity:

```yaml
text:
  - platform: tailscale
    auth_key:
      name: "Tailscale Auth Key"
      mode: password
```

The key is stored persistently in ESP32 NVS, used as a runtime override, retained across reboots, and masked in the entity state.

This makes it possible to build a generic firmware image without embedding a user-specific Tailscale key.

## First-run workflow

A gateway can be configured without storing Wi-Fi credentials or a Tailscale auth key in the firmware.

```text
ESP32-S3 boots
        ↓
ESPHome fallback AP
        ↓
Wi-Fi configured through Captive Portal
        ↓
ESP32 joins the LAN
        ↓
Open ESPHome Web Server
        ↓
Enter Tailscale Auth Key
        ↓
Key stored in NVS
        ↓
Tailscale connects
        ↓
NAPT enabled
        ↓
Subnet Router active
```

This allows the same firmware image to be deployed on multiple ESP32-S3 devices and configured after flashing.

## Minimal gateway configuration

```yaml
esphome:
  name: esp32-tailscale-gateway
  friendly_name: ESP32 Tailscale Gateway

esp32:
  board: esp32-s3-devkitc-1
  framework:
    type: esp-idf
    sdkconfig_options:
      CONFIG_LWIP_TCPIP_TASK_STACK_SIZE: "6144"
      CONFIG_LWIP_TCPIP_CORE_LOCKING: n

external_components:
  - source:
      type: git
      url: https://github.com/KamilDz1990/esphome-tailscale.git
      ref: main
    components:
      - tailscale
    refresh: 0s

psram:
  mode: octal
  speed: 80MHz

wifi:
  power_save_mode: none
  ap:
    ssid: "esp32-tailscale-gateway"
    password: "change-me-123"

captive_portal:

logger:
  level: INFO

api:
  reboot_timeout: 0s

ota:
  - platform: esphome

web_server:
  port: 80

tailscale:
  hostname: "esp32-tailscale-gateway"
  max_peers: 16
  advertise_routes: "192.168.0.0/24"
  gateway_snat: true
  disable_telemetry: true

text:
  - platform: tailscale
    auth_key:
      name: "Tailscale Auth Key"
      mode: password

binary_sensor:
  - platform: tailscale
    connected:
      name: "VPN Connected"
    subnet_router:
      name: "Subnet Router"

switch:
  - platform: tailscale
    vpn_enabled:
      name: "VPN Enabled"
```

See [`esp32-s3-tailscale-gateway.yaml`](esp32-s3-tailscale-gateway.yaml) for a ready-to-use example.

## Tailscale setup

After the gateway joins the tailnet:

1. Open the Tailscale Admin Console.
2. Approve the advertised subnet route, for example `192.168.0.0/24`.
3. Disable node key expiry for an unattended gateway if appropriate for your setup.
4. Configure Tailnet Grants / ACLs so only intended users and devices can access the advertised LAN.

This implementation is a **subnet router**, not an exit node. It routes only the advertised LAN subnet unless the implementation is intentionally extended.

## Current test status

Reference setup:

- ESP32-S3-DevKitC-1
- ESP-IDF
- PSRAM enabled
- ESPHome 2026.7.x
- Wi-Fi uplink
- advertised subnet: `192.168.0.0/24`

The gateway has been tested continuously for more than **2 days and 17 hours** without observed connectivity or stability problems.

Remote LAN access has been successfully verified from:

- LTE / mobile data,
- external Wi-Fi networks,
- locations outside the local network.

The intended workload is low-bandwidth remote access to ESPHome, Home Assistant, ESPEMS and other IoT / LAN devices.

## Security notes

- Do not expose the ESPHome web server directly to the public Internet.
- Consider web server authentication on networks that are not fully trusted.
- Treat the Tailscale auth key as a secret.
- Use Tailnet Grants / ACLs to restrict access to the advertised subnet.
- This is a hobby / experimental embedded networking project, not a hardened enterprise router.

## Fork change log

A focused summary of changes introduced by this fork is available in [`FORK_CHANGES.md`](FORK_CHANGES.md).

## Upstream project and credits

The original ESPHome Tailscale integration and most of the underlying component architecture come from:

**Csontikka / ESPHome Tailscale**  
https://github.com/Csontikka/esphome-tailscale

This fork builds on that work and focuses specifically on the ESP32-S3 subnet-gateway use case.

## License

This fork retains the upstream licensing terms. See [`LICENSE`](LICENSE) for details.
