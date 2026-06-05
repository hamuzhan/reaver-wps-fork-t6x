# Pixie Dust (Offline Attack)

The Pixie Dust attack (`-K` / `-Z`) recovers the WPS PIN **offline** by
exploiting weak randomness in the AP's E-S1/E-S2 nonces. Reaver does not crack
anything itself here: it collects the cryptographic values exchanged in a single
M1→M3 handshake and feeds them to the external **`pixiewps`** tool.

Relevant files: `pixie.c` (+ `pixie.h`) and the patched hooks in the vendored
`wps/` tree. Enabled via `argsparser.c:107`.

## 1. The collected values

`pixiewps` needs six values; an optional seventh (AP uptime) helps against some
RNG seeds:

| Value | What it is | Collected in | Stored as |
|-------|-----------|--------------|-----------|
| **PKE** | Enrollee (AP) DH public key | `wps_registrar.c:1923` (`wps_process_e` pubkey) | `pixie.pke` |
| **PKR** | Registrar (our) DH public key | `wps_attr_build.c:65` (build DH pubkey) | `pixie.pkr` |
| **E-Nonce** | Enrollee nonce | `wps_registrar.c:1699` (`wps_process_enrollee_nonce`) | `pixie.enonce` |
| **E-Hash1** | Enrollee hash of first half | `wps_registrar.c:1763` (`wps_process_e_hash1`) | `pixie.ehash1` |
| **E-Hash2** | Enrollee hash of second half | `wps_registrar.c:1783` (`wps_process_e_hash2`) | `pixie.ehash2` |
| **AuthKey** | Derived session auth key | `wps_common.c:133` (`wps_derive_keys`) | `pixie.authkey` |
| uptime | AP uptime from beacon timestamp | `cracker.c:65` `extract_uptime` | `globule->uptime` |

The `struct pixie` global (`pixie.h:9`) holds the hex-formatted strings:

```c
struct pixie {
    char *authkey, *pkr, *pke, *enonce, *ehash1, *ehash2;
    int do_pixie;     // set by -K/-Z
    int use_uptime;   // set by -u
};
extern struct pixie pixie;   // defined in pixie.c:24
```

Each hook is guarded by `if (pixie.do_pixie)` and uses the helpers
`pixie_format()` (binary → hex, `pixie.c:26`) and the `PIXIE_SET` macro
(`pixie.h:28`, strdup with free of the old value).

## 2. End-to-end flow

```mermaid
flowchart TD
    START([reaver -K -b BSSID]) --> SETUP["argsparser: pixie.do_pixie=1\nmax_pin_attempts=1"]
    SETUP --> LOOP["crack(): one attempt"]
    LOOP --> EXCH["do_wps_exchange()"]
    EXCH --> M1["receive M1\n→ store E-Nonce, PKE"]
    M1 --> M2["build/send M2\n→ store PKR, derive AuthKey"]
    M2 --> M3["receive M3\n→ store E-Hash1, E-Hash2"]
    M3 --> TRIGGER["wps_process_e_hash2()\ncalls pixie_attack()"]
    TRIGGER --> RUN["pixie_attack(): build pixiewps cmd\nrun in worker thread"]
    RUN --> RESULT{pixiewps prints\n'[+] WPS pin:'?}
    RESULT -->|yes| SETPIN["set_pin(pin);\nset wps->dev_password"]
    RESULT -->|no| NACK["send WSC_NACK; exit(1)"]
    SETPIN --> TO{timeout hit\nduring run?}
    TO -->|yes| SAVE["update_wpc_from_pin(); exit(0)\n(use -p to finish)"]
    TO -->|no| CONT["exchange continues with real PIN\n→ may recover WPA PSK directly"]
    CONT --> DONE([KEY_DONE / quit after pixie])
```

The trigger point is important: `pixie_attack()` is invoked from
`wps_process_e_hash2()` (`wps_registrar.c:1788`), i.e. **as soon as M3 is
parsed** — that is the first moment all six values exist.

## 3. `pixie_attack()`

Source: `pixie.c:116`.

1. If `do_pixie`, format the optional `-u <uptime>` argument.
2. Assemble the command into `ptd.cmd`:
   ```
   pixiewps [-u UPTIME] -e PKE -s EHASH1 -z EHASH2 -a AUTHKEY -n ENONCE {-S | -r PKR}
   ```
   With small DH keys (`-S`, `get_dh_small()`) it passes `-S` and omits PKR;
   otherwise it passes `-r PKR` (`pixie.c:125-129`).
