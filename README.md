ESP32-S3 Tailscale Gateway

This repository is a modified fork of the original
Csontikka/esphome-tailscale
project.

The original project turns an ESPHome device into a native Tailscale node.

This fork extends that functionality by allowing an ESP32-S3 to operate as a lightweight Tailscale subnet gateway for an existing LAN.

The primary use case is remote access to low-bandwidth LAN and IoT devices through Tailscale without requiring a Raspberry Pi, PC, VPN server, port forwarding, or changes to the LAN router.

Architecture
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

The ESP32-S3 remains connected to the existing Wi-Fi network as a normal station. It does not need to become the primary router for the LAN.

Changes from upstream

Compared with the original Csontikka/esphome-tailscale project, this fork adds the following functionality:

Tailscale subnet routing

New component options:

tailscale:
  advertise_routes: "192.168.0.0/24"
  gateway_snat: true

advertise_routes announces the selected LAN subnet to the Tailscale control plane.

gateway_snat enables Source NAT / NAPT between the Tailscale WireGuard interface and the local network.

This allows remote Tailscale peers to communicate with devices on the LAN without adding static routes to the local router.

ESP-IDF IP forwarding and NAPT

The component enables the required lwIP functionality:

CONFIG_LWIP_IP_FORWARD
CONFIG_LWIP_IPV4_NAPT

A MicroLink helper was added to enable and disable NAPT directly on the WireGuard network interface.

NAPT follows the VPN lifecycle:

Tailscale CONNECTED
        ↓
Gateway NAPT enabled
        ↓
Subnet Router active

Tailscale disconnected / stopped
        ↓
Gateway NAPT disabled

NAPT is also explicitly disabled before the MicroLink/WireGuard interface is destroyed.

Subnet Router status entity

A new ESPHome binary sensor reports whether subnet routing is actually operational.

binary_sensor:
  - platform: tailscale
    subnet_router:
      name: "Subnet Router"

The sensor becomes active only when:

Tailscale is connected,
subnet routing is configured,
gateway SNAT is enabled,
and NAPT has been successfully activated.
Optional YAML authentication key

In the upstream component the Tailscale auth key is mandatory:

tailscale:
  auth_key: !secret tailscale_auth_key

In this fork the YAML auth key is optional.

An ESP32-S3 can therefore boot with no Tailscale credentials compiled into the firmware.

tailscale:
  hostname: "esp32-tailscale-gateway"
  advertise_routes: "192.168.0.0/24"
  gateway_snat: true

If no authentication key is available, Wi-Fi, ESPHome API and the web server continue to operate normally while the Tailscale client waits for configuration.

MicroLink is not started with an empty or dummy authentication key.

Runtime Tailscale Auth Key configuration

The authentication key can be entered through an ESPHome text entity:

text:
  - platform: tailscale
    auth_key:
      name: "Tailscale Auth Key"
      mode: password

The key is:

entered from the ESPHome web interface or Home Assistant,
stored persistently in ESP32 NVS,
used as a runtime override,
retained across device reboots,
masked when exposed through the entity.

This makes it possible to build a generic firmware image without embedding a user-specific Tailscale key.

First-run configuration

A gateway can be configured without storing Wi-Fi credentials or a Tailscale auth key in the firmware.

Typical first boot:

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

This allows the same firmware image to be deployed on multiple ESP32-S3 devices and configured after flashing.

Current test status

Reference hardware:

ESP32-S3-DevKitC-1
ESP-IDF
PSRAM enabled
ESPHome 2026.7.x
Wi-Fi uplink
advertised subnet: 192.168.0.0/24

Remote LAN access has been successfully tested from:

LTE / mobile data,
external Wi-Fi networks,
locations outside the local network.

The gateway is intended primarily for low-bandwidth remote access to ESPHome, Home Assistant and other IoT/LAN devices.

Upstream project

The Tailscale ESPHome integration and the majority of the underlying component originate from:

Csontikka / ESPHome Tailscale

https://github.com/Csontikka/esphome-tailscale

This fork builds on that work and focuses specifically on the ESP32-S3 subnet-gateway use case.
