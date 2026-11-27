# PR 提交包 — OpenDesign Linux 通道

给上游提两个独立 PR。每个都基于 tag `open-design-v0.27.2`（commit `d9691d7`），
改动最小、互不依赖、可单独评审合并。

**重要：`upstream-pr/*.patch` 是干净版（不含 `MODIFIED FILE` 标注和
`release/patches/` 引用）。`release/patches/*.patch` 是同一个改动但带
Apache-2.0 §4(b) 标注的发布版——两者只差注释，源码行为完全一致。
提 PR 用 `upstream-pr/`，发布自制产物用 `release/patches/`。**

---

## PR 1 — `fix(web): 打包上下文的 CMS host release 守卫误判`

**文件**：`apps/web/next.config.ts`（+6 −1）
**补丁**：`upstream-pr/0001-fix-web-cms-host-release-guard.patch`

### 分支名

```
fix/web-cms-host-release-guard
```

### Commit message

```
fix(web): guard CMS host release on a workspace-only input

`NEXT_PUBLIC_CMS_HOST_RELEASE` was gated on
`existsSync(resolve(WEB_ROOT, 'src/components/touchpoint-component.ts'))`.
The web package's `files` list includes `src`, so that guard is also true
inside the packaged (tarball-installed) context, where the fingerprint's
`apps/web/**` inputs do not exist. `readFileSync` then threw ENOENT while
next.config.ts was loading, and the web sidecar died on start.

Probe `CMS_HOST_RELEASE_INPUTS[0]` instead — it is the first file the
fingerprint actually reads and it only exists in the workspace. In a
workspace build nothing changes (the file is present, so the fingerprint is
emitted exactly as before); in a packaged context the guard is now correctly
false and the variable is `undefined`.

Refs: the Linux packaging lane is the first consumer of a server-runtime
web build installed from tarballs.
```

### PR 描述正文

````markdown
## Why

`NEXT_PUBLIC_CMS_HOST_RELEASE` is a build-time constant derived from a
fingerprint over `CMS_HOST_RELEASE_INPUTS` (18 workspace paths under
`apps/web/**` and `packages/contracts/**`). The guard that decides whether to
compute it was:

```ts
existsSync(resolve(WEB_ROOT, 'src/components/touchpoint-component.ts'))
```

`WORKSPACE_ROOT` / `WEB_ROOT` differ by two path segments, and the web
package's `files` list ships `src/`. So after `pnpm pack` + install, the
guarded file **does exist** inside the installed package, while the files the
fingerprint then reads (`apps/web/app/layout.tsx`, …) **do not**. The result:

```
Error: ENOENT: no such file or directory,
  open '.../node_modules/@open-design/web/apps/web/app/layout.tsx'
```

thrown while `next.config.ts` is being loaded, which takes the web sidecar
down on startup. Verified while bringing up the Linux packaging lane
(`tools-pack linux`, AppImage): the daemon and desktop sidecars came up
healthy while the web sidecar exited immediately with exactly that ENOENT.

## What

Guard on `CMS_HOST_RELEASE_INPUTS[0]` instead — the first path the
fingerprint genuinely reads, and a workspace-only path.

- **Workspace build:** `apps/web/app/layout.tsx` exists → guard is `true` →
  fingerprint emitted exactly as before. No behaviour change.
- **Packaged build:** the path is absent → guard is `false` →
  `NEXT_PUBLIC_CMS_HOST_RELEASE` is `undefined`, which is what the surrounding
  comment already says packaged servers need ("need not retain source files to
  load this config").

## Notes

- `CMS_HOST_RELEASE_INPUTS` is already exported from the same file and
  `cmsHostReleaseFingerprint()` already iterates it, so this reuses the
  existing contract rather than introducing a second notion of "is this the
  workspace".
- `apps/web/tests/runtime/cms-host-release-fingerprint.test.ts` still passes;
  it exercises `cmsHostReleaseFingerprint` directly and is unaffected by the
  guard.
- The guard can't simply test `WORKSPACE_ROOT` presence: `resolveWorkspaceRoot()`
  throws when `OD_WORKSPACE_ROOT` is set to a non-ancestor, and packaged runs
  have no `pnpm-workspace.yaml`, so a root-level probe would fail differently
  rather than return `false`.
````

### 评审自查

- [ ] `pnpm --filter @open-design/web build` 通过
- [ ] `pnpm --filter @open-design/web test` 通过（含 `cms-host-release-fingerprint`）
- [ ] 工作区构建产物里 `NEXT_PUBLIC_CMS_HOST_RELEASE` 仍是 `sha256:…`（可在客户端 JS 里 grep 到）
- [ ] `pnpm typecheck` 通过

---

## PR 2 — `fix(tools-pack): 打通 linux 打包通道`

**文件**：`tools/pack/src/linux.ts`（+27 −1）
**补丁**：`upstream-pr/0002-fix-tools-pack-linux-lane.patch`

### 分支名

```
fix/tools-pack-linux-lane
```

### Commit message

```
fix(tools-pack): make the linux packaging lane produce a runnable AppImage

Three independent blockers, none of which the mac/win lanes hit:

1. `runProductionInstall()` runs outside the workspace, so the workspace's
   `ignoredBuiltDependencies: [onnxruntime-node]` does not apply and the
   package's install hook fires. On Linux it downloads CUDA EP binaries for a
   version whose GPU asset 404s on GitHub, failing the install. The CPU
   runtime already ships in the npm package, so pass
   `ONNXRUNTIME_NODE_INSTALL_CUDA=skip`.

2. `pnpm pack` rewrites `workspace:*` to concrete versions, and the assembled
   app depends on the tarballs via `file:`. Neither npm nor pnpm dedupes a
   transitive `@open-design/<pkg>@<version>` requirement onto a root `file:`
   dependency, so the production install goes to the registry for the
   internal packages — which are private — and 404s. Pin every internal
   package through `overrides` (and `pnpm.overrides`) so all resolution paths
   land on the local tarballs. (pnpm 10.33.2 confirmed to keep its own
   `pnpm.overrides` from overriding a root `file:` dep, hence npm's top-level
   `overrides` carries the fix.)

3. electron-builder's module collector picks its traversal by package
   manager. `npm_execpath` is set by the containerized bootstrap and makes
   detection choose pnpm, whose collector expects a `.pnpm` virtual store and
   silently drops hoisted transitive deps (`setimmediate`, …) from the
   AppImage. Declare `packageManager: "npm@11.11.0"` on the assembled app —
   electron-builder reads that field before lock files and environment — so it
   walks npm's flat layout.

Also documented (no code change) that the Linux lane must NOT add an
Electron-ABI `@electron/rebuild` step: the Linux daemon runs on the bundled
plain Node (`nodeCommandRelative` in the packaged config), so the production
`npm install` already compiles `better-sqlite3` / `node-pty` for the right
ABI. An Electron-ABI rebuild produces a `NODE_MODULE_VERSION`
mismatch.
```

### PR 描述正文

````markdown
## Why

The Linux packaging lane is gated off in CI (`ENABLE_STABLE_LINUX`, with a note
about the containerized pnpm bootstrap), so none of the following had been
exercised end-to-end. Running `pnpm exec tools-pack linux build --to appimage
--containerized` locally surfaces three blockers in sequence, each of which
fails in a way that looks unrelated to the actual cause.

### Blocker 1 — `onnxruntime-node` install hook

`runProductionInstall()` installs into the assembled app directory, which lives
outside the workspace. The workspace's `ignoredBuiltDependencies:
[onnxruntime-node]` therefore does not apply (different install root), the
package's install hook runs, and on Linux it tries to download CUDA execution
provider binaries. For the pinned onnxruntime version that GPU asset does not
exist on GitHub → 404 → install fails. The CPU runtime is already inside the
npm package, and the workspace's own intent is CPU-only.

→ pass `ONNXRUNTIME_NODE_INSTALL_CUDA=skip` for that one install.

### Blocker 2 — internal packages resolve from the registry

`pnpm pack` rewrites `workspace:*` into concrete version ranges. Neither npm
nor pnpm dedupes a transitive `@open-design/<pkg>@<version>` requirement onto a
root `file:` dependency, so a fresh production install resolves some internal
packages through the registry — where they are private — and 404s.

→ pin every internal package with `overrides` **and** `pnpm.overrides`, so
whichever resolver runs, all paths land on the local tarballs. Both are
needed: `install` may run under either manager depending on
`OD_TOOLS_PACK_PNPM_BIN`, and pnpm's own `pnpm.overrides` does not override a
root `file:` dependency (verified on pnpm 10.33.2), so npm's top-level
`overrides` is the one that actually fixes the pnpm path today.

### Blocker 3 — electron-builder's collector picks pnpm

With a flat, npm-installed `node_modules`, electron-builder must collect deps
by walking the npm layout. It chooses that strategy from the detected package
manager, and `npm_execpath` (exported by the containerized bootstrap) makes
detection answer "pnpm" — whose collector expects a `.pnpm` virtual store and
therefore walks a layout that isn't there, dropping hoisted transitive deps
(`setimmediate` and friends) from the AppImage. The app then fails at runtime
inside the daemon with a missing module, far from the packaging step.

→ declare `packageManager: "npm@11.11.0"` on the assembled app's
`package.json`. electron-builder reads that field before lock files and env.

### Also: no `@electron/rebuild` on Linux

Worth stating because it is the obvious-looking fix and it is wrong. The
mac/win lanes run their daemons as Electron-as-node and need an Electron-ABI
rebuild of the native modules. The Linux lane runs the daemon on the bundled
plain Node (`nodeCommandRelative` in `open-design-config.json`), so the
production `npm install` has already compiled `better-sqlite3` and `node-pty`
via node-gyp against exactly that Node. Rebuilding for Electron ABI leaves the
daemon with a `NODE_MODULE_VERSION` mismatch. I added the comment in this
file so the next person doesn't rediscover it the hard way. (The stray comment
is attached to `runElectronBuilderLinux`; happy to relocate it if you prefer
it elsewhere.)

## Verification

On Ubuntu (glibc 2.34, Wayland) and in the containerized build:

- `tools-pack linux build --to appimage --containerized` completes; produces a
  ~580 MB AppImage.
- Launched from FUSE: desktop, web and daemon sidecars each report
  `ready: true` over IPC; all three survive well past the 90 s bootstrap
  deadline.
- `GET /api/health` → `{"ok":true,"version":"0.27.2"}`; `GET /` → 200.
- Main window reaches `dom-ready` / `did-finish-load` / `loadURL success` on
  `od://app/onboarding`.
- 478 bundled plugins register; `/api/agents` detects host CLIs normally.
- Native modules check out: `better_sqlite3.node` exports
  `node_register_module_v137` (Node 24 ABI), `pty.node` builds, and hoisted
  transitive deps such as `setimmediate` are present in the AppImage.

## Notes

- These are three independent fixes; they are in one commit because the lane
  cannot be exercised end-to-end without all three. Happy to split them into
  three PRs if you'd rather review them separately — say the word.
- No test changes. `tools/pack/tests/linux.test.ts` still passes (it pins the
  hardcoded pnpm version against root `packageManager`, which this doesn't
  touch).
- Enablement: this does not flip `ENABLE_STABLE_LINUX`. Whether the lane should
  become CI-visible is your call; this just makes the lane correct when it is
  run.
````

### 评审自查

- [ ] `pnpm --filter tools-pack test`（或 `tools/pack/tests/linux.test.ts`）通过
- [ ] `pnpm typecheck` 通过
- [ ] 至少一次真实 `tools-pack linux build` 成功产出可运行 AppImage
- [ ] 若上游要求拆分：把 3 处改动拆成 3 个 commit / PR（注释即可拆）

---

## 提交命令

```bash
# 准备
git clone https://github.com/nexu-io/open-design.git od-pr
cd od-pr
git remote add fork git@github.com:<your-handle>/open-design.git

# ---- PR 1 ----
git checkout -b fix/web-cms-host-release-guard open-design-v0.27.2
git apply /path/to/upstream-pr/0001-fix-web-cms-host-release-guard.patch
git commit -am "fix(web): guard CMS host release on a workspace-only input"
git push -u fork fix/web-cms-host-release-guard
# → open PR against main, paste the body from above

# ---- PR 2 ----（从 main 重新开分支，不要叠在 PR 1 上，因为两者无依赖）
git checkout main
git pull
git checkout -b fix/tools-pack-linux-lane
git apply /path/to/upstream-pr/0002-fix-tools-pack-linux-lane.patch
git commit -am "fix(tools-pack): make the linux packaging lane produce a runnable AppImage"
git push -u fork fix/tools-pack-linux-lane
# → open PR against main, paste the body from above
```

## 合并之后你能得到什么

- 可以直接基于官方 tag 构建 Linux 包，**不再需要 `MODIFIED FILE` 标注**，
  也就没有"衍生版"的商标/署名负担（只剩剥 vela 这一步）。
- 上游 CI 若要开启 `ENABLE_STABLE_LINUX`，这两个补丁就是前置条件。
- `release/` 目录里的 README / NOTICES 可以相应简化。
