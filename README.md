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

## Getting started with the current firmware

These steps apply to [`esp32-s3-tailscale-gateway_22102026.yaml`](esp32-s3-tailscale-gateway_22102026.yaml), revision **22102026**, on a device with this firmware already installed. A YAML file is a build configuration, not a flashable firmware image.

### Hardware and network

The current configuration targets an **ESP32-S3-DevKitC-1 with 16 MB flash and octal PSRAM at 80 MHz**, with a WS2812 RGB LED on **GPIO48**. Check that your board matches these settings before building and flashing.

Power the board through its USB power input using a suitable USB supply. This configuration uses **2.4 GHz Wi-Fi** for network access; no Ethernet or additional GPIO wiring is needed for normal operation.

The advertised LAN is **`192.168.0.0/24`** (mask `255.255.255.0`). If your LAN uses a different subnet, change `tailscale.advertise_routes` before building and flashing. The web interface does not expose a subnet configuration field in this YAML.

### Connect Wi-Fi

1. Power on the gateway and allow approximately one minute for the fallback access point to appear if Wi-Fi is not configured.
2. Connect your phone or laptop to **`esp32-s3-tailscale-gateway`**, password **`1234567890`**. Stay connected even if the device reports no Internet access.
3. Open the captive portal, or enter **http://192.168.4.1/** manually.
4. Select your home 2.4 GHz Wi-Fi network, enter its password, and save.
5. Reconnect your phone or laptop to your home network.
6. Find the gateway's assigned IP address in your router's DHCP/client list and open **`http://<gateway-LAN-IP>/`**. You can also try **http://esp32-s3-tailscale-gateway.local/** if local name resolution is supported.

The setup address `192.168.4.1` is separate from the address assigned by your home router. Consider reserving the gateway's LAN address in your router's DHCP settings.

### Register with Tailscale

1. Sign in to the [Tailscale admin console](https://login.tailscale.com/admin) and generate an **auth key** under Keys. Leave **Ephemeral** disabled for a persistent gateway. Keep the key private.
2. In the gateway's local web interface, paste the full key into **Tailscale Auth Key** and save the field.
3. Make sure **VPN Enabled** is on.
4. Wait for **VPN Connected** and **Subnet Router** to become active.
5. In the Tailscale admin console, locate **`esp32-tailscale-gateway`**. Approve the device if required, then approve its advertised route **`192.168.0.0/24`**.
6. Check that your tailnet access policy permits the intended users/devices to access the target LAN services.

**Subnet Router** reports the gateway's local routing state. It does not confirm that the route has been approved or that your access policy allows a remote client.

### Test remote access

1. Install Tailscale on your phone and sign in to the same tailnet.
2. Enable Tailscale, turn off the phone's Wi-Fi, and use mobile data.
3. Open a known LAN service using its existing IP address and port. For example, if Home Assistant is locally available at `http://192.168.0.50:8123/`, open that same address remotely.

Use the actual address of your service. Start with IP addresses; local discovery and local hostnames may not work across the subnet router. If another Wi-Fi network uses the same subnet as your home LAN, test over mobile data to help rule out overlapping routes.

### RGB LED status — revision 22102026

| Color | Meaning |
| --- | --- |
| Red | Wi-Fi is not connected to the router, including fallback AP setup without an uplink. |
| Blue | Wi-Fi is connected, but no ESPHome API client is connected. |
| Green | Wi-Fi and at least one ESPHome API client, such as Home Assistant, are connected. |

The LED is updated on startup and Wi-Fi/API connection events. Disconnect handlers wait 250 ms before updating it. There is no periodic five-second LED update, orange connection state, or IP-address-based AP color in this revision.

**The LED does not indicate Tailscale status.** Blue can be normal when running without an ESPHome API client. Opening the web interface does not establish an ESPHome API connection. Check **VPN Connected**, **Subnet Router**, and actual remote access to verify the VPN. The **Status: Normalny** text is set at startup and is not a VPN health check.

### Restart and factory reset

- For a normal restart, use **Restart** in the web interface or cycle power.
- To restore factory settings, let the firmware boot normally, then hold the button connected to **GPIO0** (usually **BOOT**) for at least **5 seconds**. The reset sequence starts with five on/off flashes over five seconds, followed by another five-second delay before factory reset and restart. You can release the button once the sequence begins.
- Wi-Fi/API connection changes can interrupt the visual flashing pattern.
- Do not hold BOOT while applying power for this procedure: on typical boards this enters firmware download mode. The RESET/EN button is different from GPIO0/BOOT.
- After factory reset, configure Wi-Fi and Tailscale again. Prepare a new auth key if the previous key was single-use or has expired. Remove any obsolete device entry separately in the Tailscale admin console.

### Troubleshooting

| Symptom | Check |
| --- | --- |
| Setup Wi-Fi is not visible | Wait approximately one minute and check power. If saved Wi-Fi works, the gateway may already be connected to your router. |
| Local web interface does not open | Check the current DHCP address, use HTTP, and make sure your client is not on an isolated guest network. |
| VPN Connected stays off | Check VPN Enabled, the auth key, Internet access, and any required device approval. |
| VPN connects but LAN services do not open | Check route approval, access policy, Subnet Router, the configured subnet, and whether the target service works locally. |
| LED stays blue | This indicates no ESPHome API client; check the VPN entities separately. |

For more detail, see [ESPHome Captive Portal](https://esphome.io/components/captive_portal/), [Tailscale auth keys](https://tailscale.com/docs/features/access-control/auth-keys), and [Tailscale subnet routers](https://tailscale.com/docs/features/subnet-routers).

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

See [`esp32-s3-tailscale-gateway_22102026.yaml`](esp32-s3-tailscale-gateway_22102026.yaml) for the full configuration, including LED status and factory reset. The minimal example above uses a different setup SSID/password and omits these features. Older configurations are kept in [`stare/`](stare/).

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
- The current full YAML serves the web interface over HTTP without authentication, and does not configure an OTA password or ESPHome API encryption. Configure it only on a trusted network; add appropriate protections before broader deployment.
- The setup AP password in the full YAML is shared. Change it before distributing devices.
- Password masking in the web interface does not encrypt HTTP traffic.
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
