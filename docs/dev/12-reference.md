# Reference: Types, Constants & Glossary

A quick-lookup companion to the narrative docs. Most definitions live in
`defs.h`, `cprintf.h`, and `libwps/libwps.h`.

## 1. Enums (`defs.h`, `cprintf.h`)

### `debug_level` (cprintf.h:4)
Logging verbosity for `cprintf()`. Lower = more important. `-v` raises it, `-q`
forces `CRITICAL`.

| Value | # | Shown by default (INFO)? |
|-------|---|--------------------------|
| `CRITICAL` | 0 | yes |
| `INFO` | 1 | yes |
| `WARNING` | 2 | no (needs `-v`) |
| `VERBOSE` | 3 | no (needs `-vv`) |
| `DEBUG` | 4 | no (needs `-vvv`; also enables wpa_supplicant hexdumps) |

### `key_state` (defs.h:108)
PIN keyspace progress.

| Value | # | Meaning |
|-------|---|---------|
| `KEY1_WIP` | 0 | Brute forcing the first half. |
| `KEY2_WIP` | 1 | First half found; brute forcing the second. |
| `KEY_DONE` | 2 | Full PIN recovered. |

### `wps_result` (defs.h:123)
Return of `do_wps_exchange()`.

| Value | Meaning |
|-------|---------|
| `KEY_ACCEPTED` | This message/half accepted (or PIN done). |
| `KEY_REJECTED` | NACK/timeout proving a half is wrong → advance pin. |
| `RX_TIMEOUT` | Transient timeout → retry same pin. |
| `EAP_FAIL` | EAP-Failure without NACK → retry. |
| `UNKNOWN_ERROR` | Anything else. |

### `wps_type` (defs.h:157)
WPS/EAP message types recognized by `process_packet()`.

| Name | Value | |
|------|-------|--|
| `TERMINATE` | -1 | EAP-Failure received |
| `UNKNOWN` | 0 | not a recognized WPS message |
| `IDENTITY_REQUEST` | 1 | EAP identity request |
| `IDENTITY_RESPONSE` | 2 | (sent by us) |
| `M1`..`M8` | 0x04,0x05,0x07,0x08,0x09,0x0A,0x0B,0x0C | WPS registration messages |
| `DONE` | 0x0F | WSC_Done |
| `NACK` | 0x0E | WSC_NACK |
| `WPS_PT_DEAUTH` | 0xFF | deauth frame seen |

### `nack_code` (defs.h:132)
WPS configuration-error / NACK reason codes. Notable ones used in logic:
`MESSAGE_TIMEOUT` (16, "fake NACK" detection) and `SETUP_LOCKED` (15, AP WPS
locked). Full list at `defs.h:132-155`.

### `wfa_elements` (defs.h:176)
WPS attribute/element type IDs (e.g. `MESSAGE_TYPE=0x1022`,
`CONFIGURATION_ERROR=0x1009`, `PUBLIC_KEY=0x1032`, `ENROLLEE_NONCE=0x101A`,
`ENROLLEE_HASH_1/2=0x1014/0x1015`, `AP_SETUP_LOCKED=0x1057`,
`VENDOR_EXTENSION=0x1049`, `VERSION=0x104A`). Used when scanning WPS payloads in
`exchange.c` and `libwps.c`.

### `encryption_type` (defs.h:101)
`NONE`, `WEP`, `WPA` — used by the (broken/unused) `supported_encryption()`.

### `wps_locked_state` (libwps.h:26)
`UNLOCKED`, `WPSLOCKED`, `UNSPECIFIED` — wash's view of the AP's WPS lock.

## 2. Packed wire structs (`defs.h`, under `#pragma pack(1)`)

| Struct | Maps to | Source |
|--------|---------|--------|
| `radio_tap_header` | radiotap header (version/pad/len/flags[/rate]/txflags) | `defs.h:338` |
| `dot11_frame_header` | 802.11 MAC header (fc, duration, addr1-3, frag_seq) | `defs.h:351` |
| `authentication_management_frame` | open-system auth body | `defs.h:361` |
| `association_request_management_frame` | assoc-request body | `defs.h:368` |
| `association_response_management_frame` | assoc-response body | `defs.h:374` |
| `beacon_management_frame` | beacon fixed params (timestamp, interval, capability) | `defs.h:381` |
| `llc_header` | LLC/SNAP (dsap/ssap/control/org/type) | `defs.h:388` |
| `dot1X_header` | 802.1X (version/type/len) | `defs.h:397` |
| `eap_header` | EAP (code/id/len/type) | `defs.h:404` |
| `wfa_expanded_header` | WFA expanded EAP (id/type/opcode/flags) | `defs.h:412` |
| `wfa_element_header` | WPS TLV element (type/length) | `defs.h:420` |
| `tagged_parameter` | 802.11 IE tag (number/len) | `defs.h:426` |

libwps redefines its own `radio_tap_header`, `dot11_frame_header`,
`management_frame`, `tagged_parameter`, and `data_element` to stay independent
(`libwps.h:101-136`).

## 3. Key constants (`defs.h`)

