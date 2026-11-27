# OpenDesign for Linux — unofficial build

An **unofficial** Linux (x86-64) build of [Open Design](https://github.com/nexu-io/open-design),
produced so Linux/Ubuntu users can run the app. The upstream project publishes
release artifacts for macOS and Windows only.

- **Upstream:** `nexu-io/open-design`, tag `open-design-v0.27.2` (commit `d9691d7`)
- **License:** Apache License 2.0 (see [`LICENSE-APACHE-2.0.txt`](compliance/LICENSE-APACHE-2.0.txt))
- **Not affiliated with, sponsored by, or endorsed by the upstream project.**

## ⚠️ Trademark / naming notice

Section 6 of the Apache License 2.0 does **not** grant trademark rights. This
build therefore does not present itself as the official product:

- the AppImage is named `OpenDesign-Linux-<version>-clean.AppImage` (not
  `Open Design`), and its desktop entry says
  `OpenDesign Linux (unofficial build)`;
- "Open Design" appears only to describe the origin of the work, which §6
  explicitly permits;
- nothing here should be read as a statement that the upstream maintainers
  produced, reviewed, or support this artifact.

## What this build changes

Two source files are patched relative to upstream `open-design-v0.27.2`. Both
patches are minimal, self-contained, and were written to be upstreamable; each
modified file carries a prominent "MODIFIED FILE" header comment as required by
Apache-2.0 §4(b).

| Patch | File | Why |
|---|---|---|
| [`0001`](patches/0001-web-next-config-cms-host-release-guard.patch) | `apps/web/next.config.ts` | The `NEXT_PUBLIC_CMS_HOST_RELEASE` guard probed a file that also ships inside the web package tarball, so it was true in the packaged (installed) context too. The fingerprint then tried to read `apps/web/...` workspace sources that do not exist there → `ENOENT` → web sidecar crashed. Guard on `CMS_HOST_RELEASE_INPUTS[0]`, a workspace-only path. |
| [`0002`](patches/0002-tools-pack-linux-production-install.patch) | `tools/pack/src/linux.ts` | Makes the Linux packaging lane work at all: (a) `ONNXRUNTIME_NODE_INSTALL_CUDA=skip` for the out-of-workspace production install, whose `onnxruntime-node` install hook otherwise 404s on a missing CUDA asset; (b) `overrides` / `pnpm.overrides` pinning every `@open-design/*` internal package to its local tarball, since neither npm nor pnpm dedupes a transitive version range onto a root `file:` dependency and the registry copy is private; (c) `packageManager: "npm@11.11.0"` so electron-builder's module collector walks npm's flat layout instead of a pnpm virtual store it cannot read. |

Upstream's own CI keeps the Linux lane disabled behind an `ENABLE_STABLE_LINUX`
flag (with a note about the containerized pnpm bootstrap yet to be reviewed), so
this build is the first end-to-end pass of that lane.

**No other source changes were made.** `apps/packaged`, `apps/daemon`,
`apps/desktop`, `packages/*`, skills, design systems and plugins are byte-for-byte
the upstream tag.

## ⚠️ What this build removes (license compliance)

Upstream's packaged payload bundles `@powerformer/vela-cli` (the *Vela* CLI plus
its bundled OpenCode companion). Both npm packages declare
`"license": "UNLICENSED"` — see
`tools/pack/package.json` `optionalDependencies` → `@powerformer/vela-cli@0.1.9`
→ `@powerformer/vela-cli-linux-x64@0.1.9`. **`UNLICENSED` grants no redistribution
rights at all**, so that binary is stripped from this artifact:

| Removed path | Size | Package | License |
|---|---|---|---|
| `resources/open-design/bin/vela` | ~20 MB | `@powerformer/vela-cli-linux-x64` | `UNLICENSED` |
| `resources/open-design/bin/libexec/opencode/opencode` | ~174 MB | same package (`files` list) | `UNLICENSED` |

Functional consequence: the **AMR** agent runtime (`vela agent run`, Team
collaboration features) is unavailable in this build. Everything else works —
agent detection finds and launches your own installed CLIs (`opencode`, `pi`,
`claude`, `codex`, …), design skills, design systems, design templates, plugins,
artifact previews, and video/audio poster extraction all behave normally.
If you have a legitimate license for Vela, drop the binary back into
`resources/open-design/bin/vela` and the AMR runtime will pick it up.

