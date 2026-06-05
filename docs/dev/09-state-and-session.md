# Global State & Sessions

Reaver keeps almost all mutable state in one global structure and persists the
attack progress to `.wpc` session files so an interrupted run can resume.

Relevant files: `globule.c` / `globule.h` (state), `session.c` (persistence).

## 1. The `globule` global

A single heap-allocated `struct globals *globule` (`globule.h:39`) holds shared
state. It is allocated and zeroed in `globule_init()` (`globule.c:38`), which
also sets two non-zero defaults:

```c
globule->resend_timeout_usec = 200000;  // 200 ms packet-resend tick
globule->output_fd = -1;                // no -O output by default
```

`globule_deinit()` (`globule.c:54`) frees everything: the `p1`/`p2` strings, the
`wps_data`, the pcap handle, and all owned strings/fds.

### Access pattern: getters/setters only

Modules never read or write `globule->field` directly (with rare exceptions like
`session.c` reordering the `p1`/`p2` arrays). Instead there are ~70 typed
accessors in `globule.c`. Conventions:

- Scalars: `set_x(v)` / `get_x()`.
- Owned strings (`set_ssid`, `set_iface`, `set_session`, `set_pin`,
  `set_static_p1/p2`, `set_exec_string`): free the old value and `strdup` the new
  one; passing `NULL` clears.
- Fixed buffers (`set_bssid`, `set_mac`): `memcpy` 6 bytes.
- Length-carrying buffers (`set_ap_rates`, `set_ap_ext_rates`, `set_ap_htcaps`):
  reallocate and store the length alongside.
- `set_output_fd()` (`globule.c:661`) additionally writes the pcap global header
  when a valid fd is set.
- `set_debug()` (`globule.c:307`) also raises `wpa_debug_level` to `MSG_DEBUG`
  when the level is `DEBUG`, enabling the vendored hexdumps used by
  `tools/logfilter.py`.

### Field groups

| Group | Fields (selected) |
|-------|-------------------|
| Target | `bssid`, `mac`, `ssid`, `iface`, `channel`, `ap_capability`, `vendor_oui`, `uptime` |
| AP IEs (replayed) | `htcaps`/`len`, `ap_rates`/`len`, `ap_ext_rates`/`len` |
| PIN keyspace | `p1[]`, `p2[]`, `p1_index`, `p2_index`, `static_p1`, `static_p2`, `use_pin_string`, `key_status` |
| Timing | `delay`, `fail_delay`, `recurring_delay(_count)`, `lock_delay`, `rx_timeout`, `m57_timeout`, `resend_timeout_usec`, `timer_id`, `out_of_time` |
| Behavior flags | `dh_small`, `external_association`, `oo_send_nack`, `win7_compat`, `ignore_locks`, `eap_terminate`, `timeout_is_nack`, `repeat_m6`, `mac_changer`, `validate_fcs`, `fixed_channel`, `auto_channel_select`, `wifi_band` |
| Protocol | `last_wps_state`, `opcode`, `eap_id`, `eapol_start_count`, `nack_reason`, `wps` (the `wps_data`) |
| I/O | `handle` (pcap), `output_fd`, `fp` (log file), `session`, `exec_string`, `pin`, `debug`, `max_pin_attempts`, `max_num_probes` |

The full annotated list is in `globule.h:39-163`; an enum reference is in
[12-reference.md](12-reference.md).

> The Pixie Dust values are **not** in `globule`; they live in the separate
> `struct pixie pixie` global ([06-pixie-dust.md](06-pixie-dust.md)).

## 2. Session files (`.wpc`)

A session captures exactly enough to resume the keyspace walk: the two indices,
the key status, and the (possibly reordered) `p1`/`p2` arrays.

### File location and name

`gen_sessionfile_name()` (`session.c:44`):
- Default: `<CONF_DIR>/<bssid>.wpc` if `CONF_DIR` exists, else `<bssid>.wpc` in
  the current directory.
