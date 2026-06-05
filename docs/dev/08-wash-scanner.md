# Wash Scanner

`wash` discovers WPS-capable APs and reports their WPS metadata (version, locked
state, vendor, etc.). It is the same binary as `reaver`, entered through
`wash_main` (`wpsmon.c:155`) when `argv[0]` contains "wash"
([01-architecture.md](01-architecture.md)).

Relevant files: `wpsmon.c` (orchestration + printing), `libwps/libwps.c` +
`libwps/libwps.h` (the standalone WPS IE parser), and shared 802.11 helpers in
`80211.c`.

## 1. Two data sources, two modes

**Source** (`wpsmon.c:159`):
- `INTERFACE` — live capture from a monitor-mode interface (`-i`).
- `PCAP_FILE` — one or more capture files (`-f file1 file2 ...`), processed
  passively.

**Mode** (`enum`, set by `-s`/`-u`):
- `SURVEY` (default) — passively listen to beacons/probe responses.
- `SCAN` — additionally **send probe requests** to elicit WPS info faster
  (only when live and not passive).

## 2. `wash_main` — setup

Source: `wpsmon.c:155`.

1. `globule_init()`, then wash-specific defaults: no auto channel select, band
   from `-2`/`-5` (default BG), debug=INFO, FCS validation on, logs to stderr,
   default probe count (`wpsmon.c:184-191`).
2. Unbuffered stdout/stderr so piped output is live (`wpsmon.c:193`).
3. `getopt_long` parses options (table below). `-O` opens a pcap output fd.
4. Validate source vs interface (`-i` and `-f` are mutually exclusive,
   `wpsmon.c:281`).
5. Loop over capture sources; for each, `capture_init()` then `monitor()`
   (`wpsmon.c:299-332`). For an interface this loops once and blocks; for files
   it processes each in turn.

### Wash options

| Short | Long | Effect | Source |
|-------|------|--------|--------|
| `-i` | `--interface` | Capture interface. | `wpsmon.c:203` |
| `-f` | `--file` | Read from pcap files (passive). | `wpsmon.c:200` |
| `-c` | `--channel` | Fixed channel (sets `fixed_channel`). | `wpsmon.c:209` |
| `-n` | `--probes` | Max probes per AP in scan mode. | `wpsmon.c:226` |
| `-O` | `--output-file` | Mirror packets of interest to pcap. | `wpsmon.c:213` |
| `-5` | `--5ghz` | Add the A/N band. | `wpsmon.c:220` |
| `-2` | `--2ghz` | Add the B/G band. | `wpsmon.c:223` |
| `-s` | `--scan` | Scan mode (send probes). | `wpsmon.c:232` |
| `-u` | `--survey` | Survey mode (default). | `wpsmon.c:235` |
| `-F` | `--ignore-fcs` | Disable FCS validation. | `wpsmon.c:238` |
| `-a` | `--all` | Show APs even without WPS. | `wpsmon.c:241` |
| `-j` | `--json` | Emit extended WPS info as JSON. | `wpsmon.c:229` |
| `-U` | `--utf8` | Show raw UTF-8 ESSID (no sanitizing). | `wpsmon.c:244` |
| `-p` | `--progress` | Show crack progress from `.wpc`. | `wpsmon.c:247` |
| `-b` | `--bssid` | Restrict to one BSSID. | `wpsmon.c:206` |

## 3. `monitor()` — the capture loop

Source: `wpsmon.c:344`.

- For a live interface:
  - If `-c` was given, set that channel once.
  - Otherwise install a **POSIX interval timer** that fires `SIGALRM` every
    `CHANNEL_INTERVAL`, whose handler `sigalrm_handler()` (`wpsmon.c:611`) calls
    `next_channel()` — this is the channel-hopping mechanism. Start channel is 1
    (BG) or 34 (AN) (`wpsmon.c:386-389`).
  - Install a non-restarting `SIGINT` handler so Ctrl-C breaks the pcap loop
    cleanly (`wpsmon.c:392`, `sigint_handler` `wpsmon.c:149` calls
    `pcap_breakloop`).
- Print the table header once (unless JSON mode) (`wpsmon.c:400`).
- Loop `next_packet()` → `parse_wps_settings()` until SIGINT
  (`wpsmon.c:414`).

> Note: wash uses its **own** SIGALRM/SIGINT handlers defined in `wpsmon.c`,
> distinct from reaver's `sigalrm.c`/`sigint.c` (which serve the attack's receive
> timer). Same signals, different jobs.

## 4. `parse_wps_settings()` — per-packet processing

Source: `wpsmon.c:429`.

```mermaid
flowchart TD
    P([packet]) --> MF{beacon or\nprobe-resp?}
    MF -->|no| DROP([ignore])
    MF -->|yes| BSSIDF{matches -b\nfilter?}
    BSSIDF -->|no| DROP
    BSSIDF -->|yes| TAGS["parse_beacon_tags()\nchannel, ssid, vendor"]
    TAGS --> FREQ{channel known?}
    FREQ -->|no| RADIO["freq_to_chan(rt_channel_freq)\nfallback current chan"]
    FREQ -->|yes| WPS
    RADIO --> WPS["parse_wps_parameters()\n(libwps)"]
    WPS --> DEDUP{is_done()?\n(seen before / complete)}
    DEDUP -->|done| SKIP([skip output])
    DEDUP -->|new| PROBE{scan mode &&\nshould_probe?}
    PROBE -->|yes| SENDP["send_probe_request()"]
    PROBE -->|no| PRINT
    SENDP --> PRINT["print row / JSON\n(if WPS active or -a)"]
    PRINT --> COMPLETE{probe-resp or\nno WPS?}
    COMPLETE -->|yes| MARK["mark_ap_complete()"]
    COMPLETE -->|no| DONE([wait for more])
```

