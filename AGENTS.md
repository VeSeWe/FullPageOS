# AGENTS.md — FullPageOS

Guidance for AI agents (and humans) working in this repository. Act as a
senior Linux-distribution, Raspberry Pi, systemd, Docker, and release
engineer.

## What this project is

FullPageOS is a Raspberry Pi OS distribution that boots straight into a
full-screen Chromium kiosk. Images are built with
[CustomPiOS](https://github.com/guysoft/CustomPiOS) (an external, non-vendored
dependency), which composes "modules" into a chroot of a Raspberry Pi OS Lite
base image.

Current mission: modernize to Raspberry Pi OS Lite on Debian 13 (Trixie) for
Raspberry Pi 3/4/5 — 64-bit (`arm64`) default build, `armhf` kept as a
variant, standard images staying lightweight and Docker-free, plus a genuinely
optional Docker variant (Engine + Buildx + Compose v2 from Docker's official
Debian repo) with verified kernel/runtime capabilities and bounded logging,
and a generic kiosk wait-for-URL feature. Pi 1, Pi 2, the original Zero, and
Zero W are out of scope. The user's containerized service lives outside this
repo — never embed application stacks, private images, or registry
credentials.

**TODO.md is the single source of truth** for scope, decisions (D1–D9), the
build matrix, task list (FPOS-001…021), hardware gates (HW-1…3), blockers,
and the work log. Discovery (FPOS-001) is complete; do not redo it.

## Repository layout

- `src/build_dist` — build entry point; delegates to
  `${CUSTOM_PI_OS_PATH}/build_custom_os`. `src/custompios_path` is generated
  and gitignored.
- `src/config` — main build config (`MODULES=...`, `DIST_VERSION`, sizes,
  `GUI_STARTUP_SCRIPT`). Overridable via `src/config.local` (untracked) and
  `src/variants/<name>/config`.
- `src/modules/fullpageos/` — the FullPageOS CustomPiOS module:
  - `config` — `FULLPAGEOS_*` defaults.
  - `start_chroot_script` — runs inside the image chroot at build time.
  - `filesystem/boot/` → copied to the boot partition
    (`${BASE_BOOT_MOUNT_PATH}`): `fullpageos.txt` (kiosk URL), etc.
  - `filesystem/opt/custompios/scripts/` — runtime kiosk scripts
    (`run_onepageos`, `start_chromium_browser`, `get_url`, refresh helpers).
  - `filesystem/root_init/` → copied to `/` (systemd units: splashscreen,
    x11vnc, clear_lighttpd_cache).
- `src/variants/` — variant configs (only `no-acceleration` is tracked;
  `.gitignore` ignores the rest — update it when adding tracked variants).
- `.github/workflows/main.yml` — CI image build.
- `src/image/README` — where the base image goes for local builds.

Untracked local-only files (`node_modules/`, `package.json`, lockfiles,
`pnpm-lock.yaml`, `yarn.lock`, `bunfig.toml`, `graphify-out/`, editor dirs)
are **not part of the project**. Never commit, modify, or delete them.

## Mandatory working method — one TODO item per turn

Work on **exactly one atomic TODO item per turn**, even when items are
related. Never implement several items in one response.

At the start of every turn:

1. Read `TODO.md`.
2. Inspect `git status` and preserve all existing user changes.
3. Consult the graphify repository map (installed graphify tools /
   `graphify-out/`).
4. Verify relevant graphify findings against the actual source files before
   acting on them.
5. Select only the next incomplete TODO item.
6. State which single item you are working on and its acceptance criteria.

At the end of every turn:

1. Run the tests appropriate to that one item.
2. Update `TODO.md`: status, files changed, decisions made, commands/tests
   run, results, remaining risks or blockers.
3. Mark the item complete only when its acceptance criteria pass.
4. Do not begin another item.
5. Stop and wait for the user to say `continue`.

Task states in TODO.md: `[ ]` not started, `[~]` the one current task (never
more than one), `[x]` completed and verified, `[!]` blocked.

If an item grows beyond one cohesive change, split it into smaller TODO items
before implementing — splitting/planning counts as that turn's single item;
stop afterward. If blocked, document the blocker and evidence in `TODO.md`,
then stop. Never hide failures or mark partially verified work complete. A
test failure produces a new or blocked TODO item — never a silently weakened
acceptance criterion.

## Workflow rules

- Respect the recorded decisions (D1–D9). New proposals and hardware gates
  (HW-1…3) need user input — don't self-approve them. Hardware gates close
  only on user-reported results; prepare precise test instructions when a
  gate is reached.
- **Never commit unless the user explicitly asks.** All work happens on
  branch `dev64`, based on tag 0.14.0 (D7) — this fork is for the user's own
  use, merging into upstream `devel` is not a goal; do not rebase onto
  `devel`. Never commit to `devel`/`master` directly; no force-pushes,
  resets, cleans, or history rewrites.
- Commit only intended files. `git status` must stay clean of local-only
  noise (see above).

## Environment constraints (this sandbox)

- **No local image builds** (B1): no root/sudo, no loop devices, no
  qemu/binfmt. Real builds and image inspection happen in GitHub Actions;
  boot/runtime proof happens at user-run hardware gates. Don't attempt
  loop-mounts or chroot builds locally.
- **GitHub egress may be blocked** (B2): verify external facts via WebSearch
  or CI logs, and record the evidence in TODO.md rather than assuming.