- With `--enable-savetocurrent` (`-DSAVETOCURRENT`): always
  `<bssid>.wpc` in the current directory (`session.c:45`).
- `-s <file>` overrides the path entirely (`get_session()`).

`CONF_DIR` is the compile-time `localstatedir/lib/<target>` from configure
([02-build-system.md](02-build-system.md)). The BSSID is rendered without
delimiters (`mac2str(..., '\0')`).

### File format

Plain text, one value per line (`save_session()` `session.c:191`):

```
<p1_index>\n
<p2_index>\n
<key_status>\n
<p1[0]>\n <p1[1]>\n ... <p1[P1_SIZE-1]>\n     (10000 lines)
<p2[0]>\n <p2[1]>\n ... <p2[P2_SIZE-1]>\n     (1000 lines)
```

So a `.wpc` file is the 3 header lines plus 11000 candidate lines, preserving the
exact attack order (including any jump-queue reordering).

### Save

`save_session()` (`session.c:191`):
- Skips saving in string-pin mode (`session.c:193`).
- Skips if nothing has been tried yet (`session.c:200`).
- Writes the indices, status, and both arrays.

It is called periodically from `crack()` (every `DISPLAY_PIN_COUNT` loops,
`cracker.c:303`), at the end of `reaver_main` (`wpscrack.c:134`), and from the
SIGINT handler (`sigint.c:66`).

### Restore

`restore_session()` (`session.c:54`):
1. Determine the file name (explicit `-s` or default for the BSSID).
2. Decide whether to restore:
   - With `-s`, answer is auto-yes.
   - In string-pin mode, auto-no.
   - Otherwise prompt `Restore previous session for <bssid>? [n/Y]` on stderr
     (so it is visible even when stdout is redirected) (`session.c:105`).
3. Read the indices, status, and arrays back into `globule`.
4. On failure or decline, reset to a fresh state (`index=0`, `KEY1_WIP`).
5. If a static PIN was given, jump-queue it (below); returns `-1` if that PIN was
   already tested (so `crack()` aborts), unless the previous session already
   reached `KEY_DONE`.

## 3. Crack progress (for `wash -p`)

`get_crack_progress()` (`session.c:254`) reads only the 3 header lines of a
BSSID's `.wpc` and computes a percentage string:

| key_status | attempts | formula |
|------------|----------|---------|
| `KEY1_WIP` | `p1_idx` | `attempts*100 / (P1_SIZE+P2_SIZE)` |
| `KEY2_WIP` | `P1_SIZE + p2_idx` | same denominator |
| `KEY_DONE` | — | `100.0` |

This is what wash's `-p` column displays
([08-wash-scanner.md](08-wash-scanner.md)).

## 4. The jump-queue

When a specific PIN half is supplied, the candidate arrays are reordered so that
value is tried next rather than from the front:

- `jump_p1_queue(value)` (`session.c:314`): if `value` is at or after the current
  `p1_index`, rotate it to `p1_index`. Return values:
  - `0` value equals current index (no-op)
  - `1` array reorganized
  - `-1` value already tried (before current index)
  - `-2` already past first half (`KEY2_WIP`/`KEY_DONE`)
- `jump_p2_queue(value)` (`session.c:377`): the analogous logic for the second
  half.

These directly manipulate `globule->p1[]` / `globule->p2[]` (one of the few
direct-access exceptions) and are used by `restore_session()` and by
`update_wpc_from_pin()` after a Pixie/`-p` success (`cracker.c:39`).

## 5. Lifecycle summary

```
reaver_main
  globule_init()          # allocate + defaults
  process_arguments()     # populate target/flags
  crack()
    generate_pins()       # fill p1/p2
    restore_session()     # maybe resume
    ... loop ...          # mutate indices/key_status, periodic save_session()
  save_session()          # final
  globule_deinit()        # free everything
```

Ctrl-C at any point: `sigint_handler` terminates the WPS session, calls
`save_session()`, `globule_deinit()`, and exits (`sigint.c:50`).
