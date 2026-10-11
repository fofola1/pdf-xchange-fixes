# Fixing the PDF-XChange Editor crash (Wine rpcrt4 deadlock)

**Short version:** PDF-XChange Editor crashes after ~40 minutes of use because of a
Wine bug in `rpcrt4` (the named-pipe RPC server overflows the 64-handle wait limit
and the RPC server thread shuts down, deadlocking COM marshaling). Two things fix it:

1. A launcher fix packaged in this repo's `PKGBUILD` (quiet, sane launcher).
2. A patched Wine that works around the actual Wine bug (`wine-rpcrt4-np.patch`).

You must rebuild **both** to get a fully stable installation. The PKGBUILD change
alone improves things but does **not** prevent the crash.

---

## TL;DR

```bash
# 1) rebuild pdf-xchange with the fixed launcher (from this repo)
cd /home/fofola/Desktop/pdf-xchange
makepkg -si          # after you have reviewed PKGBUILD and .SRCINFO

# 2) rebuild Wine with the rpcrt4 workaround (see "Building patched Wine")

# 3) verify
WINEDEBUG=fixme-all pdf-xchange --debug-dump
```

---

## What the log showed (evidence)

The captured log (`wine_debug.log`, 216,530 lines) shows the crash sequence:

```
0120:err:rpc:rpcrt4_protseq_np_wait_for_new_connection wait failed with error 87
0454:err:rpc:RpcAssoc_BindConnection receive failed with error 1726
0024:err:sync:RtlpWaitForCriticalSection ... rpc_server.c: server_cs ... blocked by 012c, retrying (60 sec)
0024:err:sync:RtlpWaitForCriticalSection ... combase/rpc.c: dcom registered server interfaces ... blocked by 012c ...
0024:err:seh:call_seh_handlers invalid frame ...
0024:err:seh:NtRaiseException Exception frame is not in stack limits => unable to dispatch exception.
cp: cannot stat '/tmp/pdf-xchange/*': No such file or directory
```

Root cause: Wine's `rpcrt4` named-pipe server exceeds ntdll's **64-handle
`WaitForMultipleObjectsEx` limit** (`error 87` = `STATUS_INVALID_PARAMETER`; Wine
bug 60334 logs the same as "wait handle count 65"). The wait failure makes
`RPCRT4_server_thread()` terminate, which leaves a thread holding `server_cs` /
`dcom registered server interfaces` forever. Every COM/RPC call from then on
deadlocks, and the main thread finally dies → "unable to dispatch exception".

The ~193k `bidi:GetCharacterPlacementW` and ~14.8k `NtQueryInformationProcess`
fixme lines are normal noise, **not** the trigger.

