# Upstream port research

Research date: 2026-08-14

This note records primary-source evidence for the iOS 16.5.1-16.6.1,
mount, reboot, update-check, build, and version work. Commit links point to the
three repositories named in the request. No secondary write-up was used.

## Executive findings

- The selected P013onEr baseline is branch `v2.4.9.x` at
  `e890f579b2f27828ee0f549d3aa13ad8cdf49f87` (`2.4.9.25`). It already contains
  a roothide-adapted DarkSword implementation through the single-parent import
  commit `8da6ae490ce26ff40f1d42512d87596317ad5f37`.
- DarkSword advertises iOS `15.0` through `16.7.16` in the target branch and
  excludes A8. The app's end-to-end support string still limits arm64e to
  `16.5.1`; arm64 is advertised through `16.7.16`. The target history also
  contains `632286c`, which updated XPF specifically for iPhone X on 16.6.1,
  followed by newer XPF revisions. Therefore the requested `16.5.1-16.6.1`
  support is already represented for A9-A11 devices, but not for A12+ devices
  above `16.5.1`. Do not claim A12+ `16.6.1` support based only on DarkSword's
  framework plist.
- The w2599 source commits for the requested mount UI/backend and device reboot
  action are `eb6ac4d4a9ae0efe8a7d34a79bf616ed2ebd9d3f` and
  `e23613255ce9841eb93b10620a379ae2e1e82c37`. The current feature branch has
  equivalent cherry-picks `21645e0` and `7b7acd9`.
- A direct "disable update checks" implementation exists in w2599 commit
  `54328997d34dc194115bb48f28fad5e1e63938e5`, but it is bundled with an
  unrelated package-source change. Port only the two `return NO;` hunks from
  `Application/Dopamine/UI/DOUIManager.m`, or implement the same behavior
  cleanly; do not cherry-pick that whole commit.
- The baseline workflow does not build the triggering commit. Its checkout
  step clones `roothide/Dopamine2-roothide` default branch into the workspace.
  It must be changed to `actions/checkout` with recursive submodules (or an
  explicit checkout of the triggering SHA) before GitHub CI can validate a
  P013onEr feature branch.

## Repository and baseline map