| Macro | Value | Meaning |
|-------|-------|---------|
| `P1_SIZE` / `P2_SIZE` | 10000 / 1000 | First/second-half candidate counts. |
| `DEFAULT_MAX_NUM_PROBES` | 15 | wash probes per AP. |
| `MAX_ASSOC_FAILURES` | 10 | Association retry warning threshold. |
| `MAC_ADDR_LEN` | 6 | MAC length. |
| `WPS_TAG_NUMBER` / `VENDOR_SPECIFIC_TAG` | 0xDD | Vendor-specific IE id. |
| `DOT1X_AUTHENTICATION` | 0x888E | 802.1X ethertype. |
| `SIMPLE_CONFIG` | 0x00000001 | WFA WPS type. |
| `EAPOL_START_MAX_TRIES` / `WARN_FAILURE_COUNT` | 10 / 10 | Retry/warn thresholds. |
| `M57_DEFAULT_TIMEOUT` / `M57_MAX_TIMEOUT` | 400000 / 1000000 µs | M5/M7 timeouts. |
| `DEFAULT_DELAY` | 1 s | Inter-attempt delay. |
| `DEFAULT_TIMEOUT` | 10 s | RX timeout. |
| `DEFAULT_LOCK_DELAY` | 60 s | Wait when WPS locked. |
| `WPS_DEVICE_NAME` … `WPS_RF_BANDS` | see [07-crypto-keys.md](07-crypto-keys.md) | `--win7` registrar identity. |
| `IEEE80211_FTYPE_*` / `IEEE80211_STYPE_*` | bitmasks | 802.11 frame type/subtype decoding. |

## 4. CLI option cross-reference

### reaver (`argsparser.c:46`, usage `wpscrack.c:141`)
`-i` interface · `-b` bssid · `-m` mac · `-e` essid · `-c` channel · `-s` session
· `-C` exec · `-f` fixed · `-5` 5GHz · `-v` verbose · `-q` quiet · `-p` pin ·
`-d` delay · `-l` lock-delay · `-g` max-attempts · `-x` fail-wait ·
`-r` recurring-delay · `-t` timeout · `-T` m57-timeout · `-A` no-associate ·
`-N` no-nacks · `-S` dh-small · `-L` ignore-locks · `-E` eap-terminate ·
`-J` timeout-is-nack · `-F` ignore-fcs · `-w` win7 · `-K`/`-Z` pixie-dust ·
`-O` output-file · `-M` mac-changer · `-6` repeat-m6 · `-h` help.

### wash (`wpsmon.c:163`, usage `wpsmon.c:621`)
`-i` interface · `-f` file · `-c` channel · `-n` probes · `-O` output-file ·
`-2` 2GHz · `-5` 5GHz · `-s` scan · `-u` survey · `-a` all · `-j` json ·
`-U` utf8 · `-p` progress · `-F` ignore-fcs · `-b` bssid · `-h` help.

> Defaults are applied in `init_default_settings()` (`argsparser.c:223`) for
> reaver and inline in `wash_main` (`wpsmon.c:184`) for wash.

## 5. Glossary

| Term | Definition |
|------|------------|
| **WPS** | Wi-Fi Protected Setup. Simplified Wi-Fi onboarding; its PIN method is what Reaver attacks. |
| **Registrar / Enrollee** | WPS roles. Reaver acts as a *registrar* against the AP (the *enrollee* here). |
| **PIN** | 8-digit WPS PIN; the 8th digit is a checksum of the first 7. |
| **P1 / P2** | First half (4 digits) and second half (3 digits) of the PIN keyspace. |
| **M1–M8** | The WPS registration protocol messages. M4/M6 NACKs reveal which half is wrong. |
| **EAP / EAPOL** | Extensible Authentication Protocol (over LAN). WPS runs as EAP-Expanded (WFA). |
| **EAPOL-Start** | Frame that kicks off an EAP session (`send_eapol_start`). |
| **WSC** | Wi-Fi Simple Config, the WPS protocol name; `WSC_NACK`/`WSC_Done` are messages. |
| **NACK** | Negative ack. A NACK after M3/M5 signals the corresponding PIN half is wrong. |
| **DH** | Diffie–Hellman key exchange (group 5 / 1536-bit here). |
| **PKE / PKR** | Enrollee (AP) / Registrar (our) DH public keys. |
| **E-Nonce / R-Nonce** | Enrollee / Registrar random nonces. |
| **E-Hash1/2** | Enrollee commitments to its secret nonces, per PIN half. |
| **AuthKey / KeyWrapKey / EMSK** | Session keys derived via KDK + KDF. |
| **PSK1 / PSK2** | HMACs of the two PIN halves under AuthKey. |
| **Pixie Dust** | Offline attack recovering the PIN from one handshake via weak AP RNG. |
| **pixiewps** | External tool that performs the Pixie Dust computation. |
| **Radiotap** | Pseudo-header carrying radio metadata (channel, signal, flags) on captured frames. |
| **FCS** | Frame Check Sequence; trailing CRC-32 of an 802.11 frame. |
| **IE / Tagged parameter** | Information Element; TLV in management frames (SSID, rates, WPS, …). |
| **OUI** | Organizationally Unique Identifier; first 3 MAC bytes → chipset vendor. |
| **BSSID** | The AP's MAC address (target). |
| **wext** | Linux Wireless Extensions (legacy channel control API). |
| **nl80211 / libnl** | Modern netlink-based Wi-Fi control (alternative channel backend). |
| **`.wpc`** | Reaver session file storing PIN indices, status, and candidate arrays. |
| **`globule`** | The single global state struct (`struct globals`). |
| **`wpabuf`** | wpa_supplicant growable buffer type used by the WPS code. |
| **monitor mode** | NIC mode that captures raw 802.11 frames; required for both tools. |

## 6. Where to read next

- New to the code? Start with [01-architecture.md](01-architecture.md).
- Understanding the online attack: [03-reaver-attack-flow.md](03-reaver-attack-flow.md)
  + [05-pin-keyspace.md](05-pin-keyspace.md).
- Pixie Dust: [06-pixie-dust.md](06-pixie-dust.md) + [07-crypto-keys.md](07-crypto-keys.md).
- Scanning: [08-wash-scanner.md](08-wash-scanner.md).
- Building/porting: [02-build-system.md](02-build-system.md) +
  [10-wireless-interface.md](10-wireless-interface.md).
