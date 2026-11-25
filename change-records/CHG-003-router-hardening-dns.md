# CHG-003 — Router Management Hardening & DNS Malware Blocking

| Field | Detail |
|---|---|
| **Change ID** | CHG-003 |
| **Date** | 2026-11-23 |
| **Owner** | Jeff (Home Network Administrator) |
| **Status** | ✅ Closed — implemented and tested |
| **Reason** | Restore controls lost in the router replacement. The previous router disabled remote management, used HTTPS for administration, restricted admin access and filtered DNS for malware. None of these carried over automatically |
| **Framework** | NIST SP 800-53: AC-17, AC-3, SC-8, CM-7, SI-3 / SC-7 (network-level DNS filtering), CM-8 |

> Public version. Router model, firmware version and host-specific IP addresses are withheld; public DNS resolver addresses are shown because they are part of the evidence.

> **Lesson recorded:** CHG-001 did not include a *control carry-over check*. Replacing a device silently removed controls listed in the risk exception. Future device replacements must first list every control the outgoing device provides.

---

## 1. Admin Access — AC-17, SC-8, CM-7

| Setting | Before | After | |
|---|---|---|---|
| Admin panel: Force HTTPS | Off | **On** | Configured |
| LuCI: Force HTTPS | Off | **On** | Configured |
| SSH | **On (vendor default)** | **Off** | Configured |
| Auto-logout | 5 min | 5 min | Verified (default) |
| Ping from WAN | Off | Off | Verified (default) |
| HTTPS remote access | Off | Off | Verified (default) |
| SSH remote access | Off | Off (unavailable while SSH is off) | Verified (default) |
| Vendor cloud management | Not enrolled | Not enrolled | Verified |

**Note:** the admin interface uses a self-signed certificate. Traffic is encrypted, but the browser can't verify the certificate's identity, so an unexpected certificate warning should be treated as a red flag.

## 2. Admin Restriction: Control Substitution — AC-3

| Previous router | This router |
|---|---|
| Admin access restricted to two devices **by MAC address** | No MAC allow-list available. Admin access is restricted **by network zone**: IoT, camera and IPTV zones have Input REJECT (only DHCP/DNS allowed), so they can't reach the admin interface |

**Why this is stronger:** MAC addresses can be spoofed by any device on the network. Zone restriction is enforced by the firewall and doesn't depend on an identifier the client controls.

**Test (phone, VPN off, airplane mode with Wi-Fi only):**

| # | From | To | Result |
|---|---|---|---|
| 03a | IoT network | router, IoT-side address | ✅ `ERR_CONNECTION_REFUSED` |
| 03b | IoT network | router, trusted-side address (where the admin panel listens) | ✅ `ERR_CONNECTION_REFUSED` |
| 03c | Trusted network | router, trusted-side address | ✅ Login page loads over HTTPS (positive control) |

**Attempt 1 marked invalid:** the phone's VPN was active and the browser lacked local-network permission, so the request was blocked on the client and the router's firewall was never exercised. Re-tested with both factors removed.

## 3. DNS Malware Blocking — restored

| Step | Result |
|---|---|
| **dns-00** Before | Mode **Automatic** (ISP resolvers); rebinding protection off. Confirms the control was lost |
| **dns-01** Configured | Manual DNS **1.1.1.2 / 1.0.0.2** (Cloudflare malware blocking); **DNS rebinding protection on** |
| **dns-02** Verified | `malware.testcategory.com` → default resolver `0.0.0.0`; `@1.1.1.2` `0.0.0.0`; `@1.1.1.1` real addresses |
| **dns-03** Override | "Override DNS Settings of All Clients" **on**. The same query to `@1.1.1.1` now returns `0.0.0.0`, so devices that hard-code their own DNS are also filtered |

**Known limitation:** the override applies to **unencrypted** DNS. Applications using encrypted DNS (DoH/DoT) may bypass it.

## 4. Monitoring: Claim Replaced with a Real Control — CM-8

The earlier risk exception listed *"ongoing monitoring of IoT VLAN traffic"* with no supporting tool, process or evidence. It is replaced with:

**Monthly connected-client review (detective control)**
1. List live clients per zone (Trusted / Camera / IoT / IPTV)
2. Compare against the approved inventory
3. Investigate unknown devices and record the outcome
4. Confirm the IPTV device is still only in the quarantine VLAN
5. Log the review (date, findings, actions)

This primarily satisfies CM-8; it supports SI-4 in a basic, manual form. **First review completed 2026-11-22** (inventory baseline, CHG-002 §6).

## 5. Recovery

A post-change configuration archive was taken after CHG-003 and stored offline. It is the current recovery point covering CHG-001 to CHG-003.

## 6. Evidence

Folder: `screenshots/chg-003-hardening-dns/`

| File | Shows |
|---|---|
| [`hard-01-admin-access.png`](../screenshots/chg-003-hardening-dns/hard-01-admin-access.png) | Force HTTPS, SSH off, remote access off |
| [`hard-02-goodcloud-off.png`](../screenshots/chg-003-hardening-dns/hard-02-goodcloud-off.png) | Vendor cloud management not enrolled |
| [`hard-03a.png`](../screenshots/chg-003-hardening-dns/hard-03a.png), [`hard-03b.png`](../screenshots/chg-003-hardening-dns/hard-03b.png), [`hard-03c.png`](../screenshots/chg-003-hardening-dns/hard-03c.png) | Zone-based admin restriction + positive control |
| [`dns-00-before.png`](../screenshots/chg-003-hardening-dns/dns-00-before.png) | ISP DNS (control missing) |
| [`dns-01-manual-dns.png`](../screenshots/chg-003-hardening-dns/dns-01-manual-dns.png) | Malware-blocking DNS + rebinding protection |
| [`dns-02-filtering-verified.png`](../screenshots/chg-003-hardening-dns/dns-02-filtering-verified.png) | Filtering verified by comparison |
| [`dns-03a-override-on.png`](../screenshots/chg-003-hardening-dns/dns-03a-override-on.png), [`dns-03b-override-verified.png`](../screenshots/chg-003-hardening-dns/dns-03b-override-verified.png) | DNS bypass intercepted |
