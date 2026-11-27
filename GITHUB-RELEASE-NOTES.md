## OpenDesign for Linux — unofficial build (v0.27.2)

A working Linux (x86-64) build of [Open Design](https://github.com/nexu-io/open-design),
built because upstream publishes macOS and Windows artifacts only.

**Unofficial.** Produced by Linux users, not by the upstream maintainers, and not
reviewed or endorsed by them. Upstream does not support Linux, so please do not
file Linux bugs against their tracker unless the bug also reproduces on
macOS/Windows.

> **glibc ≥ 2.34.** Works on Ubuntu 22.04+, Debian 12+, Fedora 36+. Does **not**
> run on Ubuntu 20.04 (glibc 2.31). The 2.34 floor comes from `node-pty`'s
glibc binaries compiled on the build host (`openpty` / `forkpty`), not from
the bundled Node runtime itself (which needs only 2.28).

---

### Download & run

| File | Size | SHA-256 |
|---|---|---|
| `OpenDesign-Linux-0.27.2-clean.AppImage` | 524,075,545 B | `aa4fbf9906c3380f0a0ce3ee0eb3b6d533e424f5d30bc65ef982aa3193f6d127` |

```bash
chmod +x OpenDesign-Linux-0.27.2-clean.AppImage
./OpenDesign-Linux-0.27.2-clean.AppImage
```

**If it fails to start — try `--no-sandbox`.** The most common startup error is
Chromium's SUID-sandbox check aborting the launch:

```
FATAL:sandbox/linux/suid/… The SUID sandbox helper binary was found, but is not
configured correctly. Rather than run without sandboxing I'm aborting now. …
Trace/breakpoint trap (core dumped)
```

An AppImage unpacks to a per-launch `/tmp/.mount_*/` directory owned by *you*, so
the `chrome-sandbox` helper inside it is not root-owned with mode 4755, and
Chromium refuses to start. Relaunch with the sandbox disabled:

```bash
./OpenDesign-Linux-0.27.2-clean.AppImage --no-sandbox
```

That resolves the sandbox abort. (If you specifically want the sandbox on, extract
once and run `sudo chown root:root squashfs-root/chrome-sandbox && sudo chmod 4755
squashfs-root/chrome-sandbox`, then start `squashfs-root/AppRun` without the flag —
but `--no-sandbox` is the path of least resistance.)

No FUSE (containers, WSL):

```bash
./OpenDesign-Linux-0.27.2-clean.AppImage --appimage-extract
cd squashfs-root && ./AppRun --no-sandbox
```

---

### What works

- Desktop app (Electron), the Next.js web UI, and the `od` daemon all start and
  report healthy.
- Agent detection finds and launches **your own** installed CLIs — `opencode`,
  `pi`, `claude`, `codex`, `aider`, `amp`, `kimi`, `kiro`, `qwen`, `hermes`,
  `cursor-agent`, `copilot`, `devin`, `trae-cli`, and the rest of the bench.
- 478 bundled plugins register; skills, design systems, design templates, prompt
  templates, and artifact previews all behave normally.
- Projects, SQLite persistence, sandboxed preview, video/audio poster extraction.

### What does not work in this build

**The AMR agent runtime and Vela-backed Team collaboration are disabled.** Upstream
bundles the Vela CLI (`@powerformer/vela-cli`, plus its OpenCode companion) into
the payload, and both npm packages declare `"license": "UNLICENSED"` — which
grants no redistribution rights at all, not even the right to copy the file. So
they are stripped from this artifact:

```
resources/open-design/bin/vela                      (20 MB, stripped)
resources/open-design/bin/libexec/opencode/opencode (174 MB, stripped)
```

Consequences: the AMR card shows as unavailable, and Team sharing/collab paths
that shell out to `vela` are inert. If you hold a legitimate Vela license, drop
the binary back at `resources/open-design/bin/vela` and AMR will pick it up.

**Auto-update is disabled on Linux** by design (`unsupported-platform`). This
build is not updatable in place; watch this repo for new releases.

**A leaked credential and working-dir clutter are stripped.** The upstream
`@ffmpeg-installer/ffmpeg` npm package was published with the author's `.env`
file — containing an `NPM_CONFIG_TOKEN` — plus a set of editor `~` backup files
(`download.js~`, `package.json~`, …). Neither is read at runtime (the package's
entry is `index.js`), so this build excludes them from the payload:

```
resources/app/node_modules/@ffmpeg-installer/ffmpeg/.env   (npm token, stripped)
resources/app/node_modules/@ffmpeg-installer/ffmpeg/*~     (12 backup files, stripped)
```

This is a hygiene fix for redistribution: the token is the upstream author's,
but shipping it would re-expose it in every copy of this artifact. ffmpeg itself
(the GPLv3 static binary) is untouched, so poster extraction and Motion export
still work.

---

### What this build changes vs upstream

Built from tag **`open-design-v0.27.2`** (commit `d9691d7`). Two source files are
patched — both minimal, both written to be upstreamable, each carrying a
prominent `MODIFIED FILE` header as Apache-2.0 §4(b) requires.

**1. `apps/web/next.config.ts`** — the `NEXT_PUBLIC_CMS_HOST_RELEASE` guard probed
`src/components/touchpoint-component.ts`, which the web package's `files` list
ships. So inside a tarball-installed (packaged) context the guard was `true` while
the files the fingerprint then reads (`apps/web/**`) do not exist →
`readFileSync` threw `ENOENT` while `next.config.ts` loaded → the web sidecar died
on start. Now guards on `CMS_HOST_RELEASE_INPUTS[0]`, a workspace-only path.

**2. `tools/pack/src/linux.ts`** — three blockers in the Linux packaging lane,
which upstream CI keeps disabled behind `ENABLE_STABLE_LINUX`:

- the out-of-workspace production install runs `onnxruntime-node`'s install hook,
  which 404s downloading a CUDA EP asset that does not exist for the pinned
  version → inject `ONNXRUNTIME_NODE_INSTALL_CUDA=skip` (CPU runtime ships in the
  npm package);
- `pnpm pack` rewrites `workspace:*` to concrete versions, and neither npm nor
  pnpm dedupes a transitive `@open-design/<pkg>@<version>` requirement onto a root
  `file:` tarball dependency, so the install goes to the registry for internal
  packages that are private → 404. Pin every internal package via `overrides`
  (and `pnpm.overrides`);
- electron-builder picks its module-collection strategy from the detected package
  manager, and `npm_execpath` (set by the containerized bootstrap) made it choose
  pnpm, whose collector expects a `.pnpm` virtual store and silently dropped
  hoisted transitive deps (`setimmediate`, …) from the AppImage → declare
  `packageManager: "npm@11.11.0"` on the assembled app.

Everything else — `apps/daemon`, `apps/desktop`, `apps/packaged`, `packages/*`,
skills, design systems, plugins, templates — is byte-for-byte upstream.

Patch files and full rationale: `patches/` and `README.md` in the release bundle.

---

### Verification

Run on Ubuntu (glibc 2.34, Wayland) and inside the containerized build:

- desktop / web / daemon sidecars each report `ready: true` over IPC, and all
  three survive well past the launcher's 90-second bootstrap deadline;
- `GET /api/health` → `{"ok":true,"version":"0.27.2"}`; `GET /` → HTTP 200;
- main window reaches `dom-ready` / `did-finish-load` / `loadURL success` on
  `od://app/onboarding`;
- 478 bundled plugins register; `/api/agents` detects host CLIs normally;
- `better_sqlite3.node` exports `node_register_module_v137` (Node 24 ABI),
  `pty.node` builds, hoisted transitive deps present.

---

### First launch is slow

FUSE cold page cache, one-time plugin seeding (478 plugins), SQLite init, and the
first window paint can together exceed the launcher's 90-second bootstrap window
on a cold disk. If it times out, relaunch — the second start is fast because the
page cache and plugin registry are warm. This is the single most common "it
didn't start" report, and it is not a crash.

Logs: `~/.config/Open Design/namespaces/release-stable-linux/logs/`

---

### Licensing

- **Core Open Design:** Apache-2.0. Full text ships in the artifact at
  `LICENSE` and `usr/share/doc/opendesign/LICENSE-APACHE-2.0.txt`.
- **Electron / Chromium:** `LICENSE.electron.txt`, `LICENSES.chromium.html` in the
  artifact.
- **Not affiliated with, sponsored by, or endorsed by the upstream project.**
  Apache-2.0 §6 grants no trademark rights; "Open Design" appears here only to
  describe the origin of the work, and the app's own desktop entry is named
  `OpenDesign Linux (unofficial build)`.
- **Known third-party terms worth reading** (details in
  `compliance/THIRD-PARTY-NOTICES.md`): ffmpeg is a **GPLv3** static binary
  (`N-47683-g0e8eb07980-static`, source published at johnvansickle.com/ffmpeg);
  `gsap` is under GreenSock's standard no-charge license (**not** an OSI license);
  libvips is **LGPL-3.0** and its `.so` files are kept unmodified and replaceable;
  `jszip` is dual MIT/GPL (MIT elected); bundled skills and templates keep their
  own licenses.

If you fork or redistribute this artifact, delete
`resources/app/node_modules/@ffmpeg-installer/linux-x64/ffmpeg` if you need a
GPL-free build. Poster extraction degrades gracefully without it; Motion artifact
export stops working (it cannot verify a decodable frame).

---

### Building it yourself

```bash
git clone https://github.com/nexu-io/open-design.git
cd open-design && git checkout open-design-v0.27.2
git apply 0001-web-next-config-cms-host-release-guard.patch
git apply 0002-tools-pack-linux-production-install.patch

pnpm exec tools-pack linux build --to appimage --containerized --portable \
  --namespace release-stable-linux --app-version 0.27.2 --dir /tmp/od-build

# strip the UNLICENSED Vela payload before redistributing
rm -rf <out>/linux-unpacked/resources/open-design/bin/{vela,libexec/opencode}
```

The build also strips the leaked `@ffmpeg-installer/ffmpeg/.env` (npm token) and
its `~` backup files automatically — the packaging `files` list excludes them, so
the produced AppImage is already clean on that front.

Repacking an existing AppImage instead (about 40 seconds, no container needed):

```bash
./OpenDesign-Linux-0.27.2-clean.AppImage --appimage-extract
rm -rf squashfs-root/resources/open-design/bin/{vela,libexec/opencode}
# drop the leaked ffmpeg .env + editor backups if the source AppImage has them
rm -f squashfs-root/resources/app/node_modules/@ffmpeg-installer/ffmpeg/.env
find squashfs-root/resources/app/node_modules/@ffmpeg-installer/ffmpeg -name '*~' -delete
# add LICENSE, rename the desktop entry, then:
ARCH=x86_64 appimagetool --comp zstd squashfs-root OpenDesign-Linux-0.27.2-clean.AppImage
```

---

**Upstream:** https://github.com/nexu-io/open-design · tag `open-design-v0.27.2`
**Issue tracker for this build:** please open issues in this repository, not upstream.