| Role | Repository/ref | SHA | Evidence |
| --- | --- | --- | --- |
| Target | `P013onEr/Dopamine2-roothide`, `v2.4.9.x` | `e890f579b2f27828ee0f549d3aa13ad8cdf49f87` | [commit](https://github.com/P013onEr/Dopamine2-roothide/commit/e890f579b2f27828ee0f549d3aa13ad8cdf49f87), [branch](https://github.com/P013onEr/Dopamine2-roothide/tree/v2.4.9.x) |
| Official source | `opa334/Dopamine`, `wip/darksword-integration` | current remote head observed as `63b580d25b24cc23c88109247f3cb75831d2d68b` | [branch](https://github.com/opa334/Dopamine/tree/wip/darksword-integration) |
| Feature source | `w2599/Dopamine`, `rh2.4.9.x_modify_blacklist_ds` | current remote head observed as `d794d3d6c17980995052a0160ab47eeec1186bde` | [branch](https://github.com/w2599/Dopamine/tree/rh2.4.9.x_modify_blacklist_ds) |

The P013onEr baseline stores `2.4.9.25` in
`BaseBin/_external/basebin/.version`. Its Xcode Debug and Release settings both
use `MARKETING_VERSION = 2.4.9`. The baseline commit itself changes the basebin
version from `2.4.9.21` to `2.4.9.25`; it does not create a matching Git tag.

## DarkSword and iOS support evidence

### Official implementation commits

The official DarkSword integration first appears as:

- `9dc2267396ca8db3b3d9f88e6c525b8172ae9c0d`, "Add DarkSword exploit":
  [commit](https://github.com/opa334/Dopamine/commit/9dc2267396ca8db3b3d9f88e6c525b8172ae9c0d).
- `85aa77608c7e8b75ff972a7a77334de3ff95b054`, "DarkSword fixes (#776)":
  [commit](https://github.com/opa334/Dopamine/commit/85aa77608c7e8b75ff972a7a77334de3ff95b054),
  [pull request](https://github.com/opa334/Dopamine/pull/776).

The add commit touches 18 files. The portable unit is not just
`DarkSword.m`; it includes:

- `Application/Dopamine/Exploits/DarkSword/{DarkSword.h,DarkSword.m,Info.plist}`
- `Application/Dopamine.xcodeproj/project.pbxproj` for the framework target,
  embed phase, IOKit/IOSurface/libjailbreak linkage, and build dependency
- `Application/Dopamine/Jailbreak/DOJailbreaker.m` for
  `IOSurface_map_cleanup()`
- compatibility changes in the other exploit frontends
- `BaseBin/libjailbreak/src/{info.c,info.h,primitives_IOSurface.h,primitives_IOSurface.m,primitives_external.h}`

The fixes commit changes only `DarkSword.m`. It searches backward for the
corrupted filter instead of deriving the PCB start with an older-version-
incompatible mask, and bounds the physical scan so the free thread does not
map beyond the memory object. This fix is material for older iOS versions and
should accompany any DarkSword import.

The target did not preserve those upstream SHAs as ancestors. Instead,
`8da6ae490ce26ff40f1d42512d87596317ad5f37` is a single-parent import titled
"Merge remote-tracking branch 'upstream/wip/darksword-integration' into
2.4.3-merger": [target commit](https://github.com/P013onEr/Dopamine2-roothide/commit/8da6ae490ce26ff40f1d42512d87596317ad5f37).
Its imported `DarkSword.m` blob is identical to the fixed upstream blob
(`50b3764f74893af1c1d534bd531836e4fc4c026d`), so the target already has both
the implementation and #776's fix.

### What "16.5.1-16.6.1 support" means

The target's
`Application/Dopamine/Exploits/DarkSword/Info.plist` declares a supported
range of `15.0` through `16.7.16`, priority `890`, excluding A8. `DOExploit.m`
uses this metadata to decide whether an exploit flavor can be selected.

That metadata must be read together with
`Application/Dopamine/Jailbreak/DOEnvironmentManager.m`:

| Architecture | P013onEr `v2.4.9.x` advertised range | Result for 16.5.1-16.6.1 |
| --- | --- | --- |
| arm64e (A12 and newer) | iOS 15.0-16.5.1 | 16.5.1 yes; 16.6-16.6.1 no |
| arm64 (A9-A11; DarkSword excludes A8) | iOS 15.0-16.7.16 | 16.5.1-16.6.1 yes |

The official `opa334/2.x` README and `versionSupportString` similarly advertise
`15.0-16.5.1` for arm64e and `16.0-16.6.1` for arm64. Primary-source links:
[README](https://github.com/opa334/Dopamine/blob/2.x/README.md) and
[`DOEnvironmentManager.m`](https://github.com/opa334/Dopamine/blob/2.x/Application/Dopamine/Jailbreak/DOEnvironmentManager.m).

Consequently, no source reviewed here establishes a new arm64e kernel/PPL
chain for iOS 16.6-16.6.1. Changing only the UI string or DarkSword plist would
overstate support.

For A9-A11, target commit
[`632286c`](https://github.com/P013onEr/Dopamine2-roothide/commit/632286caddb3d6d8304da163d498ede97d25bc05)
is explicitly titled "fix patchfinding on iphoneX(ios16.6.1)" and advances the
XPF submodule to roothide/XPF commit
[`614544c`](https://github.com/roothide/XPF/commit/614544c5d1130e5b27192c6648edefe530b67423).
The selected `2.4.9.25` baseline uses a later, rewritten XPF line at
`3fb4bb3`, so reverting its submodule pointer to the old fix commit would be a
regression.

## Mount and reboot implementation

### Backend and UI commits

`eb6ac4d4a9ae0efe8a7d34a79bf616ed2ebd9d3f`, "jbctl_mount and unject":
[commit](https://github.com/w2599/Dopamine/commit/eb6ac4d4a9ae0efe8a7d34a79bf616ed2ebd9d3f).

- Adds `jbctl internal mount <path>` in `BaseBin/jbctl/src/internal.m`.
- Temporarily steals kernel credentials, performs a read-only `bindfs` mount,
  and restores the original credentials.
- Also adds unrelated process/input-method uninject behavior to
  `BaseBin/systemhook/src/common.c`. That file is not a dependency of mounting
  and should be excluded unless that behavior is explicitly desired.

`e23613255ce9841eb93b10620a379ae2e1e82c37`, "mountUI":
[commit](https://github.com/w2599/Dopamine/commit/e23613255ce9841eb93b10620a379ae2e1e82c37).

- Extends the backend with `initMountPath` and `internal unmount`. A snapshot
  of the original directory is stored below `JBROOT_PATH("/mnt")`, then bound
  read-only over the original path.
- Persists mount paths in `/var/mobile/newFakePath_RH.plist` and reapplies them
  from `DOJailbreaker.m` during jailbreak initialization.
- Adds mount/unmount management and translations in
  `Application/Dopamine/UI/Settings/DOSettingsController.m` and the Simplified
  Chinese localization files.
- Adds the requested whole-device reboot action in the same settings file.
  After confirmation it executes `JBROOT_PATH("/sbin/reboot")` through
  `exec_cmd_root`. This is distinct from Dopamine's pre-existing SpringBoard
  restart and userspace reboot actions.
- Also includes unrelated `DOMainViewController.m` branding placeholders and
  a developer-specific `jbupdate.sh` containing a LAN address. Neither is
  required for mount or reboot and should not be shipped.

The target feature branch currently contains equivalent cherry-picks
`21645e0eed5508d1f4ab920ed4a6a23c28f5bb14` and
`7b7acd94785647add7801a34caaf027b67bae16d`; their functional diffs match the
w2599 commits against this baseline.

### Dependencies and risks

- `e236132` depends on the `mount` command introduced by `eb6ac4d` and on the
  target's existing `jbclient_root_steal_ucred`, `JBROOT_PATH`, `exec_cmd`, and
  `exec_cmd_root` APIs.
- Both commands access `argv[1]` without an `argc` guard. Add argument
  validation when hardening the port.
- UI path standardization does not establish an allowlist. Root (`/`) is
  rejected only by `length > 1`; sensitive system directories remain
  selectable. The snapshot copy may also be large. A production port should
  constrain paths and report copy/mount failures.
- `UIAlertControllerStyleActionSheet` needs a popover anchor on iPad. The
  imported unmount UI does not configure one, so it can throw on iPad.
- The reboot button is rendered even when not jailbroken, while its handler
  resolves a binary under `JBROOT_PATH`. Gate it on a usable jailbreak or
  provide a deliberate non-jailbroken reboot implementation.
- Remove `jbupdate.sh`; besides being unrelated, it embeds
  `root@192.168.31.158` and uses `/var/jb`, which is inconsistent with this
  randomized-root roothide target.
- The two imported commits currently produce `git diff --check` warnings
  because of whitespace in their original code. Clean this before the final
  push.

## Disabling update checks

The target baseline still performs update checks in
`Application/Dopamine/UI/DOUIManager.m`:

- `isUpdateAvailable` fetches and compares the latest release tag.
- `getLatestReleases` synchronously reads the roothide GitHub releases API.
- `environmentUpdateAvailable` compares the running app and jailbroken
  environment versions.

w2599 commit `54328997d34dc194115bb48f28fad5e1e63938e5`
([source](https://github.com/w2599/Dopamine/commit/54328997d34dc194115bb48f28fad5e1e63938e5))
disables both update decisions by immediately returning `NO` from
`isUpdateAvailable` and `environmentUpdateAvailable`. It also modifies
`DOBootstrapper.m` to add a package repository; that second file is unrelated
and must not be ported for this request.

Returning `NO` at both decision points suppresses app and environment update
prompts. It leaves the release-fetching helpers in place for any explicit
changelog/update view callers. If "disable checking" means no GitHub request at
all, also audit callers of `getLatestReleases`, `getUpdatesInRange`, and
`launchedReleaseNeedsManualUpdate`; merely returning `NO` in the two methods
does not prove those helpers are unreachable.

The later w2599 commit `5ab1b46d6abf74929409a64a678e6fc571c03c87`
(`Update Check`) re-enables and replaces update discovery with w2599 release
asset parsing. It is contrary to the current requirement and should not be
ported.

## CI and release/version baseline

Primary source:
[`roothide.yml` at `e890f57`](https://github.com/P013onEr/Dopamine2-roothide/blob/e890f579b2f27828ee0f549d3aa13ad8cdf49f87/.github/workflows/roothide.yml).

The workflow triggers on `workflow_dispatch`, pull requests, and pushes, and
uses a macOS 14 runner. It installs Procursus tools, Theos with the iOS 16.5
SDK, trustcache, and libarchive; then runs `gmake` and uploads the TIPA.

The checkout step is incorrect for branch validation:

```yaml
- name: Checkout
  run: |
    git clone --recursive https://github.com/roothide/Dopamine2-roothide ${{ github.workspace }}
```

This builds the default branch of another repository rather than
`${{ github.sha }}` from P013onEr. Replace it with a checkout of the triggering
commit and recursive submodules before using CI success as the release gate.
The w2599 branch's `actions/checkout@main` pattern demonstrates the right
repository/ref behavior, but a stable pinned major such as
`actions/checkout@v4` is preferable.

Version sources at the baseline are:

- `BaseBin/_external/basebin/.version`: `2.4.9.25`
- Xcode `MARKETING_VERSION`: `2.4.9` for Debug and Release
- artifact name: `roothide-Dopamine-<basebin-version>-<short-sha>.tipa`

For the requested "increase the version only after a successful build" flow,
first make CI build the actual feature SHA. Treat the successful feature build
as validation, then bump `.version` in a separate release commit and build
that exact commit again. If the user-visible app version also needs the patch
component, update both Xcode `MARKETING_VERSION` entries consistently; the
current project deliberately separates `2.4.9` marketing version from
`2.4.9.25` basebin/artifact version.

## Recommended port boundary

1. Keep the DarkSword implementation already present in `e890f57`; do not
   re-import it. Preserve #776's fixed `DarkSword.m` and the existing roothide
   adaptations.
2. Port/harden only the mount backend from `eb6ac4d` and mount/reboot pieces
   from `e236132`; drop uninject changes, branding placeholders, and
   `jbupdate.sh`.
3. Disable both update decision points using only the relevant
   `DOUIManager.m` hunks from `5432899`, then verify whether any remaining
   callers still fetch releases.
4. Repair the GitHub Actions checkout and validate the exact pushed SHA.
5. After CI passes, increment the intended version source(s), rebuild that
   release commit, and tag/release only the verified artifact.
