# CHG-002 — IoT & Camera Network Segmentation

| Field | Detail |
|---|---|
| **Change ID** | CHG-002 |
| **Dates** | 2026-11-22 to 2026-11-23 |
| **Owner** | Jeff (Home Network Administrator) |
| **Status** | ✅ Closed — implemented and tested |
| **Depends on** | CHG-001 (closed) |
| **Reason** | Separate IoT and camera devices from trusted devices, carrying over the segmentation policy from the previous router |
| **Framework** | NIST SP 800-53: SC-7, AC-4, CM-8 |

> Public version. MAC addresses, host-specific IPs, device hostnames that reveal camera models, BSSIDs and Wi-Fi credentials are withheld. Broadcast SSIDs are neutral names; networks are referred to by **role** in this document. Private addresses are written as `x.x.N.x`, where the third octet identifies the network zone.

---

## 1. Design Decisions

| Decision | Rationale |
|---|---|
| **Use the vendor's built-in IoT and Guest networks**, not hand-built VLANs | Wi-Fi devices don't pass through the switch, so a separate SSID bound to its own bridge, subnet and zone gives the same separation. A vendor-supported configuration survives firmware upgrades and is less to maintain |
| **Cameras on their own network**, separate from other IoT | Higher sensitivity (in-home video), a separate credential, and independent policy control. The previous policy is kept unchanged |
| **Neutral broadcast SSIDs** | SSIDs are public; a name like "Camera" tells anyone nearby which devices to target |
| **Unique passphrase per network** | A credential leaked from one device class doesn't give access to another |
| **Vendor firewall zones verified, not assumed** | The zone policy was checked in the firewall configuration before any devices moved |

## 2. Network Design

| Network (role) | Subnet | Firewall zone | Devices |
|---|---|---|---|
| Trusted | `x.x.8.x/24` | lan | Laptops, phones |
| **Camera network** | `x.x.9.x/24` | guest → wan only | Cameras ×2 |
| **IoT network** | `x.x.10.x/24` | iot → wan only | TVs ×2, streaming stick |
| IPTV quarantine | `x.x.30.x/24` | IPTV → wan only | IPTV box (CHG-001) |

## 3. IoT Network

- 2.4 GHz + 5 GHz bands, **WPA2-PSK/WPA3-SAE** (each device uses the strongest mode it supports; WPA/TKIP rejected), SSIDs shown
- **AP Isolation ON** — Wi-Fi clients can't communicate with each other
- **Block WAN Subnets ON** — no access to the upstream ISP network segment
- Zone `iot`: Input REJECT / Forward REJECT, forwards to `wan` only; vendor rules allow DHCP (UDP 67–68) and DNS (TCP/UDP 53) to the router only

## 4. Camera Network

- 2.4 GHz, **WPA2-PSK/WPA3-SAE**, unique passphrase, neutral SSID
- **AP Isolation ON**, **Block WAN Subnets ON**
- Zone `guest`: Input REJECT / Forward REJECT, forwards to `wan` only
- Viewing verified from the trusted network through the vendor cloud, so **no inbound exception** is required

## 5. Risk Decisions

| Item | Decision |
|---|---|
| **Randomized BSSID off** (vendor warns a fixed BSSID may allow tracking) | **Accepted.** The router is stationary at the home location, so tracking reveals only a location that's already observable. Likelihood low, impact low. Benefit: stable IoT reconnection. Revisit if the router is ever used while travelling |
| **AP Isolation across bands** — may not apply between 2.4 GHz and 5 GHz clients on the same bridge | **Accepted.** All IoT-network devices are the same trust class (TV/streaming). Cameras are on a separate network and unaffected |
| **Camera footage depends on vendor cloud security** | **Residual risk**, outside the scope of network controls. A self-hosted recording server is flagged as a future project |

## 6. Migration & Inventory

- TVs and streaming stick moved to the IoT network; cameras moved to the camera network; all devices functional
- A TV previously found on the **trusted** network (matched by MAC) now leases from the IoT subnet, which corrects a deviation from the earlier policy
- **Inventory reconciled** from the live client list. Unidentified devices were resolved as rotating private MAC addresses (Apple devices) and a privacy-focused phone that randomizes its MAC by design. One prior device is no longer connected and is being monitored
- Open item: streaming stick MAC confirmation

## 7. Verification & Testing

| Test | Result | Evidence |
|---|---|---|
| IoT devices lease from `x.x.10.x/24` | ✅ | iot-05 |
| Cameras lease from `x.x.9.x/24` | ✅ | cam-03 |
| Trusted → IoT TV: ping + nmap 80, 443, 8001, 8002 | ✅ PASS — router reject, `closed` / `reset ttl 64` | iot-06 |
| Trusted → Camera: ping + nmap 80, 443, **554 (RTSP)**, 2020 | ✅ PASS — router reject, `closed` / `reset ttl 64` | cam-04 |

## 8. Evidence

Folder: `screenshots/chg-002-iot-cameras/`

| File | Shows |
|---|---|
| [`iot-01-wifi-settings.png`](../screenshots/chg-002-iot-cameras/iot-01-wifi-settings.png), [`iot-01b-wifi-5ghz.png`](../screenshots/chg-002-iot-cameras/iot-01b-wifi-5ghz.png) | IoT Wi-Fi security settings (both bands) |
| [`iot-02-network-settings.png`](../screenshots/chg-002-iot-cameras/iot-02-network-settings.png) | Subnet, AP Isolation, Block WAN Subnets |
| [`iot-03-firewall-zone.png`](../screenshots/chg-002-iot-cameras/iot-03-firewall-zone.png) | `iot ⇒ wan` only |
| [`iot-04-traffic-rules.png`](../screenshots/chg-002-iot-cameras/iot-04-traffic-rules.png) | DHCP/DNS exceptions |
| [`iot-05-dhcp-lease.png`](../screenshots/chg-002-iot-cameras/iot-05-dhcp-lease.png) | All three IoT devices lease from the isolated IoT subnet (x.x.10.x/24). One unrelated lease is redacted |
| [`iot-06-test-blocked.png`](../screenshots/chg-002-iot-cameras/iot-06-test-blocked.png) | Trusted → IoT blocked |
| [`cam-01-wifi-settings.png`](../screenshots/chg-002-iot-cameras/cam-01-wifi-settings.png) | Camera Wi-Fi security settings |
| [`cam-02-network-settings.png`](../screenshots/chg-002-iot-cameras/cam-02-network-settings.png) | Camera subnet, AP Isolation, Block WAN Subnets |
| [`cam-03-dhcp-lease.png`](../screenshots/chg-002-iot-cameras/cam-03-dhcp-lease.png) | Cameras in the camera subnet |
| [`cam-04-test-blocked.png`](../screenshots/chg-002-iot-cameras/cam-04-test-blocked.png) | Trusted → camera blocked (incl. RTSP) |
