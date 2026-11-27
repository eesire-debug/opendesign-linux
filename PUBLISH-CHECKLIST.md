# Publish checklist

Run through this before pushing the artifact anywhere public.
Verified against `OpenDesign-Linux-0.27.2-clean.AppImage`
(SHA-256 `aa4fbf9906c3380f0a0ce3ee0eb3b6d533e424f5d30bc65ef982aa3193f6d127`,
524,075,433 B) on 2026-11-27.

## Artifact

- [x] `sha256sum -c SHA256SUMS.txt` passes against the file you are uploading.
- [x] `./OpenDesign-Linux-0.27.2-clean.AppImage --appimage-extract` succeeds and
      the tree contains `LICENSE`, `LICENSE.electron.txt`, `LICENSES.chromium.html`.
- [x] `resources/open-design/bin/` contains **only** `node` — no `vela`, no
      `libexec/opencode/`.
- [x] `grep -r "vela" resources/open-design-config.json` finds nothing.
- [x] `Open Design.desktop` says `Name=OpenDesign Linux (unofficial build)`.
- [x] Launch it, wait for the window, confirm onboarding renders and
      `GET /api/health` returns `{"ok":true,"version":"0.27.2"}`.

## Strips (license + hygiene)

- [x] Vela CLI + OpenCode companion stripped (`resources/open-design/bin/{vela,libexec/opencode}`).
- [x] Leaked `@ffmpeg-installer/ffmpeg/.env` (an `NPM_CONFIG_TOKEN`) stripped.
- [x] All `@ffmpeg-installer/ffmpeg/*~` editor backup files stripped.
- [x] No other `.env` or `~` files anywhere in the payload.

## PII / secret scan (full payload)

- [x] No builder identity or host path leaked (no `/home/<user>/`, no build-host name).
- [x] No private keys (`BEGIN PRIVATE/RSA/OPENSSH`) — only library checks + docs examples.
- [x] No JWT secrets — only the standard zod test vector.
- [x] No real API keys — the one `phc_…` is a **public** PostHog key from the
      upstream `hyperframes` package; `AKIA…` hits are base64-WASM byte coincidences.
- [x] Only real emails are Chromium/FreeBSD/OpenBSD license-attribution addresses
      (public, ship in every Electron app).

## Docs

- [x] `README.md` states: upstream tag + commit, unofficial, not affiliated,
      Apache-2.0, what was patched, what was stripped, and that Linux auto-update
      is unsupported.
- [x] `README.md` + `GITHUB-RELEASE-NOTES.md` lead with **`--no-sandbox`** as the
      first fix for a startup error (the SUID-sandbox abort).
- [x] `compliance/THIRD-PARTY-NOTICES.md` is current for the exact artifact.
- [x] `patches/*.patch` apply cleanly to a fresh `open-design-v0.27.2` clone
      (`git apply --check`).
- [x] The documented patches, applied to a clean tag, reproduce the exact source
      that built this binary (diff is empty) — no undocumented drift.

## Licensing decisions still yours

- [ ] ffmpeg: keep (GPLv3 binary, rely on the published upstream source) or
      delete (Motion export stops working, poster extraction degrades).
- [ ] gsap: never publish it as a standalone downloadable asset.
- [ ] libvips: keep the `.so` files unmodified so they remain replaceable.
- [ ] Do not use "Open Design" as the name of *your* product, repo, or release.

## Release mechanics

- [x] Name the release/asset without the trademark: `OpenDesign-Linux-0.27.2-clean`.
- [x] Release notes link upstream (`https://github.com/nexu-io/open-design`)
      and the tag, so the provenance is one click away.
- [x] Say plainly that upstream ships macOS/Windows only and does not support
      Linux, so nobody files Linux bugs upstream against your build.
- [ ] If you host a repo: the repo's own `LICENSE` is Apache-2.0 and its README
      repeats the unofficial disclaimer on the first screen.
