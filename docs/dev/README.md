# Reaver Developer Documentation

This directory contains **developer-oriented** documentation for
`reaver-wps-fork-t6x` (v1.6.6). It explains how the source code is laid out,
how the two programs work internally, and how the WPS attacks are implemented.

> For **user-facing** documentation (how to run the tools, command-line
> examples), see the files in the parent `docs/` directory
> (`README.REAVER`, `README.WASH`, `reaver.1`) and the top-level `README.md`.

All references use the `path:line` convention so you can jump straight to the
source, e.g. `cracker.c:87` is the `crack()` function.

---

## What is Reaver?

Reaver implements a **brute-force attack against Wi-Fi Protected Setup (WPS)
registrar PINs** in order to recover WPA/WPA2 passphrases, as described in
Stefan Viehböck's 2011 paper. The project ships **two programs built from a
single binary**:

| Program  | Role                                                              | Entry point |
|----------|------------------------------------------------------------------|-------------|
| `reaver` | Actively attacks one AP (online brute force **or** Pixie Dust)    | `reaver_main` in `wpscrack.c:40` |
| `wash`   | Passively/actively scans for WPS-enabled APs and reports metadata | `wash_main` in `wpsmon.c:155` |

`wash` is a **symlink** to `reaver`; `main.c:9` dispatches to the right
`*_main()` based on `argv[0]`'s basename. See
[01-architecture.md](01-architecture.md).

Two attack methods exist:

1. **Online brute force** — Reaver acts as a WPS *registrar*, repeatedly
   associating with the AP and walking the WPS PIN keyspace. The protocol leaks
   whether the first and second halves of the PIN are correct independently,
   shrinking the keyspace from 10^8 to ~11,000 attempts. See
   [03-reaver-attack-flow.md](03-reaver-attack-flow.md) and
   [05-pin-keyspace.md](05-pin-keyspace.md).
2. **Offline Pixie Dust** (`-K`/`-Z`) — Collects cryptographic values from a
   single exchange and hands them to the external `pixiewps` tool to recover the
   PIN offline. See [06-pixie-dust.md](06-pixie-dust.md).

---

## Platform & dependencies

- **Linux only.** Uses raw 802.11 monitor-mode capture/injection through
  `libpcap`, Linux Wireless Extensions (or libnl), and POSIX `timer_settime`.
  It does **not** build natively on macOS (a few `__APPLE__` code paths exist
  for development convenience but are not a supported target).
- Links against `-lpcap -lpthread -lrt -lm`.
- Runtime: `pixiewps` (optional, for Pixie Dust), `aircrack-ng` (optional).

See [02-build-system.md](02-build-system.md) for the full build pipeline.

---

## Documentation map

| # | Document | Topic |
|---|----------|-------|
| — | [README.md](README.md) | This index + project overview |
| 01 | [01-architecture.md](01-architecture.md) | Layered architecture, module dependency graph, dispatch |
| 02 | [02-build-system.md](02-build-system.md) | Makefile, configure, generated files, object groups |
| 03 | [03-reaver-attack-flow.md](03-reaver-attack-flow.md) | `crack()` loop + WPS M1–M8 state machine (sequence diagram) |
| 04 | [04-packet-layer.md](04-packet-layer.md) | 802.11 frame build/parse, radiotap, FCS/CRC, pcap I/O |
| 05 | [05-pin-keyspace.md](05-pin-keyspace.md) | PIN split (p1/p2), checksum, keyspace, session jump-queue |
| 06 | [06-pixie-dust.md](06-pixie-dust.md) | Offline attack, data collection hooks, pixiewps invocation |
| 07 | [07-crypto-keys.md](07-crypto-keys.md) | DH, KDF, AuthKey/KeyWrapKey/EMSK, PSK derivation |
| 08 | [08-wash-scanner.md](08-wash-scanner.md) | wash monitor, libwps IE parsing, JSON, vendor detection |
| 09 | [09-state-and-session.md](09-state-and-session.md) | `globule` global state, `.wpc` session format |
| 10 | [10-wireless-interface.md](10-wireless-interface.md) | Interface/MAC, channel hopping (wext/libnl3/apple) |
| 11 | [11-vendored-code.md](11-vendored-code.md) | Vendored trees (wpa_supplicant, Wireless Tools) and patches |
| 12 | [12-reference.md](12-reference.md) | `defs.h` enums/structs reference + glossary |

---

## Source tree at a glance

```
reaver-wps-fork-t6x/
├── AGENTS.md            # contributor / build cheat-sheet
├── README.md           # user-facing overview
├── docs/               # user docs (READMEs, man page) + dev/ (this folder)
├── tools/
│   └── logfilter.py    # pipes wpa_supplicant/reaver debug logs into pixiewps
└── src/
    ├── main.c          # argv[0] dispatcher
    ├── wpscrack.c      # reaver_main + usage
    ├── wpsmon.c        # wash_main + monitor loop
    ├── cracker.c       # crack(): main online brute-force loop
    ├── exchange.c      # do_wps_exchange(): EAP/WPS state machine (M1-M8)
    ├── send.c          # packet TX + receive-timer arming
    ├── builder.c       # 802.11 / LLC / 802.1X / EAP / WFA frame construction
    ├── 80211.c         # RX, auth/assoc, beacon parsing, FCS validation
    ├── pins.c          # PIN assembly from p1/p2 tables
    ├── keys.c          # static PIN tables k1[10000], k2[1000]
    ├── pixie.c         # Pixie Dust: spawn pixiewps with collected values
    ├── keys-derivation in wps/wps_common.c (vendored, patched)
    ├── session.c       # .wpc save/restore + crack progress + jump-queue
    ├── globule.c/.h    # global state struct + getters/setters
    ├── defs.h          # constants, 802.11/EAP/WPS enums and packed structs
    ├── iface.c         # MAC read + channel switching (wext/libnl3/apple)
    ├── init.c          # wps_data init + pcap capture init
    ├── argsparser.c    # reaver command-line parsing
    ├── misc.c crc.c pcapfile.c sigalrm.c sigint.c version.c  # support
    ├── libwps/         # WPS IE parser used by wash (authored)
    ├── lwe/            # Wireless Tools v29 (vendored)
    └── common/ crypto/ tls/ utils/ wps/   # wpa_supplicant code (vendored)
```

The distinction between **authored** code (top-level `src/*.c`, `libwps/`) and
**vendored** code (`common/ crypto/ tls/ utils/ wps/ lwe/`) is important: the
vendored trees should not be hand-edited except for the small Reaver-specific
patches documented in [11-vendored-code.md](11-vendored-code.md).
