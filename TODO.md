# FullPageOS arm64 and optional Docker work

## Goal

Modernize FullPageOS to produce clearly named 32-bit (`armhf`) and 64-bit (`arm64`)
images based on Raspberry Pi OS Lite (Debian 13 Trixie) for Raspberry Pi 3/4/5,
keep the standard image lightweight and Docker-free, and offer a genuinely
optional Docker variant (Docker Engine + Buildx + Compose v2 from Docker's
official Debian apt repository) with verified kernel/runtime capabilities,
bounded logging, a generic kiosk URL-readiness feature, and no embedded
application stack or registry credentials.

## Confirmed decisions

From the mission brief (fixed):
- Primary target: Raspberry Pi 3/4/5, Raspberry Pi OS Lite 64-bit, Debian 13 Trixie, `arm64`.
- Secondary target: 32-bit Raspberry Pi OS Lite on Pi 3/4/5; Docker on `armhf` only if verified.
- Pi 1 / Pi 2 / original Zero / Zero W: out of scope.
- Standard images contain no Docker Engine, containerd, Compose plugin, Docker apt source, or daemon config.
- Docker install: official Docker Debian repo, keyring under `/etc/apt/keyrings`, `Signed-By`, no `curl | sh`, no `apt-key`, packages `docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`.
- Docker logging: `local` driver, `max-size=10m`, `max-file=3`, compression if supported, via valid `/etc/docker/daemon.json` (merge, don't clobber).
- Kiosk user is NOT added to the `docker` group by default.
- No registry credentials anywhere; `docker login` is a documented post-install step.
- User's containerized service stays outside this repo; only a placeholder Compose example in docs.

From discovery (2026-08-31), decided unless overridden:
- D1: Build via the existing CustomPiOS mechanism. CustomPiOS `devel` already supports
  `BASE_ARCH=arm64` (base module globs `*-raspios-*-arm64-*` images) and per-distro
  modules/variants. No custom build system.
- D2: Do NOT use upstream CustomPiOS's `docker` module: it installs legacy Python
  `docker-compose` and has known build failures (upstream issues #126, #139). Instead
  create a FullPageOS-local module `src/modules/fullpageos-docker` that follows Docker's
  official Debian instructions.
- D3: Keep the kiosk on X11 for this migration. All FullPageOS runtime scripts
  (xdotool, feh, x11vnc, startx via CustomPiOS `gui` module) are X11-based; Trixie still
  ships xserver-xorg and Chromium runs fine under X11. A labwc/Wayland rewrite is a
  separate future project, not part of this work. (Trixie *desktop* defaults to labwc,
  but we build from Lite and install our own X stack — unaffected.)
- D4: Chromium package on Trixie is `chromium` (`chromium-browser` is a transitional
  dummy). Install/reference `chromium` and keep a compatibility fallback in scripts.
- D5: Docker variants are expressed as CustomPiOS variants whose config appends the
  `fullpageos-docker` module to `MODULES`; the default build never includes it.
- D6: Pin mutable inputs: CI must pin the CustomPiOS ref and use versioned (dated)
  Raspberry Pi OS image URLs + sha256, overridable via config. No more
  `raspios_lite_armhf_latest` + unpinned CustomPiOS default branch.

Decided by user 2026-09-01 (former proposals P1–P3):
- D7 (P1, modified): Work stays on branch `dev64`, based on tag 0.14.0 (6e773cc).
  This fork is for the user's own use — merging into upstream `devel` is NOT a goal;
  do not rebase onto `devel`. (devel's 17 extra commits — e2e tests, Chromium
  translate policy — are intentionally not included.)
- D8 (P2, as proposed): Default (no-variant) build = **arm64 standard**; `armhf`,
  `docker` (arm64+Docker) and `armhf-docker` are variants.
- D9 (P3, as proposed): Artifact naming
  `FullPageOS-<debian_codename>-<arch>[-docker]-<version>.img.gz`
  (e.g. `FullPageOS-trixie-arm64-0.15.0.img.gz`), derived from the build's own
  arch/variant variables.

## Build matrix

| # | Artifact (CI name)                  | BASE_ARCH | Base image                        | Variant        | Docker | Status    |
|---|-------------------------------------|-----------|-----------------------------------|----------------|--------|-----------|
| 1 | fullpageos-trixie-arm64             | arm64     | RPi OS Lite arm64 (Trixie, dated) | (default)      | no     | planned   |
| 2 | fullpageos-trixie-armhf             | armv7l    | RPi OS Lite armhf (Trixie, dated) | armhf          | no     | planned   |
| 3 | fullpageos-trixie-arm64-docker      | arm64     | RPi OS Lite arm64 (Trixie, dated) | docker         | yes    | planned   |
| 4 | fullpageos-trixie-armhf-docker      | armv7l    | RPi OS Lite armhf (Trixie, dated) | armhf-docker   | yes    | gated on armhf Docker verification |

Docker's official Debian repo advertises trixie + both `arm64` and `armhf`
(verify actual armhf package availability during FPOS-014; docs claim ≠ tested).

## Tasks

- [x] FPOS-001: Repository and architecture audit (discovery)
  - Scope: map build entry points, module composition, config inheritance, base-image
    selection, overlays, chroot scripts, kiosk startup, systemd units, CI, artifact
    naming; identify hard-coded assumptions; assess build-environment privileges.
  - Acceptance criteria: findings + build matrix + task list recorded in TODO.md.
  - Expected verification: file inspection, graphify graph vs source cross-check,
    shellcheck/bash -n baseline. DONE — see work log.

- [x] FPOS-002: Branch base and repo hygiene
  - Scope: P1 decided → D7 (stay on `dev64`, 0.14.0 base, no upstream merge). Remaining:
    confirm `dev64` hygiene — untracked local files (`node_modules/`, `package.json`,
    lockfiles, `.idea`, `graphify-out/`, dotfiles) stay untouched and out of commits;
    decide whether AGENTS.md/CLAUDE.md/TODO.md get committed to dev64.
  - Acceptance criteria: on `dev64` (0.14.0 base per D7); `git status` shows only
    intended changes; no destructive git operations used.
  - Expected verification: `git branch --show-current`, `git log -1`, `git status`.
    DONE — see work log 2026-09-01.

- [x] FPOS-003: Configurable, pinned base-image selection — DONE, see work
  log 2026-09-01. (Note: the override variable is `BASE_ZIP_IMG`, not
  `ZIP_IMG` — old README wording was stale.)
  - Scope: add config knobs (e.g. in `src/config` + documented `config.local`
    overrides) for base image URL/filename + sha256 per arch; update `src/image/README`
    (currently says "Rasbian" and `*-raspbian.zip`); document `ZIP_IMG` override;
    default to a dated Trixie release, not `*_latest`.
  - Acceptance criteria: a builder can select armhf vs arm64 base image explicitly;
    defaults are pinned Trixie images; docs updated; no mutable-"latest" default left.
  - Expected verification: static review; grep for `raspios_lite_armhf_latest`,
    `raspbian` remnants; shellcheck on changed files.

- [x] FPOS-004: arm64 build configuration (default build) — DONE, see work
  log 2026-09-02.
  - Scope: implement P2 — default config targets `BASE_ARCH=arm64` with matching image
    glob; ensure `DIST_NAME`/version vars unchanged where possible.
  - Acceptance criteria: config resolves BASE_ARCH=arm64 by default; armhf achievable
    via variant (FPOS-005); nothing references armhf implicitly in default path.
  - Expected verification: source config in a bash harness and assert exported vars;
    shellcheck.

- [x] FPOS-005: 32-bit (armhf) variant preserved — DONE, see work log 2026-09-02.
  - Scope: add `src/variants/armhf/config` (BASE_ARCH=armv7l + armhf image pattern);
    update `.gitignore` (currently ignores `src/variants/*` except `no-acceleration`)
    to track new variants.
  - Acceptance criteria: `build_dist armhf` selects armhf base + arch; variant tracked
    by git; no-acceleration variant unaffected.
  - Expected verification: bash harness asserting variant config export; git ls-files.

- [x] FPOS-006: Trixie chroot-script compatibility — packages & apt hygiene —
  DONE, see work log 2026-09-02.
  - Scope: in `src/modules/fullpageos/start_chroot_script`: replace `chromium-browser`
    with `chromium` (+ transitional fallback); drop `--force-yes` (removed/deprecated in
    modern apt); review removed-package list (scratch, oracle-java8-jdk era) for Trixie
    relevance; replace ntp.conf append (Trixie uses systemd-timesyncd; `/etc/ntp.conf`
    edit is dead config) with timesyncd-appropriate handling or removal; install emoji
    font via `fonts-noto-color-emoji` package instead of build-time wget from GitHub.
  - Acceptance criteria: script contains no obsolete package names/flags; every
    installed package exists in Debian trixie (verified against packages.debian.org or
    apt in CI chroot); no network fetches of unpinned artifacts at build time.
  - Expected verification: shellcheck + bash -n; package-name existence check; CI build
    (FPOS-010) is the end-to-end gate.

- [x] FPOS-007: Trixie chroot-script compatibility — users, paths, boot config —
  DONE, see work log 2026-09-02.
  - Scope: replace hard-coded `pi` with `${BASE_USER}` (usermod, chown, sudo -u,
    /home/pi, chpasswd) throughout module scripts; fix `[ "${BASE_BOARD}" == raspberrypi* ]`
    glob bug (SC2081 — the cmdline.txt splash edit silently never runs today); confirm
    all boot-partition writes use `${BASE_BOOT_MOUNT_PATH}`; runtime scripts keep
    /boot/firmware (correct for Bookworm+).
  - Acceptance criteria: `grep -rn '\bpi\b'` in module shows only justified hits
    (docs/back-compat); glob test fixed with `[[ ]]` or `case`; shellcheck clean for
    changed lines.
  - Expected verification: shellcheck; targeted grep assertions.

- [x] FPOS-008: Kiosk startup on Trixie/X11 verified in design — DONE for
  everything provable from here (scripts + mechanism + package availability),
  see work log 2026-09-02; line-level gui-module proof split into FPOS-008b.
  - Scope: verify CustomPiOS `gui` module (devel) against Trixie Lite: lightdm/xserver
    package names, autologin, `GUI_STARTUP_SCRIPT` substitution, `BASE_USER` handling
    (older gui module hard-coded /home/pi — check devel); adjust FullPageOS scripts
    (`run_onepageos`, `start_chromium_browser`, refresh/fullscreen helpers) for the
    `chromium` binary/class name where needed (`xdotool search --class chromium`,
    `killall /usr/lib/chromium/chromium` path check in `reload_fullpageos_txt`).
  - Acceptance criteria: documented verification of gui-module/Trixie fit (file-level
    evidence, not assumption); FullPageOS scripts reference correct binary/paths;
    upstream CustomPiOS gaps recorded as blockers if found.
  - Expected verification: inspection of pinned CustomPiOS ref (requires GitHub access
    — see Blockers B2; CI fallback available); shellcheck.

- [!] FPOS-008b: gui-module line-level verification (split from FPOS-008;
  blocked on B2)
  - Scope: line-verify, on the CustomPiOS ref pinned in FPOS-010: the gui
    module's package-install list (lightdm/xserver-xorg names), lightdm
    autologin config (BASE_USER vs hard-coded pi/`/home/pi`), and the
    GUI_INCLUDE_ACCELERATION default/mechanism.
  - How to unblock (either): (a) user grants GitHub egress
    (github.com/raw.githubusercontent.com) so the pinned ref can be read
    directly; or (b) first FPOS-010 CI run — build_dist runs `bash -x`, so
    the log shows the gui module's apt-get lines and autologin sed commands;
    read them from the CI log and record here.
  - Acceptance criteria: each of the three facts confirmed with file/log
    lines quoted in the work log; any hard-coded `pi` in the gui module
    recorded as an upstream gap + workaround decided.

- [x] FPOS-009: Architecture-specific artifact naming — DONE, see work log
  2026-09-02. New `src/get_image_name` derives the D9 name from config chain
  + built-image filename with a mismatch abort; CI uses it.
  - Scope: implement P3 naming in CI copy/zip step and any release tooling; ensure name
    derives from actual build config (BASE_ARCH + variant), not hard-coded strings.
  - Acceptance criteria: an armhf build physically cannot produce an artifact named
    arm64 (name comes from the build's own arch variable); names include codename,
    arch, optional `-docker`, version.
  - Expected verification: YAML review; CI dry run in FPOS-010.

- [ ] FPOS-010: CI matrix for 32-bit and 64-bit standard images
  - Scope: rework `.github/workflows/main.yml`: actions/checkout@v4, pinned CustomPiOS
    ref (D6), pinned dated base-image URLs (+sha256 check), matrix {arm64, armhf},
    artifact naming from FPOS-009, qemu/binfmt setup appropriate per arch
    (qemu-aarch64-static for arm64).
  - Acceptance criteria: CI builds both standard images successfully on GitHub runners;
    artifacts named per matrix; workflow has no mutable inputs.
  - Expected verification: workflow YAML validation locally; actual run on GitHub
    (user-triggered or push to feature branch) — result recorded here.

- [ ] FPOS-011: Image verification step in CI (standard images)
  - Scope: post-build CI step that loop-mounts the image and asserts: rootfs
    `dpkg --print-architecture` matches matrix arch; chromium present; expected
    services enabled; **no** docker-ce/containerd/compose/docker apt source/daemon.json;
    no secrets (targeted grep, values never printed).
  - Acceptance criteria: assertions codified in a script under version control and
    green in CI for both arches.
  - Expected verification: CI run output.

- [ ] FPOS-012: Docker kernel/runtime capability check script
  - Scope: add a standalone script (shipped in Docker variant, also usable manually)
    checking: 64-bit, cgroup v2 mounted + cpu/memory/pids/cpuset controllers,
    namespaces (pid/mnt/net/ipc/uts/user), overlayfs, bridge + veth, br_netfilter
    availability, ip forwarding, nftables/iptables-nft, and reporting PASS/FAIL per
    item without changing system state. Runtime checks, not filename guesses.
  - Acceptance criteria: shellcheck-clean; safe read-only behavior; documented; runs on
    non-Docker systems with meaningful output (used at hardware gate HW-2).
  - Expected verification: shellcheck; run inside this aarch64 sandbox container
    (partial signal only — documented as such); real proof at HW-2.

- [ ] FPOS-013: `fullpageos-docker` module skeleton + variant wiring
  - Scope: create `src/modules/fullpageos-docker/{config,start_chroot_script}`; add
    variants `docker` (arm64) and `armhf-docker` (gated) whose configs append the
    module to MODULES (D5); default build untouched.
  - Acceptance criteria: default `MODULES` string has no docker module; variant configs
    add it; module config exposes `FULLPAGEOS_DOCKER_*` knobs incl. optional version
    pin; structure matches CustomPiOS module conventions (config var prefix = module
    name).
  - Expected verification: bash harness asserting MODULES composition per variant;
    shellcheck.

- [ ] FPOS-014: Docker apt repo + package installation (arm64 first)
  - Scope: in module: install prerequisites; keyring to
    `/etc/apt/keyrings/docker.asc` (or .gpg) with correct perms; deb822 or one-line
    `Signed-By` source with explicit `arch=` matching BASE_ARCH and codename validated
    from `/etc/os-release` (fail build on mismatch); install the 5 official packages;
    optional `FULLPAGEOS_DOCKER_VERSION` pin; log resolved package versions into build
    output/metadata; no `apt-key`, no convenience script. armhf path implemented but
    marked experimental until verified in CI.
  - Acceptance criteria: chroot build installs current stable Docker from official repo
    on arm64; resolved versions recorded; codename/arch validation fails loudly on
    mismatch; `docker compose version` works (verified in CI/HW); legacy
    `docker-compose` absent.
  - Expected verification: CI Docker-variant build (FPOS-017) + image inspection.

- [ ] FPOS-015: Boot enablement + bounded logging in Docker variant
  - Scope: enable `docker.service` + `containerd.service` in variant only; write/merge
    `/etc/docker/daemon.json` with `log-driver=local`, `log-opts {max-size:10m,
    max-file:3, compress if supported}` using jq-based merge if file exists; validate
    JSON in build; kiosk user NOT in docker group.
  - Acceptance criteria: services enabled only in Docker images; daemon.json valid JSON
    with exactly the intended keys merged; verified via `docker info` at HW-2; docs note
    limits apply to newly created containers.
  - Expected verification: jq validation in chroot; CI image inspection (enabled
    symlinks, daemon.json content); HW-2 for `docker info`.

- [ ] FPOS-016: Generic kiosk URL-readiness feature
  - Scope: rework `run_onepageos` (and config plumbing): a boot-partition config
    (FullPageOS convention, e.g. `/boot/firmware/fullpageos-wait-for-url` or extension
    of existing files — final name decided in-task) enabling wait-for-URL independent
    of Lighttpd; fix the current bypass where `check_for_httpd=disabled` skips the URL
    check entirely (run_onepageos:9); bounded curl timeouts, documented ready-code set
    (currently 2xx/3xx/401), sleep/backoff (no busy-loop), safe URL quoting (no eval),
    journal-friendly periodic logging, no duplicate Chromium on container restarts;
    default behavior unchanged when disabled; works without Docker.
  - Acceptance criteria: with feature enabled Chromium never starts before URL ready;
    disabled ⇒ byte-for-byte-equivalent legacy behavior; shellcheck clean; unit-style
    test script exercising the wait loop against a local dummy server passes in sandbox.
  - Expected verification: local test harness (python3 http.server), shellcheck.

- [ ] FPOS-017: CI jobs for Docker variants + Docker-specific image verification
  - Scope: extend CI matrix with `docker` (and experimental `armhf-docker`) builds;
    inspection asserts docker packages present, services enabled, daemon.json valid,
    correct repo definition, no credentials; record resolved Docker versions as build
    metadata artifact.
  - Acceptance criteria: arm64 Docker image green in CI with all assertions;
    armhf-docker either green (→ matrix row 4 unlocked) or failure documented honestly
    and row 4 marked blocked/unsupported.
  - Expected verification: CI run output.

- [ ] FPOS-018: Docker runtime smoke tests (emulated/CI where possible)
  - Scope: scripted smoke test (target: run on hardware; where CI permits, run in
    systemd-nspawn/QEMU with documented limits): docker version/info, compose version,
    hello-world-class container for target arch, network, named volume surviving
    container replacement, overlay2 storage, restart policy after daemon restart,
    log rotation bound honored.
  - Acceptance criteria: script exists + shellcheck clean; each check reports
    PASS/FAIL; documentation states exactly what emulation does/doesn't prove.
  - Expected verification: script dry-run locally; full run at HW-2.

- [ ] FPOS-019: Operator documentation
  - Scope: README/docs updates: build instructions per matrix row; Docker variant docs;
    post-install `docker login` procedure + credential storage/security implications +
    credential-helper recommendation; docker-group=root-equivalent warning;
    non-deployed Compose v2 example (arm64 placeholder image, restart:
    unless-stopped, named volume, healthcheck, loopback publishing, optional service
    log bounds); URL-readiness feature docs; armhf Docker limitations; SD-card/logging
    notes; update stale README claims (default password/user story, "Rasbian",
    Raspberry Pi 2 claim vs new support matrix).
  - Acceptance criteria: docs cover build/install/verify/operate/troubleshoot for both
    image types; no credentials or real registries embedded; examples never
    auto-start.
  - Expected verification: doc review; secret-scan grep; rst/md lint if available.

- [ ] FPOS-020: Static-check harness in repo + CI
  - Scope: add a `tests/` (or repo-convention path) script running shellcheck, bash -n,
    jq JSON validation, workflow YAML parse, MODULES-composition assertions, and
    no-secrets grep; wire into CI as a fast job; fix or explicitly waive the 7 baseline
    shellcheck findings in touched files.
  - Acceptance criteria: one command runs all static checks; green locally and in CI.
  - Expected verification: local run output recorded here.

- [ ] FPOS-021: Final review — security, regression, matrix sign-off
  - Scope: end-to-end review against "Final definition of done": no credential leaks,
    no weakened permissions, standard image Docker-free proof, naming audit, TODO
    evidence trail complete, docs accurate.
  - Acceptance criteria: every DoD bullet has linked evidence (CI run, HW gate result,
    or file); open risks listed.
  - Expected verification: checklist walkthrough recorded in work log.

## Hardware test gates

- HW-1 (after FPOS-010/011): First Trixie arm64 standard image boots on Pi 3, 4, and 5;
  kiosk launches Chromium full-screen to default dashboard; x11vnc + ssh reachable.
  I will prepare exact test instructions; user reports results before gate closes.
- HW-2 (after FPOS-017/018): arm64 Docker image on Pi 4/5: capability-check script all
  PASS; docker run + compose up of a test stack; container networking; named volume
  survives replacement; restart policy survives reboot; `docker info` shows local log
  driver; log bound honored.
- HW-3 (before FPOS-021): Final validation pass on representative Pi 3, Pi 4, Pi 5
  (standard arm64 + armhf), plus armhf-docker only if matrix row 4 unlocked.
- HW gates require user-run hardware; each gets a written procedure + expected-results
  sheet when reached.

## Blockers

- B1 (environment, permanent for this sandbox): No root/sudo, no usable loop devices,
  no qemu-user-static/binfmt in this container ⇒ **actual image builds cannot run
  locally**. Mitigation: static checks + bash harnesses locally; real builds and image
  inspection in GitHub Actions; boot/Docker runtime proof at HW gates. (Container is
  itself aarch64, which helps for arm64-targeted script sanity runs.)
- B2 (environment, current): github.com / raw.githubusercontent.com / docs.docker.com
  unreachable from sandbox (proxy 502/refused); WebSearch works. 2026-09-02:
  packages.debian.org and cdn.jsdelivr.net (GitHub CDN mirror) also blocked
  (sandbox allow-list denials) — Debian package facts and upstream file
  contents must come via WebSearch snippets or CI logs. Impact: cannot clone
  CustomPiOS locally to pin a ref and line-verify the gui/base modules (FPOS-008, D6).
  Mitigation: verified key facts via search (BASE_ARCH=arm64 support, module system,
  docker-module deficiencies); ask user to allow GitHub egress OR verify module
  internals via CI logs. Not blocking planning; blocks parts of FPOS-008 evidence.
- B3 — RESOLVED 2026-09-01: `dev64` was found to be branched from 6e773cc (tag
  0.14.0), not `devel`. User decided this is intentional (personal fork, no upstream
  merge) → D7. FPOS-002 unblocked.

## Work log

### 2026-09-02 — FPOS-009: architecture-specific artifact naming — COMPLETE

Summary: new versioned script `src/get_image_name` prints the D9 artifact
base name `FullPageOS-<codename>-<arch>[-docker]-<version>` (artifact =
`<name>.img.gz`). It derives every variable part from the build itself and
cross-checks two independent sources, failing loudly on disagreement:
(1) config chain sourced in build order (`src/config` → `config.local` if
present → `src/variants/<variant>/config`) gives BASE_ARCH (mapped
arm64/aarch64→arm64, armv7l/armhf→armhf), DIST_VERSION, and the `-docker`
suffix (iff MODULES contains `fullpageos-docker`, D5 — so future docker
variants name themselves with zero CI changes); (2) the built image's own
filename (`…-raspios-<codename>-<arch>-lite.img`, found in the CustomPiOS
workspace dir — `workspace-<variant>` if present, else `workspace`; or via
explicit `FULLPAGEOS_IMAGE=<path>`) gives codename + arch. Arch mismatch
between (1) and (2) aborts ⇒ an armhf build physically cannot be labeled
arm64. `--image-path` mode prints the discovered image path so the CI copy
step copies exactly the file the name was derived from (no second glob to
drift). Prefix is literal `FullPageOS` per D9 (DIST_NAME keeps its
historical `FullpageOS` spelling for CustomPiOS-facing uses — not changed,
behavior risk). CI `main.yml`: `Copy Output`+`Zip Output`+upload replaced
by one `Name and Compress Output` step (id: artifact) calling the script,
gzip to `${NAME}.img.gz`, name exported via `$GITHUB_OUTPUT`; upload-
artifact name/path use `steps.artifact.outputs.name`. `build.img.gz` is
gone from the workflow.

Files changed: `src/get_image_name` (new, mode 755),
`.github/workflows/main.yml`, `TODO.md`.
Commands/tests run: `bash -n` + `shellcheck -s bash src/get_image_name` →
clean. 13-case TMPDIR harness (src tree copied, fake images) → 13/13 PASS:
default arm64 name; `--image-path`; armhf variant name; **armhf-variant +
arm64-image mismatch aborts** (core acceptance); config.local sourced and
mismatch still caught; docker-MODULES ⇒ `-docker` suffix
(FullPageOS-trixie-arm64-docker-0.14.0); variant without own workspace
falls back to `workspace/`; two images abort; zero images abort; unknown
variant aborts; FULLPAGEOS_IMAGE override (bookworm filename ⇒ bookworm
codename — name tracks the real base); unparseable glob-hit and
FULLPAGEOS_IMAGE names abort. Workflow YAML parsed with yq (9 steps);
extracted inline CI script shellcheck-clean. `git check-ignore` → script
not ignored; `git status` shows only intended files.
NOT verified (B1/B2): a real CI run of the new step (first FPOS-010 run is
the gate) and CustomPiOS's `workspace-<variant>` dir convention (search-
recalled, line-verification blocked by B2) — mitigated by the fallback to
`workspace/` plus loud failure if no image is found either way.

Remaining risks: if CustomPiOS renames the extracted base image or drops
the `-lite` suffix in a future base release, the glob/regex fails loudly
(fix is one pattern); rpi-imager metadata naming untouched (FPOS-019
scope); version stays 0.14.0 until a deliberate bump (D9 example shows
0.15.0 — bump is a release decision, not naming logic).

Next up: FPOS-010 (CI matrix for 32-bit and 64-bit standard images).

### 2026-09-02 — FPOS-008: kiosk startup on Trixie/X11 — COMPLETE (scripts+design); FPOS-008b split out

Script changes (runtime, module `fullpageos`):
- `start_chromium_browser`: browser binary resolved at runtime —
  `command -v chromium` ⇒ `chromium`, else `chromium-browser` (D4 fallback);
  both invocation lines (live kiosk line and dead signage-mode example) use
  `"$BROWSER"`; `--app=$( … )` now quoted (SC2046 on touched lines fixed);
  signage-mode crash-refresh path corrected from nonexistent
  `/home/<uid1000>/scripts/refresh` to `/opt/custompios/scripts/refresh`.
- `reload_fullpageos_txt`: `killall /usr/lib/chromium/chromium` (the #602
  fix, kept as primary — local git shows c1d1e17 changed it from the old
  chromium-browser path, proving Debian-style layout on current RPi OS)
  with fallbacks `killall chromium` then `killall chromium-browser`.
- `fullscreen`/`refresh`/`safe_refresh`: NO change needed — xdotool search
  patterns are case-insensitive POSIX EREs per upstream docs/man
  (jordansissel/xdotool xdotool.pod: "Matches are case-insensitive"), so
  `--class chromium` matches Debian chromium's WM_CLASS "Chromium" exactly
  as it matched legacy "Chromium-browser". Recorded as verified design.

Design verification (gui module / Trixie fit — file-level where possible):
- Startup chain confirmed with repo-path evidence (FullPageOS discussion
  #547 + module tree): lightdm autologin → XDG session
  `usr/share/xsessions/guisession.desktop` (in gui module filesystem,
  root_init) with Exec → `/opt/custompios/scripts/start_gui` →
  gui `end_chroot_script` runs
  `sed -i "s@GUI_SESSION_PLACEHOLDEFR@${GUI_STARTUP_SCRIPT}@g" …/start_gui`
  ⇒ our `GUI_STARTUP_SCRIPT=/opt/custompios/scripts/run_onepageos` slots in
  unchanged.
- Trixie package availability: lightdm 1.32.0-6+b2, xserver-xorg 1:7.7+24,
  xserver-xorg-core 2:21.1.16-1.3 — all present in trixie (D3's X11 stack
  installable).
- NOT provable from sandbox (B2; packages.debian.org + cdn.jsdelivr.net now
  also blocked — B2 updated): gui module's exact apt package list, autologin
  user handling (BASE_USER vs hard-coded pi), GUI_INCLUDE_ACCELERATION
  default on the ref we'll pin. Split into FPOS-008b [!], unblockable via
  GitHub egress or the first FPOS-010 CI log (build runs bash -x).

Files changed: `…/scripts/start_chromium_browser`,
`…/scripts/reload_fullpageos_txt`, `TODO.md`.
Commands/tests run: `bash -n` OK both; shellcheck: reload_fullpageos_txt
clean; start_chromium_browser — all remaining findings (SC2317/SC2155/
SC2162/SC2006) sit in the pre-existing dead signage-mode block after
`exit;`, untouched (FPOS-020 scope); changed lines clean. PATH-stub
harness: with `chromium` present → chromium runs; without → fallback
chromium-browser runs. File mode of rewritten reload_fullpageos_txt
still 755. `git show c1d1e17` → documented the #602 path change.
NOT verified (B1): actual kiosk boot — that's HW-1.

Remaining risks: Chromium 139 tolerates the legacy flag set (unknown flags
warn, don't fail) but flag review is worth a pass at HW-1;
signage-mode block remains dead-by-design documentation code. FPOS-008b
open for the three gui-module facts.

Next up: FPOS-009 (architecture-specific artifact naming).

### 2026-09-02 — FPOS-007: Trixie chroot script — users, paths, boot config — COMPLETE

Summary: hard-coded `pi` replaced with quoted `${BASE_USER}` in all six
functional sites of `start_chroot_script` (usermod www-data group add,
FullPageDashboard chown, chpasswd, 4× `sudo -u` setX11vncPass); the stale
`#root password` comment corrected to `#user password` (it sets the user's
password). `BASE_BOARD` test fixed: `[ "${BASE_BOARD}" == raspberrypi* ]` →
`[[ … == raspberrypi* ]]` — harness proved the old form NEVER matched (even
for exact "raspberrypi", it compared against the literal string
"raspberrypi*"), so this REACTIVATES the quiet-splash cmdline.txt edit
(` logo.nologo consoleblank=0 loglevel=0 quiet` appended) on Raspberry Pi
builds — expect visibly quieter boot at HW-1. Dashboard INIT_URL_PATH sed
requoted (var inside one double-quoted string, behavior-identical) —
clears SC2027/SC2086 on that boot-path line. Boot-path audit: all chroot
boot-partition writes use `${BASE_BOOT_MOUNT_PATH}` (unpack, cmdline.txt,
config.txt, dashboard sed, check_for_httpd) ✓; runtime files correctly
hardcode /boot/firmware (get_url:4, run_onepageos:9,
splashscreen.service:7 — right for Bookworm+/Trixie) — no changes needed.
Module `config` splash comment fixed: pointed at nonexistent
`filesystem/home/pi/media/splash.png`; real location is
`filesystem/boot/splash.png` → shown from /boot/firmware/splash.png.

Files changed: `src/modules/fullpageos/start_chroot_script`,
`src/modules/fullpageos/config`, `TODO.md`.
Commands/tests run: glob harness old-vs-new over
raspberrypi/raspberrypi3/raspberrypi64/lepotato → old form F/F/F/F (bug
confirmed), new form T/T/T/F (fix + non-Pi correctly excluded); `bash -n`
OK on both changed files; shellcheck: SC2081 and SC2027 gone, remaining
only SC1091 (info — /common.sh, expected) + SC2086 ×2 on untouched
keyboard-override lines (FPOS-020 scope) ⇒ changed lines clean;
acceptance grep `\bpi\b` over the module → only the two commented-out
legacy rc.local lines (justified back-compat docs). NOT verified (B1):
chroot execution with real BASE_USER/BASE_BOARD values from CustomPiOS —
FPOS-010 CI is the gate.

Remaining risks: exact BASE_BOARD values on the pinned CustomPiOS ref
can't be line-verified from here (B2) — if a value doesn't start with
"raspberrypi", the cmdline edit stays off (fails safe, same as today's
broken behavior). The double chown of FullPageDashboard (BASE_USER then
immediately www-data — the second wins) is preserved upstream behavior;
candidate for cleanup in a later task, noted not changed. Reactivated
cmdline edit appends to cmdline.txt once per build — single-line file,
idempotent per image, but combined with `disable_splash=1` it changes boot
visuals vs 0.14.0 images; verify at HW-1.

Next up: FPOS-008 (kiosk startup on Trixie/X11 verified in design).

### 2026-09-02 — FPOS-006: Trixie chroot script — packages & apt hygiene — COMPLETE

Summary (`src/modules/fullpageos/start_chroot_script` only): all three
`--force-yes` dropped (tools line, chromium, x11vnc) → plain
`apt-get install -y`. Chromium block now installs `chromium` (D4) with an
`if ! … then chromium-browser` transitional fallback (set -e-safe, proven in
harness). Emoji font: build-time `sudo -u pi wget` of NotoColorEmoji.ttf from
GitHub main (unpinned network fetch into /home/pi) replaced by
`apt-get install -y fonts-noto-color-emoji` (system-wide, pinned via apt).
Legacy `remove_if_installed scratch…freepats` block deleted — base is pinned
RPi OS **Lite** Trixie which ships none of those (several no longer exist in
Debian at all); the final clean/autoremove step remains. ntp.conf appends
removed (Trixie Lite has no /etc/ntp.conf reader; systemd-timesyncd with
RPi OS defaults handles time sync — comment left in place).

Package existence in trixie — every name the script installs verified via
WebSearch-backed packages.debian.org results (direct fetch proxy-blocked,
B2 now also covers packages.debian.org): chromium 139.x arm64 ✓,
checkinstall 1.6.2+git20170426-5.1 ✓, libavahi-compat-libdnssd1 0.8-16 ✓
(NO t64 rename in trixie), fbi 2.14-1 ✓, pulseaudio 17.0 ✓, x11vnc
0.9.17-1 ✓, fonts-noto-color-emoji 2.051-0+deb13u1 ✓, php-cgi/php/
php-common/lighttpd ✓, xdotool 3.20160805.1-5.1 ✓, feh ✓, xterm 398-1 ✓,
expect 5.45.4-4 ✓, screen 4.9.1 ✓, avahi-daemon 0.8-16 ✓, git/vim ✓
(arm64+armhf where checked).

Files changed: `src/modules/fullpageos/start_chroot_script`, `TODO.md`.
Commands/tests run: `bash -n` → OK; `grep -c force-yes` → 0; obsolete-token
grep (ntp.conf/wget/NotoColorEmoji/oracle-java8/scratch/…) → only the new
timesyncd comment matches; stub-apt-get harness → PASS both paths (chromium
fails ⇒ fallback installs chromium-browser without tripping set -e;
chromium succeeds ⇒ no fallback). shellcheck: 5 finding blocks remain, all
pre-existing on untouched lines — SC1091 info (/common.sh not followed —
expected for a CustomPiOS fragment), SC2081 (BASE_BOARD glob bug —
explicitly FPOS-007 scope), SC2027+SC2086 (sed quoting in dashboard config
+ keyboard overrides — FPOS-007/020). The old block's SC2046 findings are
gone; no new findings introduced.
NOT verified (B1): actually running the script in a chroot — first real
proof is the FPOS-010 CI build.

Remaining risks: Trixie php 8.4 php-cgi under lighttpd sets
"no new privileges" (broke RaspAP, which moved to php-fpm) — FullPageDashboard
only reads/writes files so likely unaffected, but if the dashboard
misbehaves in CI/HW-1, switch to php-fpm (note for FPOS-008/019).
`pulseaudio` kept per 0.14.0 commit 6e773cc, but RPi OS Bookworm+ defaults
to PipeWire — audio stack choice may need revisiting at HW-1 if audio is
broken. `checkinstall` is legacy OctoPi heritage (unused at runtime) — kept
since it exists in trixie; candidate for pruning in FPOS-019/021 review.

Next up: FPOS-007 (Trixie chroot-script compatibility — users, paths, boot
config).

### 2026-09-02 — FPOS-005: 32-bit (armhf) variant preserved — COMPLETE

Summary: added `src/variants/armhf/config` — a one-setting variant
(`export BASE_ARCH=armv7l`, D8 naming) with a header comment pointing at the
pinned `FULLPAGEOS_IMAGE_URL_ARMHF` image and the `./build_dist armhf`
invocation. `.gitignore` gained `!src/variants/armhf` next to the existing
`!src/variants/no-acceleration` (same proven re-include pattern; all other
variant dirs stay ignored). `export` used deliberately (no-acceleration uses
a plain assignment, but BASE_ARCH must reach CustomPiOS child processes and
this stays correct even if src/config ever stops exporting it).

Files changed: `src/variants/armhf/config` (new), `.gitignore`, `TODO.md`.
Commands/tests run: env -i harness sourcing `src/config` →
`src/variants/armhf/config` (simulating CustomPiOS dist→variant order) →
PASS: BASE_ARCH=armv7l and visible in a child process (export verified);
DIST_NAME/DIST_VERSION/MODULES byte-identical, no docker module. Sourcing
`no-acceleration/config` on top → PASS: GUI_INCLUDE_ACCELERATION=no,
variant unaffected/composable. `bash -n` + `shellcheck -s bash` on the new
config → clean. `git check-ignore`: armhf and no-acceleration
tracked-eligible, hypothetical `src/variants/somefuture/config` still
IGNORED. `git status` shows only `.gitignore` + the new variant dir.
NOT verified (B1/B2): a real `./build_dist armhf` run (CustomPiOS variant
sourcing + armhf base-image selection end-to-end) — proof lands with the
FPOS-010 CI matrix row.

Remaining risks: whether CustomPiOS's armv7l image glob excludes arm64
images when both sit in src/image is unverified (B2) — mitigated by the
"keep only one image" rule in src/image/README plus the `BASE_ZIP_IMG`
exact-file override, and CI downloads exactly one image per job. Real
variant-sourcing order is CustomPiOS's; simulated locally only.

Next up: FPOS-006 (Trixie chroot-script compatibility — packages & apt
hygiene).

### 2026-09-02 — FPOS-004: arm64 build configuration (default build) — COMPLETE

Summary: `src/config` now exports `BASE_ARCH=arm64` (D8) with a comment that
the 32-bit build is the upcoming `armhf` variant which overrides it —
CustomPiOS's base module derives the `*-raspios-*-arm64-*` image glob from
BASE_ARCH (verified during FPOS-001 discovery via WebSearch; direct GitHub
line-verification still blocked, B2 — proxy 502 re-confirmed today).
`DIST_NAME`/`DIST_VERSION`/`MODULES`/`GUI_STARTUP_SCRIPT` unchanged. Default
path de-armhf'd: CI download step now fetches/verifies
`FULLPAGEOS_IMAGE_URL_ARM64` (+_SHA256_ARM64) — CI's existing
`qemu-user-static` dep already provides qemu-aarch64-static, and the copy
glob `*-raspios-*-lite.img` still matches
`2025-12-04-raspios-trixie-arm64-lite.img`; README.rst build snippet flipped
to the ARM64 vars (armhf documented as the 32-bit alternative) and the
requirement line now says qemu-user-static (aarch64 for default 64-bit,
arm for 32-bit); `src/image/README` glob sentence updated to mention
BASE_ARCH-driven arch-specific matching. Full CI matrix remains FPOS-010.

Files changed: `src/config`, `.github/workflows/main.yml`, `README.rst`,
`src/image/README`, `TODO.md`.
Commands/tests run: bash harness 1 (env -i, source src/config) → PASS:
BASE_ARCH=arm64 exported; DIST_NAME=FullpageOS, DIST_VERSION=0.14.0, MODULES
string and GUI_STARTUP_SCRIPT byte-identical; no docker module in MODULES.
Harness 2 (variant-style `BASE_ARCH=armv7l` assignment after sourcing
src/config) → PASS — override works in bash sourcing order (partial signal:
real dist-config→variant-config order lives in CustomPiOS, exercised for
real in FPOS-005/010). `bash -n` + `shellcheck -s bash src/config` → clean.
Workflow parsed with yq (10 steps intact); extracted download-step script
shellcheck → only SC1091 info (sourced file not followed — expected, same
as FPOS-003). `git grep -niE 'armhf|armv7' src/config
.github/workflows/main.yml src/build_dist` → only comments and the pinned
`FULLPAGEOS_*_ARMHF` variable definitions (kept deliberately for the armhf
variant + docs); no functional armhf use left in the default path.
NOT verified (B1/B2): an actual CustomPiOS run consuming BASE_ARCH=arm64
end-to-end — that proof lands with the FPOS-010 CI build (and the chroot
script still has known Trixie issues, FPOS-006/007, so a full CI build may
fail beyond the image-selection stage).

Remaining risks: CustomPiOS BASE_ARCH/glob behavior is search-verified, not
line-verified (B2) — first FPOS-010 CI run is the hard gate; `aarch64` is
also accepted by CustomPiOS but `arm64` is used consistently here.

Next up: FPOS-005 (32-bit armhf variant preserved).

### 2026-09-01 — FPOS-003: Configurable, pinned base-image selection — COMPLETE

Summary: `src/config` now exports pinned per-arch base-image knobs
(`FULLPAGEOS_IMAGE_URL_ARM64/_ARMHF` + `FULLPAGEOS_IMAGE_SHA256_ARM64/_ARMHF`),
defaulting to the dated Raspberry Pi OS Lite Trixie release **2025-12-04**
(.img.xz URLs on downloads.raspberrypi.com + sha256 of the compressed file).
`src/image/README` rewritten (was "Rasbian"/`*-raspbian.zip`): download+verify
snippet, arch selection, `BASE_ZIP_IMG` exact-file override, no-`*_latest`
rule. README.rst: requirement line and build snippet now use the pinned vars
with a `sha256sum -c` step; the config.local paragraph fixed (`ZIP_IMG`→
`BASE_ZIP_IMG` — verified real variable name in CustomPiOS devel base module;
also removed leftover upstream "OctoPi" mention; default glob is
`*-{raspbian,raspios}*.{zip,7z,xz}`, newest wins → docs say keep only one
image). CI `main.yml`: "Download Raspbian Image" step (mutable
`raspios_lite_armhf_latest`) replaced by pinned URL + `sha256sum -c` sourced
from src/config — armhf (current default build) until the FPOS-010 matrix;
extracted `2025-12-04-raspios-trixie-armhf-lite.img` still matches the
existing `*-raspios-*-lite.img` copy glob. Full CI rework remains FPOS-010.

Evidence for pinned values (GitHub/downloads.raspberrypi.com egress blocked —
B2 — so verified via WebSearch): arm64 sha256
`681a775e…abbe6290` confirmed by 3 independent sources (geekfarm build notes,
rpi-imager debug log in raspberrypi/rpi-imager#1385 incl. official metadata
`release_date 2025-12-04`, Kuketz forum quoting raspberrypi.com). armhf sha256
`1b3e49b6…401a49ff` is **single-source** (geekfarm blog) — acceptable because
CI's `sha256sum -c` fails loudly on mismatch at first download; recheck at
FPOS-010's first CI run against the official `<image-url>.sha256`. Newer
dated releases exist (e.g. 2026-04-13 Trixie); staying on 2025-12-04 because
its checksums are verifiable from here — bumping the pin is a two-line change.
CustomPiOS override semantics verified via search of guysoft/CustomPiOS devel
(`src/modules/base/config`, `src/custompios`): `BASE_ZIP_IMG` honored if
pre-set (config.local/env), may point at `.img` directly; else newest match in
`BASE_IMAGE_PATH` (default `${DIST_PATH}/image`) wins.

Files changed: `src/config`, `src/image/README`, `README.rst`,
`.github/workflows/main.yml`, `TODO.md`.
Commands/tests run: bash harness sourcing src/config → PASS (all 4 vars
exported, URLs dated-Trixie-img.xz, sha256 64-hex, no "latest"); `bash -n` +
`shellcheck -s bash src/config` → clean; snippet mechanics test (dummy file,
`echo "<sha>  <file>" | sha256sum -c`) → OK; workflow YAML parsed with yq
(step list intact); shellcheck on the new inline CI script → clean after
SC2164 fix (`cd … || exit 1`; SC1091 info = sourced file not followed,
expected); `git grep -iE 'raspios_lite_armhf_latest|_latest|raspbian|rasbian|ZIP_IMG'`
→ only prohibition comments, BASE_ZIP_IMG docs, general README.rst prose
(lines 9/69/72 — FPOS-019 scope) and a start_chroot_script comment (FPOS-006
scope). NOT verified (impossible locally, B1/B2): actually downloading the
images; end-to-end CI build (current chroot script still has Trixie
incompatibilities — FPOS-006/007 — so CI may fail later in the build; the
download+verify step itself is self-contained).

Remaining risks: armhf sha256 single-source until first CI download; pinned
release will age (bump deliberately, never revert to `_latest`); CustomPiOS
ref itself still unpinned in CI (explicitly FPOS-010/D6 scope).

Next up: FPOS-004 (arm64 default build configuration).

### 2026-09-01 — FPOS-002: Branch base and repo hygiene — COMPLETE

Summary: D7 in effect — branch `dev64` (base 6e773cc, confirmed = tag 0.14.0 via
`git tag --points-at`), synced with origin/dev64 at 8194807. User committed and
pushed AGENTS.md + CLAUDE.md + TODO.md themselves (commit 8194807), resolving the
"commit the agent docs?" question. Local sandbox/editor dotfiles (.bashrc,
.bash_profile, .gitconfig, .profile, .zprofile, .zshrc, .ripgreprc, .idea, .vscode,
.claude/, .mcp.json, node_modules/) added to `.git/info/exclude` — per-clone local
config, NOT a repo change — so `git status` is now empty. The tracked `.gitignore`
was not touched (it already ignores graphify-out/ and variants except
no-acceleration). No files were modified or deleted; no destructive git ops.

Files changed: `.git/info/exclude` (local-only, untracked by design); TODO.md
(status + this entry).
Commands/tests run: `git branch --show-current` → dev64; `git log --oneline -3`;
`git status --short` → empty after exclude; `git branch -vv` → dev64 == origin/dev64;
`git tag --points-at 6e773cc` → 0.14.0.
Decisions: agent docs live in the repo on dev64 (user's commit 8194807).
Remaining risks: `.git/info/exclude` is per-clone — a fresh clone in a similar
sandbox will show the dotfiles again until re-added. Acceptable; documented here.

Next up: FPOS-003 (configurable, pinned base-image selection).

### 2026-09-01 — Process: mission brief codified into AGENTS.md (no FPOS item)

Summary: User supplied the full mission brief/working-method prompt. Updated
AGENTS.md (untracked, with CLAUDE.md → @AGENTS.md) to encode it: one atomic TODO
item per turn with start/end-of-turn protocol, task-state markers ([ ]/[~]/[x]/[!],
max one [~]), never commit unless explicitly asked, graphify-then-verify-against-
source rule, layered testing expectations, Docker-variant conventions (official
repo/keyrings/Signed-By/5 packages/version pin, services enabled only in variant,
bounded local logging via merged daemon.json, no docker group for kiosk user,
runtime capability checks over assumptions, no blind cgroup-v1 flags), kiosk
readiness constraints (check_for_httpd bypass, no eval, bounded timeouts, no
duplicate Chromium), docs/example rules (non-deployed Compose example, docker
login post-install), and security rules. Brief is consistent with existing
D1–D6/FPOS plan — no task-list changes needed.

Files changed: AGENTS.md (rewritten), TODO.md (this entry, decisions, B3, FPOS-002).
Commands run: git log/branch/merge-base — found `dev64` was branched from 6e773cc
(tag 0.14.0), not `devel`; asked user.
Decisions (user, same day): P1→D7 modified — stay on `dev64`/0.14.0, personal fork,
no upstream merge, no rebase onto devel; P2→D8 arm64 default approved; P3→D9 naming
scheme approved. B3 resolved. AGENTS.md branch rule updated to match D7.
Remaining risks: B1/B2 unchanged. D7 means upstream devel fixes (e2e, Chromium
translate policy) are not in this line; cherry-pick individually later if wanted.

### 2026-08-31 — FPOS-001: Repository and architecture audit (discovery) — COMPLETE

Summary: Mapped the whole build system via graphify graph (64 nodes; communities:
CI & Image Build Pipeline, Chroot Boot Script, Browser Startup, Kiosk Browser & URL
Config, Boot Splash, helper scripts) and verified every load-bearing claim against
source. No implementation files modified.

Architecture (verified in source):
- Entry point `src/build_dist` → external `${CUSTOM_PI_OS_PATH}/build_custom_os`;
  `src/custompios_path` is gitignored and generated by CustomPiOS's
  `update-custompios-paths`. CustomPiOS is NOT vendored (mutable dependency).
- `src/config`: `MODULES="base(raspicam, network, disable-services(gui(fullpageos), usage-statistics))"`,
  DIST_VERSION=0.14.0, enlarge/resize sizes, `GUI_STARTUP_SCRIPT=/opt/custompios/scripts/run_onepageos`,
  rpi-imager metadata. Config inheritance: `src/config.local` override (documented in
  README), variant configs under `src/variants/<name>/config` (only `no-acceleration`
  tracked: `GUI_INCLUDE_ACCELERATION=no`), module config
  `src/modules/fullpageos/config` (FULLPAGEOS_* defaults incl. INCLUDE_CHROMIUM/
  LIGHTTPD/DASHBOARD/WELCOME/X11VNC, OVERRIDE_PASSWORD default "raspberry"-era flow).
- Module `fullpageos`: overlays `filesystem/boot → ${BASE_BOOT_MOUNT_PATH}` (fullpageos.txt,
  fullpagedashboard.txt, splash.png), `filesystem/opt → /opt` (background.png + 8 scripts),
  `filesystem/root_init → /` (3 systemd units: splashscreen, x11vnc, clear_lighttpd_cache).
- Kiosk startup chain: CustomPiOS gui module (X11/lightdm; substitutes
  GUI_STARTUP_SCRIPT into start_gui) → `run_onepageos` loop → curl readiness check OR
  `check_for_httpd=disabled` bypass → `%BROWSER_START_SCRIPT%` (sed-replaced at build
  to `start_chromium_browser`) → `chromium-browser --kiosk --app=$(get_url)`.
- CI `.github/workflows/main.yml`: push-triggered single job; checkout@v2 (deprecated);
  CustomPiOS default branch unpinned; base image `raspios_lite_armhf_latest` (mutable,
  32-bit only); artifact fixed name `build.img.gz` (arch-blind); copies
  `src/workspace/*-raspios-*-lite.img`.

Hard-coded assumptions found (file:line evidence):
- armhf-only: CI wget `raspios_lite_armhf_latest` (main.yml:25); no BASE_ARCH anywhere
  in FullPageOS configs.
- Raspbian naming: `src/image/README` ("Rasbian", `*-raspbian.zip`), README.rst build
  docs.
- `pi` user hard-coded: start_chroot_script:53 (usermod www-data), :60 (chown
  FullPageDashboard), :111-112 (/home/pi/.fonts + sudo -u pi wget), :118 (chpasswd),
  :132,:136,:145,:149 (sudo -u pi setX11vncPass) — despite `${BASE_USER}` being used at
  :127, so the var is available.
- Default password: chpasswd flow + VNC password fallback "raspberry"
  (start_chroot_script:145); README documents pi/raspberry.
- `chromium-browser`: start_chroot_script:41, start_chromium_browser:17,:22; also
  `/usr/lib/chromium/chromium` path in reload_fullpageos_txt:2; xdotool class
  "chromium" in refresh/safe_refresh/fullscreen.
- X11-only stack: xdotool/feh/x11vnc/startx assumptions throughout; x11vnc.service
  requires display-manager.service.
- /boot vs /boot/firmware: chroot script correctly uses `${BASE_BOOT_MOUNT_PATH}`;
  runtime scripts hardcode /boot/firmware (OK for Bookworm+ incl. Trixie);
  splashscreen.service hardcodes /boot/firmware/splash.png (OK).
- Obsolete/broken bits: `apt-get --force-yes` (start_chroot_script:37,:41,:124);
  ntp.conf append (:170-171 — Trixie uses systemd-timesyncd; dead config); build-time
  wget of NotoColorEmoji from GitHub (unpinned, network-dependent, into /home/pi);
  glob bug `[ "${BASE_BOARD}" == raspberrypi* ]` (:22, SC2081) ⇒ quiet-splash
  cmdline.txt edit NEVER executes today; remove-list targets ancient Raspbian packages.
- Mutable deps: CustomPiOS unpinned in CI; base image "latest"; dashboard/welcome repos
  cloned from master at build time (FULLPAGEOS_*_REPO_* in module config — existing
  behavior, noting as reproducibility risk, not changing scope unilaterally).
- Readiness bypass confirmed: run_onepageos:9 — `grep -q disabled check_for_httpd`
  ORed with the curl check ⇒ Lighttpd-disabled builds launch Chromium with no URL
  check at all (mission's suspicion verified). Ready-code set today: 2xx/3xx/401.

External facts verified (WebSearch; GitHub direct fetch blocked — see B2):
- CustomPiOS devel: BASE_ARCH ∈ {armv7l (default), arm64, aarch64}; base module image
  glob switches to `*-raspios-*-arm64-*`; needs qemu-aarch64-static for arm64-on-x86;
  gui module = X11 + GUI_STARTUP_SCRIPT sed into start_gui (older revs hard-code
  /home/pi — must re-verify on pinned ref, FPOS-008).
- Upstream CustomPiOS `docker` module installs legacy Python docker-compose; issues
  #126/#139 document build failures ⇒ D2 (own module).
- Docker official Debian repo: trixie supported; arches amd64/arm64/armhf(/ppc64el);
  recommended package set matches mission's list. armhf availability still needs
  empirical verification (FPOS-014/017).
- RPi OS Trixie (released 2025-10-02): Debian 13, kernel 6.12, `chromium` package
  (`chromium-browser` transitional), desktop=labwc/Wayland (Lite unaffected → D3),
  deb822 apt sources, /boot/firmware (since Bookworm).

Environment assessment:
- Sandbox: uid 1000, no sudo, no /dev/loop*, no binfmt entries, no qemu-user-static ⇒
  cannot build images locally (B1). Host arch aarch64. github.com blocked via proxy
  (B2). Available: shellcheck, jq, python3, bash.
- Repo state: detached HEAD at 0.14.0 == 6e773cc; local `devel` 17 ahead (e2e tests,
  Chromium translate policy); origin/HEAD → devel. Untracked local-only files
  (node_modules, package.json, lockfiles, .idea, .vscode, dotfiles, graphify-out) —
  will not be committed or modified.

Files changed: TODO.md (created — this file). No implementation files touched.

Commands/tests run: git status/log/branch/ls-files; find; file reads of all 33 tracked
files' relevant subset; graphify query (existing graph, 64 nodes); losetup/binfmt/qemu/
sudo probes; clone + codeload + WebFetch attempts (blocked, B2); 4 WebSearch queries;
shellcheck baseline: 7 findings across module scripts (SC2046 ×4, SC2081 ×2, SC2027 ×2,
SC2288, SC2155); `bash -n` passes on all shell scripts.

Decisions: D1–D6 recorded above; P1–P3 proposed for approval.

Remaining risks: B1 (no local builds — CI/hardware are the real gates), B2 (can't
line-verify CustomPiOS modules from sandbox yet), armhf Docker support is
docs-claimed but unproven, upstream gui module Trixie fit unproven until FPOS-008/010,
dashboard/welcome build-time clones from mutable master remain a reproducibility risk
outside current scope.