3. Run it via `pixie_run_thread()` and inspect the result.
4. On success: `set_pin(pinbuf)` and copy the PIN into `wps->dev_password` so the
   ongoing exchange can continue toward the WPA PSK.
   - If the run only finished **after** the receive timeout fired, it instead
     saves progress via `update_wpc_from_pin()` and `exit(0)`, telling the user
     to re-run with `-p <PIN>` (`pixie.c:136-141`).
5. On failure: send a WSC_NACK and `exit(1)` (`pixie.c:147`).
6. Finally, free all collected values (`PIXIE_FREE`, `pixie.c:152`).

## 4. Why a worker thread? (`pixie_run_thread`)

Source: `pixie.c:89`. `pixiewps` can take longer than the WPS receive timeout.
If Reaver simply blocked on it, the AP could time out / lock the session. So:

- The actual `pixiewps` invocation runs on a pthread (`pixie_thread`,
  `pixie.c:83`), which is why the binary links `-lpthread`.
- The main thread polls `thread_done` every 2 ms. If `get_rx_timeout()` elapses
  first, it sends a **silent WSC_NACK** to keep the AP happy and records
  `timeout_hit` (`pixie.c:100-106`).
- `cprintf_mute()`/`cprintf_unmute()` (`misc.c:80`) silence Reaver's own logging
  while the child writes to stdout, avoiding interleaved output (`pixie.c:91`).

`pixie_run()` (`pixie.c:36`) is the blocking core: it `popen()`s the command,
echoes the child's output, and scans for the success marker
`"[+] WPS pin:"` (`PIXIE_SUCCESS`, `pixie.c:35`). It handles an `<empty>` PIN
(valid: PIN is the empty string) and copies the parsed PIN into the buffer.

## 5. Caveats encoded in the code

- **Realtek + small DH keys**: the README warns not to combine `-S` with Realtek
  APs; the code path simply passes `-S`/omits PKR, leaving the choice to the
  user.
- After a successful Pixie run, `crack()` calls `update_wpc_from_pin()` and
  breaks out of the loop (`cracker.c:330`), so Reaver quits after the Pixie
  attack rather than continuing the online brute force.
- `update_wpc_from_pin()` (`cracker.c:39`) turns off pixie mode, re-parses the
  recovered PIN, and reorganizes the `p1`/`p2` arrays via the jump-queue so a
  saved `.wpc` reflects the found PIN.

## 6. `tools/logfilter.py` — the manual alternative

`tools/logfilter.py` (Python 3, tab-indented) is a standalone way to run Pixie
Dust from logs instead of the built-in path. It reads wpa_supplicant/reaver
**debug** output on stdin and extracts the same values by matching hexdump
lines:

| Log line contains | Captured field | Expected length |
|-------------------|----------------|-----------------|
| `Enrollee Nonce` | `e_nonce` | 16 bytes |
| `DH own Public Key` | `pkr` | 192 bytes |
| `DH peer Public Key` | `pke` | 192 bytes |
| `AuthKey` | `authkey` | 32 bytes |
| `E-Hash1` / `E-Hash2` | `e_hash1` / `e_hash2` | 32 bytes |
| `Network Key` | `wpa_psk` | (the recovered PSK) |
| `E-SNonce1` / `E-SNonce2` | `e_snonce1/2` | 16 bytes |

When all six Pixie inputs are present (`got_all_pixie_data`, `logfilter.py:67`)
it `exec`s:

```
pixiewps --pke ... --pkr ... --e-hash1 ... --e-hash2 ... --authkey ... --e-nonce ...
```

This requires running reaver with debug logging (`-vvv`, which sets
`wpa_debug_level = MSG_DEBUG` via `set_debug()` `globule.c:307`) and piping the
output through the filter.

## 7. Relevant flags

| Flag | Effect | Source |
|------|--------|--------|
| `-K`, `--pixie-dust` / `-Z` | Enable Pixie Dust; set `max_pin_attempts=1`. | `argsparser.c:107` |
| `-S`, `--dh-small` | Use small DH keys (passes `-S` to pixiewps). | `argsparser.c:179`, `pixie.c:129` |
| `-u` | Pass AP uptime (`-u`) to pixiewps. | `argsparser.c:112` |

> The cryptographic background (how PKE/PKR/AuthKey/E-Hash are derived) is in
> [07-crypto-keys.md](07-crypto-keys.md).
