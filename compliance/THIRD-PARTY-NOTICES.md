# Third-party notices — OpenDesign Linux unofficical build (v0.27.2)

This artifact is a repackaging of the upstream Open Design payload plus a
bundled Node.js runtime, an Electron/Chromium runtime, and the npm dependency
tree the upstream build produces. Most license texts travel inside the payload
itself (around 690 `LICENSE*` / `COPYING*` files under
`resources/app/node_modules`). This file summarizes the components whose
licensing deserves explicit attention when you redistribute the AppImage.

Generated 2026-11-26. Verify against the artifact you actually ship.

---

## 1. Apache-2.0 obligations this build satisfies (§4)

| §  | Obligation | How it is satisfied |
|---|------------|---------------------|
| 4(a) | Give recipients a copy of the License | `LICENSE` (Apache-2.0 full text) is at the AppDir root and also at `usr/share/doc/opendesign/LICENSE-APACHE-2.0.txt`; every AppImage unpacks the AppDir, so the text travels with the binary. |
| 4(b) | Modified files carry prominent "I changed this" notices | Both patched files open with a `MODIFIED FILE — OpenDesign Linux unofficial build` block comment, and each individual change site is additionally marked `[MODIFIED — OpenDesign Linux build]`. See `patches/`. |
| 4(c) | Retain copyright / attribution notices | Upstream's per-package notices are untouched; the payload ships them verbatim. |
| 4(d) | NOTICE file, if upstream ships one | Upstream `open-design-v0.27.2` has **no** `NOTICE` file (verified: no `NOTICE*` in the repo outside `node_modules`), so this clause does not apply. |
| 5   | Contributions under same terms unless stated otherwise | N/A — this is a distribution, not an upstream contribution. |
| 6   | **No trademark grant** | The artifact is named `OpenDesign-Linux-<ver>-clean`, the desktop entry reads `OpenDesign Linux (unofficial build)`, and the README states it is not affiliated with or endorsed by upstream. "Open Design" is used only to describe the origin of the work, as §6 permits. |

## 2. Stripped component — `vela`

| Field | Value |
|---|---|
| npm packages | `@powerformer/vela-cli@0.1.9`, `@powerformer/vela-cli-linux-x64@0.1.9` |
| Declared license | **`UNLICENSED`** (both packages; `publishConfig.access: public` notwithstanding) |
| Upstream role | `tools/pack/package.json` → `optionalDependencies`; the pack lane copies the binary into `resources/open-design/bin/vela` and the companion OpenCode tree into `resources/open-design/bin/libexec/opencode/` |
| Action taken | both paths deleted before repackaging |
| Effect | AMR / Vela agent runtime and Vela-backed Team collaboration are unavailable. Detection, launch and everything else still work. |

`UNLICENSED` in npm means no license is granted: not even the right to copy the
file, let alone redistribute it. It must never be published. If you need AMR,
obtain a written redistribution license from PowerFormer and only then put
`vela` back into `resources/open-design/bin/`.

## 3. Components that need a decision

### 3.1 ffmpeg — GPLv3 binary (**kept in this artifact; delete if you need GPL-free**)

| Field | Value |
|---|---|
| Package | `@ffmpeg-installer/ffmpeg@1.1.0` (+ platform package `@ffmpeg-installer/linux-x64`) |
| Declared licenses | `LGPL-2.1` (meta) / `GPLv3` (linux-x64 binary) |
| Actual binary | `ffmpeg version N-47683-g0e8eb07980-static`, built with `--enable-gpl --enable-version3` and statically linked against x264/x265/libtheora etc. |
| Source | `https://johnvansickle.com/ffmpeg/` (public static builds) |
| Location | `resources/app/node_modules/@ffmpeg-installer/linux-x64/ffmpeg` (~68 MB) |
| Used by | `apps/daemon/src/artifacts/motion-delivery.ts`, `chat-artifacts/cover.ts`, `motion-cover.ts`, `video-cover.ts` — poster-frame extraction and motion delivery |

GPLv3 §3 requires that you accompany the binary with, or provide written
offer for, the corresponding source. The upstream static build is publicly
published with its source at `johnvansickle.com/ffmpeg/`. This artifact keeps
the binary and points at that source.

If you need a **GPL-free** artifact, delete it:

```bash
rm -rf resources/app/node_modules/@ffmpeg-installer/linux-x64/ffmpeg
```

Consequences, checked against the source rather than assumed:

- video/audio **poster extraction** (`chat-artifacts/video-cover.ts`) catches the
  failure and returns `renderer_unavailable`, so cards fall back to live
  `<video>` previews — degraded, not broken;