Upstream status: WineHQ bug [55705](https://bugs.winehq.org/show_bug.cgi?id=55705)
("PDF Xchange editor crashes") — UNCONFIRMED, last change 2026-09-16; linked to
[60334](https://bugs.winehq.org/show_bug.cgi?id=60334) (same failure, Steam, NEW).
No released Wine contains a fix. Tracker Software does not support Wine. This
patch is a local workaround.

---

## The Wine patch (`wine-rpcrt4-np.patch`)

The patch changes `rpcrt4_protseq_np_wait_for_new_connection()` in
`dlls/rpcrt4/rpc_transport.c` so that when more than 64 listener handles are
registered on one protocol sequence, Wine waits on them **in chunks of ≤ 64** and
cycles through all of them, instead of letting the wait overflow and killing the
RPC server thread.

### Before you start

Determine your Wine version — the patch was written against Wine master, and the
surrounding comment block in the target function is identical in Wine 8/9/10/11,
so it normally applies with fuzz. Still, check:

```bash
wine --version
```

If the hunk does not apply cleanly (or you want to be sure), the change is fully
contained: open `dlls/rpcrt4/rpc_transport.c`, find
`rpcrt4_protseq_np_wait_for_new_connection`, and replace the single line

```c
        res = WaitForMultipleObjectsEx(count, objs, FALSE, INFINITE, TRUE);
```

inside the `do { ... } while (res == WAIT_IO_COMPLETION);` block with the chunked
wait shown in the patch. That is the entire change.

### Building patched Wine (Arch)

Wine on Arch is compiled from source by the `wine` package in `extra`. The
cleanest route:

```bash
# get the PKGBUILD tree for the wine package
asp export wine
cd wine

# copy the workaround patch in (adjust the path)
cp /home/fofola/Desktop/pdf-xchange/wine-rpcrt4-np.patch .

# apply it
patch -p1 < wine-rpcrt4-np.patch      # expect: "Hunk #1 succeeded ... (with fuzz ...)"

# build and install (needs base-devel; this takes a long time)
makepkg -si

# confirm the new wine is installed and the version matches what you had
wine --version
```

For **wine-staging** users, the same file exists in the staging tree:

```bash
asp export wine-staging
cd wine-staging
cp /home/fofola/Desktop/pdf-xchange/wine-rpcrt4-np.patch .
patch -p1 < wine-rpcrt4-np.patch
makepkg -si
```

Notes:

- Keep the same Wine major version you already had (mix Wine versions across
  prefixes is not recommended).
- If the build fails with a checksum/`source=` mismatch, just re-run `asp export
  wine` to refresh the PKGBUILD, then re-apply the patch.
- Downgrading after `makepkg -si` is possible with
  `sudo pacman -U /var/cache/pacman/pkg/wine-<old>.pkg.tar.zst` if anything goes
  wrong. The patch only relaxes one wait; it does not change RPC behavior when
  fewer than 64 listeners exist (the overwhelmingly common case).

---

## The PKGBUILD launcher fixes (this repo)

The updated `PKGBUILD` (pkgrel 2) fixes three launcher problems that made the
crash worse and pollute output:

| Problem | Fix |
|---|---|
| `cp "$prefix"/Tesseract/* /tmp/$pkgname` with unquoted glob → `cp: cannot stat '/tmp/pdf-xchange/*'` on empty dir | `cp -a "$prefix/Tesseract"/. "/tmp/$pkgname"/ 2>/dev/null` |
| OCR data copied back only on a clean exit line appended after the wine command → lost when Wine crashes | `trap 'cp -a "/tmp/$pkgname"/. "$prefix/Tesseract"/ 2>/dev/null; rm -rf "/tmp/$pkgname"' EXIT` |
| Unbounded `while ...; do echo Waiting...; done` busy-loop after `wineboot` | bounded wait (30 × 1s), then continue |
| Wine dumps thousands of debug lines on every run | `WINEDEBUG="${WINEDEBUG:--all}"` (default quiet, overridable) |

The `/tmp/$pkgname` location is required by the `Tesseract` symlink the package
installs; running Tesseract from the live prefix would risk corrupting
`system.reg`/`user.reg` writes — so the copy-in/copy-out dance stays, but the
trailing unstoppable `cp` line is gone.

### Rebuild the package

```bash
cd /home/fofola/Desktop/pdf-xchange
makepkg -si          # installs pdf-xchange with the fixed launcher
# or just:
makepkg -f           # build the package without installing
```

---

## Verification

1. **Launcher** — start the app and then exit immediately:

   ```bash
   pdf-xchange
   # ... exit the app ...
   # no "cp: cannot stat '/tmp/pdf-xchange/*'" must appear
   ls "/tmp/pdf-xchange" 2>/dev/null || echo "tmp dir cleaned up"   # should be gone
   ls "$HOME/.local/share/pdf-xchange/PDF Editor/wine/Tesseract" | head   # OCR data intact
   ```

2. **Patched Wine** — with the fix, no `rpc:`/`RtlpWaitForCriticalSection` errors
   over a long session:

   ```bash
   WINEDEBUG=err+all,fixme-all pdf-xchange 2>&1 | grep -E 'rpc:|RtlpWaitForCriticalSection' | head
   ```

   Run steadily for several hours (open/close PDFs, annotations, OCR) — the
   previous failure routinely appeared by ~40 minutes. Any output from the grep
   means the bug is still present.

3. **Regression check** — confirm the app still starts with a file argument:

   ```bash
   pdf-xchange /path/to/sample.pdf
   ```

---

## Optional extra: reduce trigger frequency

Disabling the plugin suites that open REST/cloud channels (Tracker: "AzureRMS /
MIP, MSAL and other cloud plugins") lowers how often the RPC listener count
grows, per the vendor. This **reduces** frequency but does **not** prevent the
crash — the Wine patch is the real fix.

```bash
# inside PDF-XChange Editor: Preferences (Ctrl+K) -> Plugins, disable:
#   Azure RMS / MIP protected documents
#   MSAL / Enterprise (Microsoft Information Protection)
# or rename the plugins out of the way before starting Windows (find in:
#   /usr/lib/pdf-xchange/PDF Editor/Plugins.x64/ ... )
```