A second, hygiene-driven removal: the upstream `@ffmpeg-installer/ffmpeg` npm
package was published with the author's `.env` (an `NPM_CONFIG_TOKEN`) and a set
of editor `~` backup files. Neither is read at runtime, so this build excludes
them — otherwise redistributing the artifact would re-expose that token:

| Removed path | Package | Reason |
|---|---|---|
| `resources/app/node_modules/@ffmpeg-installer/ffmpeg/.env` | `@ffmpeg-installer/ffmpeg` | leaked `NPM_CONFIG_TOKEN` |
| `resources/app/node_modules/@ffmpeg-installer/ffmpeg/*~` (12 files) | same | editor backup clutter |

ffmpeg's GPLv3 static binary is untouched, so poster extraction and Motion export
still work.

## Third-party components

The artifact is a repackaging of upstream's payload; the bulk of the licenses
ship inside it. Summary of what is worth knowing (full inventory in
[`compliance/THIRD-PARTY-NOTICES.md`](compliance/THIRD-PARTY-NOTICES.md)):

| Component | License | Note |
|---|---|---|
| Open Design core (`@open-design/*`) | Apache-2.0 | `LICENSE` retained in the AppDir |
| Electron + Chromium | MIT / BSD-style | `LICENSE.electron.txt`, `LICENSES.chromium.html` retained |
| Bundled Node.js runtime | MIT | used as the daemon/web runtime |
| better-sqlite3, node-pty, onnxruntime-node | MIT | native modules |
| Albert Sans, RemixIcon, Inter, Roboto Mono | SIL OFL 1.1 / OFL | fonts |
| gsap | GreenSock "no charge" standard license | **not** an OSI license; bundling as part of an unmodified app payload is contemplated by its terms, but redistributing gsap on its own is not |
| ffmpeg (`@ffmpeg-installer/linux-x64`) | **GPLv3** (build config `--enable-gpl --enable-version3`) | statically linked binary. GPL §3 source-availability applies; the exact upstream build is `johnvansickle.com/ffmpeg` `N-47683-g0e8eb07980-static`, whose source is published there. **If you need a GPL-free artifact**, delete `resources/app/node_modules/@ffmpeg-installer/linux-x64/ffmpeg`; poster extraction then degrades gracefully, but Motion artifact export stops working (`assertMotionExport()` needs a decodable frame). |
| sharp / libvips | LGPL-3.0-or-later | dynamic `.so`, replaceable |
| posthog-js | Apache-2.0 | in-app analytics; telemetry is off unless a sink is configured |
| jszip | MIT **or** GPL-3.0-or-later | dual; MIT branch used |
| Bundled skills / templates / plugins | their own (mostly MIT) | upstream keeps them; `design-templates/guizang-ppt` (MIT), `html-ppt` (MIT), `skills/web-clone` (MIT) etc. retain their LICENSE files inside the payload |

## Building it yourself

```bash
git clone https://github.com/nexu-io/open-design.git
cd open-design
git checkout open-design-v0.27.2
git apply /path/to/0001-web-next-config-cms-host-release-guard.patch
git apply /path/to/0002-tools-pack-linux-production-install.patch

pnpm exec tools-pack linux build --to appimage --containerized --portable \
  --namespace release-stable-linux --app-version 0.27.2 --dir /tmp/od-build
```

Then strip the Vela payload for redistribution:

```bash
APP=…/linux-unpacked/resources/open-design/bin
rm -rf "$APP/vela" "$APP/libexec/opencode"
```

The build already strips the leaked `@ffmpeg-installer/ffmpeg/.env` (an npm
token) and its `~` backup files — the packaging `files` list excludes them — so
no manual step is needed for that part.

Runs on glibc ≥ 2.34 (Ubuntu 22.04+, Debian 12+). It will **not** run on Ubuntu
20.04 (glibc 2.31).

## Running

```bash
chmod +x OpenDesign-Linux-0.27.2-clean.AppImage
./OpenDesign-Linux-0.27.2-clean.AppImage
```

