# Build System

Reaver uses a **hand-written `Makefile`** combined with an **autoconf-generated
`configure`** script. All build commands run from `src/`, not the repository
root.

```sh
cd src
./configure        # optional flags below
make               # builds ./reaver; ./wash is a symlink to it
sudo make install  # installs reaver + wash to $prefix/bin
make clean
```

> There is **no `make distclean`**. The real Makefile only defines `all`,
> `clean`, and `install` (plus the dev-only `extest`). Any upstream prose that
> mentions other targets is stale.

## 1. Pieces and their roles

| File | Generated? | Edited by hand? | Purpose |
|------|-----------|-----------------|---------|
| `Makefile` | No | **Yes** | The real build recipe. Edit this directly. |
| `configure.ac` | No | Yes | Autoconf source. After editing, regenerate `configure`. |
| `configure` | Yes (autoconf) | No (committed) | Detects compiler/libs, writes `config.mak` + `config.h`. |
| `config.mak.in` | No | Yes | Template for `config.mak` (`@var@` substitutions). |
| `config.mak` | Yes (`./configure`) | No | Included by `Makefile`; sets `CC`, flags, paths. |
| `VERSION` | No | Yes | Single source of the version string (`1.6.6`). |
| `version.sh` | No | Yes | Prints version from git tags or `VERSION`. |
| `install.sh` | No | Yes | Atomic install helper used by `make install`. |
| `m4/` | — | — | Autoconf macros (e.g. `AS_COMPILER_FLAG`). |

## 2. `configure.ac` — what it checks and offers

Source: `configure.ac` (54 lines).

- Derives the package version from `VERSION` via
  `esyscmd(cat VERSION | tr -d '\n')` (`configure.ac:1`).
- Base flags: `CFLAGS="-Wall ..."`, `LDFLAGS="-lm -lpcap ..."`
  (`configure.ac:8`).
- **Hard requirements** (configure fails if missing):
  - `libpcap` providing `pcap_open_live` (`configure.ac:11`).
  - Headers `stdlib.h stdint.h string.h` and `pcap.h`
    (`configure.ac:12`).
- **Optional `--enable-*` switches:**

| Flag | Effect | Source |
|------|--------|--------|
| `--enable-savetocurrent` | Adds `-DSAVETOCURRENT`: session `.wpc` files are written to the current directory instead of the configured state dir. | `configure.ac:15` |
| `--enable-libnl3` | Adds `-DLIBNL3` and pkg-config flags for `libnl-3.0`/`libnl-genl-3.0`; channel switching uses nl80211 instead of wext. | `configure.ac:20` |
| `--enable-libnl-tiny` | Same idea using `libnl-tiny` (still defines `-DLIBNL3`). | `configure.ac:31` |

> Why the libnl options matter: the default code switches Wi-Fi channels using
> **Linux Wireless Extensions (wext)**. Many modern kernels lack wext support,
> in which case the channel options silently no-op. Building against libnl makes
> channel switching work via nl80211. See
> [10-wireless-interface.md](10-wireless-interface.md).

- `AS_COMPILER_FLAG` probes optional warning flags (`configure.ac:46`).
- `cp confdefs.h config.h` produces the generated `config.h`
  (`configure.ac:51`).
- Outputs `config.mak` from `config.mak.in` (`AC_CONFIG_FILES`,
  `configure.ac:4`).

After editing `configure.ac`, regenerate and commit `configure`
(`autoreconf -i` / `autoconf`), matching the existing git history.

## 3. `config.mak.in` → `config.mak`

`config.mak.in` (9 lines) is filled in by `configure`:

```make
prefix=@prefix@
exec_prefix=@exec_prefix@
CONFDIR=@localstatedir@/lib/@target@   # default .wpc session directory
CC=@CC@
CFLAGS_USER=@CFLAGS@
LDFLAGS=@LDFLAGS@
LIBNL_CFLAGS=@LIBNL_CFLAGS@
LIBNL_LDFLAGS=@LIBNL_LDFLAGS@
```

`CONFDIR` becomes the compile-time `CONF_DIR` macro (`Makefile:15`) and is where
session files live by default; if it does not exist Reaver falls back to `.`
(see [09-state-and-session.md](09-state-and-session.md)).

## 4. `Makefile` internals

Source: `Makefile` (152 lines).

### Include + flags
- `-include config.mak` pulls in the configure results (`Makefile:5`).
- `INC=-Ilibwps -I. -Ilwe` (`Makefile:8`).
- `CFLAGS` adds `-DCONF_DIR=...`, user flags, disabled warnings
  (`-Wno-unused-variable`, `-Wno-unused-function`, `-Wno-pointer-sign`),
  libnl flags, and `-DCONFIG_IPV6` (`Makefile:10-17`).

