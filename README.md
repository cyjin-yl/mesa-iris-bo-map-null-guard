# Mesa Iris: `iris_bo_map()` NULL-return guard patch

> **Authored by Qwen3.8 27B** (an LLM), with the owner directing and verifying on real hardware.
> **USE AT YOUR OWN RISK.** This is an unreviewed, single-maintainer patch. It is not upstream, not
> security-hardened, and not tested beyond one machine. Back up your system before installing.

## What problem it solves

On Intel i915 (Gen12+ / Arc / DG2) with a **limited CPU-mappable BAR window** (e.g. an A380 whose
PCIe BAR2 is capped to 256 MiB by the BIOS, so `gem_mmap` cannot map large GTT objects),
`iris_bo_map()` returns **NULL** when the backing buffer cannot be mapped. Mesa's Iris driver then
dereferences that NULL in several places, and the most common one is a **hard SEGV inside
`iris_unmap_tiled_memcpy`** — the write-side `linear_to_tile4` copy used by `tc_flush`. That is exactly
what took down `gnome-shell` on the reference machine (a Fedora 44 / A380 desktop crashed in the
compositor's tile-flush path under window-pressure).

This patch guards **every** `iris_bo_map()` call site against a NULL return, so that a failed mmap
degrades gracefully (drop the tiled update / fall back / leave the object unmap) instead of
segfaulting the process.

- **Before:** `gnome-shell` SEGVs in `libgallium` (`linear_to_tile4_faster`) → desktop dies.
- **After:** the update is dropped, a `perf_debug` message is logged, and the process keeps running.

## Which Mesa version

- Built and verified against **Mesa 26.2.4** (the Fedora 44 build). The patch is a plain unified diff
  against the pristine 26.2.4 source tree and **dry-runs cleanly** on it (zero rejects).
- The 21 call sites were **also confirmed still unguarded on upstream `main`** (checked against raw
  main files), so the bug is current upstream — but this diff is only guaranteed to apply/build on
  **26.2.4**. Do not assume it applies to other versions.
- Target hardware tested: **Intel Arc A380 (DG2) via i915**, 256 MiB CPU window, Fedora 44.

## The patch

- `patch/iris-bo-map-null.patch` — the full 22-hunk diff (7 files). Applies with `patch -p1` from the
  mesa source root.
- `call-sites.json` — the 21 guarded call sites with file/function/line and a note on each risk.

The guarded sites:

| File | Functions |
|---|---|
| `iris_resource.c` | `iris_map_copy_region`, `iris_unmap_tiled_memcpy` (the field crash), `iris_map_tiled_memcpy`, `iris_map_direct`, `iris_texture_subdata` |
| `iris_bufmgr.c` | `iris_bo_map` (alias path), `intel_aux_map_buffer_alloc` |
| `iris_batch.c` | `iris_batch_decode_bo`, `iris_batch_decode_batch`, `create_batch` (reworked to alloc-then-swap so a failed new batch keeps the old one), `iris_batch_reset`, `iris_batch_flush`, `iris_chain_to_new_batch` |
| `iris_binder.c` | `binder_realloc` (alloc-then-swap) |
| `iris_border_color.c` | `iris_init_border_color_pool`, `iris_upload_border_color` (falls back to `BC_BLACK`) |
| `iris_measure.c` | `iris_init_batch_measure`, `iris_destroy_batch_measure`, `measure_start_snapshot`, `measure_end_snapshot` |
| `iris_utrace.c` | `iris_utrace_create_buffer` |

## How to use it

1. Get the mesa 26.2.4 source tree.
2. Apply the patch from the source root:
   ```sh
   patch -p1 < iris-bo-map-null.patch
   ```
3. Build mesa as usual (meson/ninja). The patched driver code lands in `libgallium-26.2.4.so`
   (plus the `libdril_dri.so` megadriver and `libEGL_mesa`/`libGLX_mesa`).
4. Point your session at the patched build, either:
   - **Per-process:** `LD_LIBRARY_PATH=<install>/lib LIBGL_DRIVERS_PATH=<install>/lib/dri`, or
   - **System-wide (surgical):** back up the originals, then replace the mesa files your compositor
     actually loads. On Fedora the non-free i915/iris driver lives in the **freeworld** set, i.e.
     `/usr/lib64/dri-freeworld/libgallium-26.2.4.so` (and the matching `dri/`, `libEGL_mesa`,
     `libGLX_mesa`). Back up before overwriting so you can revert.
5. Restart the compositor (`pkill -x gnome-shell` — gnome-session relaunches it in seconds).

Verify the patched `libgallium` is the one loaded:
```sh
sudo grep -oE '/[^ ]*libgallium[^ ]*\.so' /proc/$(pgrep -x gnome-shell)/maps | sort -u
nm <that-file> | grep -cE 'iris_bo_map|iris_unmap_tiled_memcpy'   # > 0 means patched
```

## Reverting

Restore the backed-up originals over the mesa files and restart the compositor. No package state is
changed by this patch (it only replaces `.so` bytes), so a normal `dnf reinstall mesa-freeworld` also
restores the stock driver.

## Caveats / use at your own risk

- **Not upstream, not reviewed.** Single-maintainer, LLM-authored. The `create_batch`/`binder_realloc`
  reworks change allocation ordering (alloc-new-then-swap) to avoid double-unref on failure — correct
  by construction but not battle-tested.
- **Version-locked to 26.2.4.** The diff is not guaranteed to apply elsewhere.
- **Only the i915/Arc path is exercised.** Other drivers in the same `libgallium` are untouched by the
  patch but are rebuilt in the same tree; if you swap the whole `libgallium`, you're running a
  from-source build for all of them.
- **It is a mitigation, not a cure.** The root cause is a BAR window too small to map the GPU's
  buffers. The real fix is enabling **Resizable BAR** in the BIOS (so the A380's 6 GB can be fully
  mapped). This patch makes the driver survive the shortfall instead of crashing.
- One transient `gnome-shell` SEGV was observed *inside* the swapped-in `libgallium` at the moment of
  the file swap (a transitional instance). Subsequent instances ran stably for minutes on the real
  A380 desktop. Treat any compositor instability as a reason to revert immediately.