Key behaviors:
- Frame classification mirrors `80211.c` (`wpsmon.c:446-456`).
- The channel is determined from the beacon tag, else the radiotap frequency,
  else the current scan channel (`wpsmon.c:475-480`).
- RSSI from `signal_strength()`; vendor OUI cached per BSSID
  (`wpsmon.c:481`, `:495`).
- When a `-b` target is found, the channel timer is stopped and the radio parked
  on that channel (`wpsmon.c:484-491`).

## 5. Deduplication — the `seen_list`

Source: `wpsmon.c:50-146`. wash keeps a fixed table of up to `MAX_APS` (512)
seen BSSIDs with per-AP flags:

| Flag | Meaning |
|------|---------|
| `SEEN_FLAG_PRINTED` | Row already printed. |
| `SEEN_FLAG_COMPLETE` | No more info expected (probe-resp seen, or non-WPS). |
| `SEEN_FLAG_PBC` | Push-button config active. |
| `SEEN_FLAG_LOCKED` | WPS locked. |
| `SEEN_FLAG_WPS_ACTIVE` | WPS present. |

Helpers: `list_insert()` (find/add, wraps when full), `was_printed()`,
`mark_ap_complete()`, `is_done()` (also updates PBC/locked/active flags),
`should_probe()` / `update_probe_count()` (probe budget per AP),
`set_ap_vendor()` / `get_ap_vendor()`. `is_pbc()` (`wpsmon.c:77`) detects
push-button mode (selected registrar = 1 and device password id = 4).

## 6. The WPS IE parser — `libwps`

`libwps/` is **authored** (not vendored) and is deliberately self-contained: it
does not touch the global state, so it could be reused as a library.

- `parse_wps_parameters()` (`libwps.c:372`) is the public entry: it locates the
  IE/tag area after the management frame header and calls `parse_wps_tags()`.
- `get_wps_data()` (`libwps.c:405`) scans for a vendor-specific tag (`0xDD`)
  whose vendor id is the WPS OUI `00 50 F2 04` and returns the WPS data blob.
- `parse_wps_tags()` (`libwps.c:202`) iterates a fixed list of WPS element types
  and copies each into the `struct libwps_data` fields, using
  `get_wps_data_element()` (`libwps.c:449`). Binary fields (UUID, selected
  registrar, config methods, etc.) are hex-encoded via `hex2str()`
  (`libwps.c:521`).
- Version 2 detection: the `VENDOR_EXTENSION` element is walked for the WFA
  subelement that carries the WPS 2.0 version (`libwps.c:312-326`).

### `struct libwps_data` (libwps.h:33)

Holds: `version`, `state`, `locked`, and string fields `manufacturer`,
`model_name`, `model_number`, `device_name`, `device_password_id`, `ssid`,
`uuid`, `serial`, `selected_registrar`,
`selected_registrar_config_methods`, `response_type`, `primary_device_type`,
`config_methods`, `rf_bands`, `os_version`. The `locked` field uses
`enum wps_locked_state { UNLOCKED, WPSLOCKED, UNSPECIFIED }` (`libwps.h:26`).

> libwps has its **own** copies of `radio_tap_header`, `dot11_frame_header`, and
> `tagged_parameter` plus its own `libwps_has_rt_header()` heuristic
> (`libwps.h:113`, `libwps.c:484`) so it stays independent of `80211.c`. The
> header comment at `libwps.h:142` explains this duplication.

## 7. Output formats

### Table (default)
Header at `wpsmon.c:404-409`; rows at `wpsmon.c:539-548`:

```
BSSID               Ch  dBm  WPS  Lck  Vendor    [Progr]  ESSID
```

- `WPS` shows the version as `M.m` (e.g. `2.0`) or `PBC` for push-button.
- `Lck` is Yes/No from the WPS locked state.
- `Vendor` comes from the chipset OUI lookup (`get_vendor_string`, below).
- `Progr` (with `-p`) is the crack percentage from any `.wpc` file
  (`get_crack_progress()`, [09-state-and-session.md](09-state-and-session.md)).
- ESSID is sanitized unless `-U` is given and `verifyssid()` passes.

### JSON (`-j`)
`wps_data_to_json()` (`libwps.c:40`) builds a JSON object incrementally
(append + free), emitting only the fields that are present, e.g.:

```json
{"bssid":"..","essid":"..","channel":6,"rssi":-50,"vendor_oui":"0050F2",
 "wps_version":16,"wps_state":2,"wps_locked":2,"wps_manufacturer":"..", ...,
 "dummy":0}
```

The trailing `"dummy":0` lets every real field end with a comma without special
casing the last element. JSON lines are flushed immediately for piping.

## 8. Vendor detection

`get_vendor_string()` (`utils/vendor.c:3`) maps a 3-byte OUI to a short
8-character chipset name (Wireshark-style), e.g. Broadcom (`00 10 18`), Atheros
(`00 03 7f`), Ralink (`00 0c 43`/`00 17 a5`), Realtek (`00 e0 4c`), Mediatek,
Marvell, Quantenna, Lantiq, Microsoft. Unknown OUIs return `"Unknown "`. Knowing
the chipset hints at Pixie Dust vulnerability ([06-pixie-dust.md](06-pixie-dust.md)).

The OUI itself is found heuristically in `parse_beacon_tags()`
(`80211.c:605-617`) and cached per BSSID in the `seen_list`.

## 9. Probe requests (scan mode)

`send_probe_request()` (`wpsmon.c:595`) builds a probe with
`build_wps_probe_request()` ([04-packet-layer.md](04-packet-layer.md)) and
injects it. The probe budget per AP is `-n` (`get_max_num_probes`), tracked via
`should_probe()` / `update_probe_count()`.