### Object groups
| Variable | Contents | Source |
|----------|----------|--------|
| `UTILS_OBJS` | base64, common, ip_addr, radiotap, trace, uuid, wpa_debug, wpabuf, os_unix, vendor, eloop | `Makefile:19` |
| `WPS_OBJS` | the wpa_supplicant WPS state machine objects | `Makefile:33` |
| `LWE_OBJS` | `lwe/iwlib.o` (Wireless Tools) | `Makefile:37` |
| `TLS_OBJS` | asn1, bignum, pkcs*, rsa, tlsv1_*, x509v3 | `Makefile:39` |
| `CRYPTO_OBJS` | AES variants, DES, DH groups, MD4/MD5, SHA1/SHA256, RC4, internal crypto/RSA/PRF | `Makefile:57` |
| `LIB_OBJS` | `libwps/libwps.o` + WPS + UTILS + TLS + CRYPTO + LWE | `Makefile:92` |
| `MAIN_OBJS` | globule, init, sigint, iface, sigalrm, misc, session, send, pins, 80211, builder, keys, crc, pixie, version, pcapfile | `Makefile:96` |
| `PROG_OBJS` | `MAIN_OBJS` + exchange, argsparser, wpscrack, wpsmon, cracker, main | `Makefile:100` |

### Per-group extra flags
- WPS objects: `-I. -Iutils` (`Makefile:113`).
- TLS objects: add `-DCONFIG_INTERNAL_LIBTOMMATH -DCONFIG_CRYPTO_INTERNAL`
  (`Makefile:114`).
- CRYPTO objects: add `-DCONFIG_TLS_INTERNAL_CLIENT/SERVER -fno-strict-aliasing`
  (`Makefile:115`).

These macros select the **internal** (self-contained) crypto/TLS backends so the
build does not require OpenSSL/GnuTLS.

### Targets
| Target | What it does | Source |
|--------|--------------|--------|
| `all` | Builds `wash` and `reaver` (default). | `Makefile:111` |
| `reaver` | Links `PROG_OBJS + LIB_OBJS` with `-lpthread -lrt` (+ libnl libs). | `Makefile:122` |
| `wash` | `ln -sf ./reaver wash` (symlink, not a separate binary). | `Makefile:119` |
| `extest` | Dev-only: compiles `exchange.c` with `-DEX_TEST` for manual exchange testing. | `Makefile:125` |
| `install` | Atomically installs `reaver` + `wash` to `$exec_prefix/bin`; creates `CONFDIR`. | `Makefile:139` |
| `clean` | Removes binaries, all objects, and generated headers. | `Makefile:146` |

### Linker line
```make
reaver: $(PROG_OBJS) $(LIB_OBJS)
    $(CC) $(CFLAGS) $(INC) $(PROG_OBJS) $(LIB_OBJS) $(LDFLAGS) $(LIBNL_LDFLAGS) -lpthread -lrt -o reaver
```
(`Makefile:122`). `-lpthread` is needed for the Pixie Dust worker thread
([06-pixie-dust.md](06-pixie-dust.md)); `-lrt` for POSIX `timer_settime`
([03-reaver-attack-flow.md](03-reaver-attack-flow.md)).

## 5. Generated files — never edit by hand

All of these are produced by the build and are git-ignored
(`.gitignore:21-27`):

| File | Produced by | Recipe |
|------|-------------|--------|
| `version.h` | `version.sh` (git/VERSION) | `Makefile:130-132` |
| `lwe/wireless.h` | copied from `lwe/wireless.<WE_VERSION>.h` | `Makefile:134` |
| `config.h` | `configure` (`cp confdefs.h config.h`) | `configure.ac:51` |
| `config.mak` | `configure` from `config.mak.in` | `configure.ac:4` |

`version.h` definition:
```make
version.h: $(wildcard $(srcdir)/VERSION $(srcdir)/../.git)
    printf '#define R_VERSION "%s"\n' "$$(cd $(srcdir); sh version.sh)" > $@
```
`version.sh` prefers `git describe --tags 'v[0-9]*'` (turning `v1.6.6-3-gabcdef`
into `1.6.6-git-3-gabcdef`), and falls back to the `VERSION` file when not in a
git tree (`version.sh:6-15`). `version.c:3` simply returns `R_VERSION`.

The Wireless-Extensions header is selected from `lwe/iwlib.h`'s `WE_VERSION`
(`Makefile:103-107`) so the local copy always matches.

## 6. `install.sh`

A small POSIX `sh` script (`install.sh`) implementing an **atomic** install: it
writes to `dest.tmp.$$` then `mv -f` into place, so a running binary is never
truncated mid-copy. Flags: `-D` (mkdir parents), `-l` (symlink), `-m mode`.
`make install` calls it as `INSTALL` (`Makefile:2`, `Makefile:140`).

## 7. Build prerequisites recap

- `libpcap-dev` (required), `build-essential`.
- Optional: `libnl-3-dev libnl-genl-3-dev` **or** `libnl-tiny` for nl80211
  channel switching.
- Linux toolchain: relies on `timer_settime` (`-lrt`), pthreads, wext/ioctl.
  Does not build natively on macOS — use a Linux box/VM/container.