- The sandbox itself is aarch64 — useful for sanity-running arm64-targeted
  scripts (read-only checks only).

## Verification expectations

Layered: (1) static checks, (2) chroot/image inspection in CI, (3) emulated
tests, (4) Docker runtime smoke tests where possible, (5) hardware gates.
State honestly what each layer proves — generic QEMU does not prove Pi
firmware/GPU/display behavior, and a check that only ran in the sandbox is
"partial signal", not proof.

- Shell: `shellcheck` and `bash -n` every changed script. Note: chroot/config
  scripts are CustomPiOS fragments — source them in a small bash harness to
  assert exported variables (e.g. `BASE_ARCH`, `MODULES` composition) rather
  than executing them for real.
- JSON: validate with `jq` (e.g. `/etc/docker/daemon.json` templates —
  merge, don't clobber).
- CI YAML: parse-check locally before pushing.
- Prefer runtime capability checks over assumptions based on filenames or
  historical folklore.
- Record verification evidence (command + result) in the TODO.md work log.
  If something can only be proven in CI or on hardware, say so explicitly —
  never claim unverified behavior works.

## Project conventions & gotchas

- Target Trixie: the browser package/binary is `chromium`
  (`chromium-browser` is a transitional dummy); use `${BASE_USER}` instead of
  hard-coded `pi`; boot partition is `/boot/firmware` at runtime and
  `${BASE_BOOT_MOUNT_PATH}` in chroot scripts; `apt-get --force-yes` is
  obsolete; time sync is systemd-timesyncd (not ntp.conf). Account for real
  Trixie changes — no minimal string-replacement of the old build.
- The kiosk stack is X11 (lightdm/startx/xdotool/x11vnc) by decision D3 —
  don't migrate to Wayland as a side effect.
- CustomPiOS module convention: module config variables are prefixed with the
  module name in caps (e.g. `FULLPAGEOS_DOCKER_*` for
  `src/modules/fullpageos-docker`).
- Pin mutable inputs (D6): dated base-image URLs + sha256, pinned CustomPiOS
  ref. Never introduce `*_latest` URLs, unpinned refs, `curl | sh`, or
  `apt-key`.
- Artifact names must derive from the build's own arch/variant variables so
  an `armhf` build can never be labeled `arm64`. A 64-bit image must report
  `uname -m` = `aarch64` and `dpkg --print-architecture` = `arm64`.

### Docker variant

- Docker enters only via the `fullpageos-docker` module appended by explicit
  variants (D2/D5). The default `MODULES` never includes it; never bolt
  Docker into the existing `fullpageos` chroot script. Standard images must
  contain no Docker Engine, containerd, Compose plugin, Docker apt source, or
  daemon config.
- Install from Docker's official Debian repo: keyring under
  `/etc/apt/keyrings`, `Signed-By` source with explicit `arch=` and the
  codename validated against `/etc/os-release` (fail loudly on mismatch).
  Packages: `docker-ce docker-ce-cli containerd.io docker-buildx-plugin
  docker-compose-plugin`. Support an optional version pin
  (`FULLPAGEOS_DOCKER_VERSION`); log resolved versions into build metadata.
  Never install legacy Python `docker-compose`; verify `docker compose
  version`.
- Enable `docker.service` + `containerd.service` only in Docker variants.
- Logging: `/etc/docker/daemon.json` with the `local` driver, `max-size=10m`,
  `max-file=3`, compression if supported — merged with any existing config
  via jq, kept valid JSON. Limits apply to newly created containers.
- The kiosk user is never added to the `docker` group by default (membership
  is root-equivalent — document that).
- Kernel/boot: don't blindly add historical flags (e.g. cgroup-v1 args).
  Verify cgroup v2 + controllers, namespaces, OverlayFS, bridge/veth,
  br_netfilter, forwarding, and nftables/iptables-nft with runtime checks;
  change cmdline/module config only when a failing test proves the need. No
  custom kernel without explicit user approval.
- `armhf` Docker is docs-claimed but unproven — never present it as
  supported until CI/hardware verifies it; record limitations honestly.

### Kiosk readiness (FPOS-016)

- Known bug: `run_onepageos:9` — `check_for_httpd=disabled` bypasses the URL
  check entirely. The generic wait-for-URL feature must work without Docker,
  use bounded curl timeouts with backoff (no busy-loop), quote URLs safely
  (no `eval`), document the ready HTTP-code set (today 2xx/3xx/401), log
  without flooding the journal, never spawn duplicate Chromium processes, and
  behave exactly like legacy FullPageOS when disabled. systemd ordering alone
  is not readiness — a started container is not a ready web app.

### Docs & examples

- The Compose v2 example is documentation-only: placeholder `arm64` image,
  `restart: unless-stopped`, named volume, healthcheck, loopback-only port
  publishing — never auto-started, never the user's real service.
- `docker login` is a documented post-install interactive step; explain where
  credentials are stored and recommend a credential helper. Docker-published
  ports can bypass firewall frontends — keep examples loopback-bound.

## Security & repository safety

- No credentials, tokens, or real registry names anywhere — source, build
  args, env files, boot partition, image layers, examples, CI, artifacts, or
  shell history baked into the image. Add secret checks where practical, but
  never print discovered secret values.
- Never weaken SSH, sudo, firewall, or file permissions to make a test pass.
- No default-password regressions; update stale docs that claim insecure
  defaults when the migration touches them.
- Keep changes minimal and consistent with repository conventions.