- **Motion artifact export** does *not* degrade: `assertMotionExport()`
  (`artifacts/motion-delivery.ts`) rejects with "Motion MP4 has no decodable
  video frame", so motion exports fail outright. Decide accordingly.

### 3.2 gsap — GreenSock standard ("no charge") license

| Field | Value |
|---|---|
| Package | `gsap` (6.1 MB, `resources/app/node_modules/gsap`) |
| Declared license | `Standard 'no charge' license: https://gsap.com/standard-license.` — **not** OSI/FSF-approved |
| Text in artifact | none (no `LICENSE` file ships; only the URL in `package.json`) |
| Terms that matter | GreenSock permits free use, including in commercial end products, **but** prohibits distributing the library itself (or a competing product built on it) as a standalone offering, and forbids the "no charge" plugins from being used in products whose primary purpose is a competing animation tool. Shipping it as an embedded implementation detail of a design app is within its terms. |
| Action | keep, but do **not** publish `gsap` as its own downloadable asset; do not advertise it as a feature you are giving away |

Not published as a separate artifact here, so this is a documentation obligation
rather than an action item.

### 3.3 sharp / libvips — LGPL-3.0-or-later

`@img/sharp-libvips-linux-x64` ships libvips as a dynamically linked `.so`
inside the package. LGPL requires that the recipient be able to replace the
library. The `.so` files are ordinary files in the AppDir, so relinking/replacing
is possible; keep them non-stripped and non-obfuscated. A written statement that
the library may be replaced is enough — no source offer is required for
dynamically linked LGPL libraries.

### 3.4 jszip — dual `MIT OR GPL-3.0-or-later`

The MIT branch is elected here; `jszip/LICENSE.markdown` travels with the payload
and states the dual grant, so recipients can also elect MIT.

### 3.5 posthog-js — Apache-2.0

`posthog-js/LICENSE` ships in the payload. Analytics is inert unless a sink is
configured (`blockedReason: 'missing_sink'` in this build's logs).

## 4. Upstream third-party assets bundled inside the payload

These are *included by upstream* and are redistributed here verbatim. Each
retains its own license file inside the artifact.

| Asset | License | Notes |
|---|---|---|
| `design-templates/guizang-ppt/` | MIT (@op7418) | README of upstream names it explicitly |
| `design-templates/html-ppt/` | MIT (@lewislulu) | likewise |
| `skills/web-clone/` | MIT (@Jane-xiaoer) | likewise |
| 478 official plugins + skills + design systems + design templates under `resources/open-design/` | mixed MIT / Apache-2.0 / CC-BY / CC0 per item; many carry their own `LICENSE` | Inventory them individually before publishing a *modified* bundle. This build does not modify them. |
| `apps/web/public/fonts/AlbertSans-*.ttf` | SIL OFL 1.1 (copyright 2021 The Albert Sans Project Authors, github.com/usted/Albert-Sans) | embedded in web assets |
| `apps/web/public/fonts/remixicon.ttf` | Remix Icon license (Apache-2.0-compatible; upstream font project) | |
| Template-embedded `Inter-Variable.ttf`, `RobotoMono-*.ttf` in `plugins/_official/examples/*` | SIL OFL 1.1 | |

## 5. Obligations from the runtimes

| Component | License | Text in artifact |
|---|---|---|
| Electron | MIT | `LICENSE.electron.txt` at AppDir root |
| Chromium + all third-party open source it links | BSD-style + many | `LICENSES.chromium.html` at AppDir root (19 MB) |
| Node.js v24.14.1 (`resources/open-design/bin/node`) | MIT | ships its own bundled license texts inside the binary's distribution dirs; verify before republishing |
| better-sqlite3 | MIT | `node_modules/better-sqlite3/LICENSE` |
| node-pty | MIT | `node_modules/node-pty/LICENSE` |
| onnxruntime-node | MIT | no `LICENSE` file in the package; MIT grant declared in `package.json` — keep the package's `package.json` to preserve the declaration |

## 6. Summary: what you MUST do before publishing

1. Keep `LICENSE` (Apache-2.0) in the artifact. ✅ already done.
2. Keep the `MODIFIED FILE` headers in both patched sources. ✅ already done.
3. Do **not** ship `vela` or its OpenCode companion. ✅ already stripped.
4. Do **not** call it "Open Design" as a product name. ✅ renamed and disclaimed.
5. **Decide about ffmpeg** — shipped as-is, relying on the upstream's published
   source at `johnvansickle.com/ffmpeg/`. Delete it if you need a GPL-free
   artifact; Motion export stops working if you do.
6. Do not publish `gsap` as a standalone download.
7. Keep libvips `.so` files replaceable (unmodified, unstripped).
8. State that upstream does not support this platform and that this build is
   unofficial.