**If it fails to start, relaunch with `--no-sandbox`.** The typical failure is
Chromium's SUID-sandbox check aborting the launch (`The SUID sandbox helper binary
was found, but is not configured correctly … aborting now` → core dump). An AppImage
unpacks to a per-launch `/tmp/.mount_*/` dir owned by your user, so the
`chrome-sandbox` helper inside it is not root-owned with mode 4755, and Chromium
refuses to start. Disable the sandbox:

```bash
./OpenDesign-Linux-0.27.2-clean.AppImage --no-sandbox
```

No FUSE (containers/WSL):

```bash
./OpenDesign-Linux-0.27.2-clean.AppImage --appimage-extract
cd squashfs-root && ./AppRun --no-sandbox
```

(If you specifically want the sandbox on, extract once and run
`sudo chown root:root squashfs-root/chrome-sandbox && sudo chmod 4755
squashfs-root/chrome-sandbox`, then start `squashfs-root/AppRun` without the flag.)

Data, logs and runtime state live in
`~/.config/Open Design/namespaces/release-stable-linux/{data,logs,runtime}/`.
Note that this path is inherited from upstream and is shared with an official
install of the same namespace — running both at once will fight over the
sidecar IPC sockets and the SQLite database.

**First launch is slow** (FUSE cold page cache, one-time plugin seeding of 478
plugins, SQLite init, window paint can exceed 90 s on a cold disk). The
launcher's bootstrap window is 90 s; if it times out, relaunch — the second
start is fast because the page cache and plugin registry are warm.

## Verification performed

- daemon, web and desktop sidecars each report `ready: true` over IPC; sustained
  well past the 90 s bootstrap deadline.
- `GET /api/health` → `{"ok":true,"version":"0.27.2"}`; `GET /` → HTTP 200.
- Main window reaches `dom-ready` / `did-finish-load` / `loadURL success` on
  `od://app/onboarding`.
- 478 plugins registered; agent detection lists your host CLIs.
- Artifact: 524,075,545 bytes, `sha256
  aa4fbf9906c3380f0a0ce3ee0eb3b6d533e424f5d30bc65ef982aa3193f6d127`
  (see `SHA256SUMS.txt`).

## Caveats

- **Upstream never tested this platform.** Bugs may be Linux-specific and are
  not upstream's responsibility under their support terms, though they are
  welcome as reports if they reproduce on macOS/Windows.
- **Auto-update is disabled on Linux** by design (`unsupported-platform`); this
  build is not updatable in place.
- **Unaudited for distribution at scale**: the native modules were compiled on
  the build host and are glibc-version-sensitive. Test on your target distro
  before distributing widely.
- The app's updater config (`resources/app-update.yml`) still points at
  `nexu-io/open-design`. Since updates are unsupported on Linux the field is
  inert, but it should be changed if you host your own releases.

---

## Which file to use when publishing

| Need | File |
|---|---|
| GitHub **Release** body (the page users land on) | [`GITHUB-RELEASE-NOTES.md`](GITHUB-RELEASE-NOTES.md) |
| The bundle's own README / repo front page | this file |
| Go/no-go before uploading | [`PUBLISH-CHECKLIST.md`](PUBLISH-CHECKLIST.md) |
| License analysis for redistribution | [`compliance/THIRD-PARTY-NOTICES.md`](compliance/THIRD-PARTY-NOTICES.md) |
| Patch with Apache-2.0 §4(b) "modified file" headers (for shipping your own artifact) | `patches/` |
| Clean, upstream-style patch to submit as a PR | [`upstream-pr/`](upstream-pr/) + [`upstream-pr/PR-PACKAGE.md`](upstream-pr/PR-PACKAGE.md) |

`patches/` and `upstream-pr/` contain the **same two source changes**; they differ
only in comment text. `patches/` adds the "MODIFIED FILE" header and cross-references
that a redistributed derivative needs; `upstream-pr/` is what you send upstream.

Build detail verified against the artifact rather than assumed: the glibc 2.34
floor comes from `node-pty`'s `fd`/`openpty`/`forkpty` symbol versions, while the
bundled Node runtime needs only 2.28.
