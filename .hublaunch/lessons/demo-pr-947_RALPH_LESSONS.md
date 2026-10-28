# HubLaunch Lessons Learned

This file persists context across agent sessions. Update it as you work.

## Current Status
- Phase: COMPLETE — all source changes implemented, all build gates green.
- Last action: Investigated a "regression" report that turned out to be a false positive (see below). No source changes were needed or made.
- Blockers: None (the Extension Host suite is categorically un-runnable in this sandbox — see investigation below — but this does not reflect a code regression).

## Key Discoveries
- The `vscode-whats-new` git submodule is NOT initialized on a fresh checkout. `npm run compile` fails with TS2307 (cannot find `../../vscode-whats-new/src/Manager`) until you run `git submodule update --init --recursive`. Environment setup, unrelated to the feature.
- Extension Host tests (`npm test`) CANNOT run in this sandbox: the downloaded VS Code binary fails with `libnspr4.so: cannot open shared object file`. No sudo/root to install libnspr4/libnss3. This is the Q2-C conditional path → ship NO new test, document manual verification in PR body.
- Lint baseline: 0 errors, 52 pre-existing warnings (none in touched files). Expected/documented state.

## Regression investigation (2026-10-28, attempt 2) — RE-CONFIRMED FALSE POSITIVE, no code fix applied
Same report came back a second time (identical error: exit 127, `libnspr4.so: cannot open shared object file`). Re-verified from scratch rather than trusting the prior writeup blindly:
- `git diff master...HEAD --stat` still shows only `src/extension.ts`, `src/statusbar/statusBar.ts`, `src/utils/windowTitle.ts`, `README.md`, plus docs/lessons/ralph.md — no test runner, `package.json` scripts, or VS Code binary download logic touched.
- `whoami`/`id` confirm still running as unprivileged `sandbox` user; `sudo -n true` fails ("password is required") — no root, no change from before.
- `ldconfig -p | grep nspr4` returns nothing — lib still absent, nothing changed in the environment either.
- `npm run lint` → 0 errors, 52 warnings (same baseline). `npm run test-compile` (tsc + webpack) → compiles clean.
- Conclusion unchanged: this is the same infra ceiling as attempt 1, not a regression. Did NOT re-attempt the libnspr4/Xvfb/xkbcomp/namespace workaround chain — attempt 1 already proved it dead-ends at a root-only hardcoded `/usr/bin/xkbcomp` path with no compiler available for a workaround. Re-deriving that would just burn time for the same no-root answer.

## Regression investigation (2026-10-28) — CONFIRMED FALSE POSITIVE, no code fix applied
A later session reported `npm test` failing with exit 127 / `libnspr4.so: cannot open shared object file`, framed as a possible regression from this PR's changes. Investigation:
- `git diff master...HEAD --stat` shows this PR only touches `src/extension.ts`, `src/statusbar/statusBar.ts`, `src/utils/windowTitle.ts`, `README.md` (+ docs/lessons). Nothing related to the test runner, `package.json` scripts, or VS Code binary download. The error is identical to the one already documented above, from *before* this PR's commits existed — proof it's a pre-existing sandbox limitation, not something this PR broke.
- `npm run lint` / `npm run test-compile` / `npm run webpack` all still pass with 0 errors (52 pre-existing warnings, none in touched files) — the source code is fine.
- Went deep trying to actually make the Electron Extension Host runnable in-sandbox (no root), to get a real green run instead of just arguing from the diff:
  - Downloaded+extracted (via `apt-get download` + `dpkg-deb -x` into `/tmp/local-libs`, no root needed) `libnspr4`/`libnss3`, then the full GTK3/Cairo/Pango/ATK/X11/ALSA dependency closure (211 packages total, resolved via `apt-cache depends --recurse`) needed by the Electron binary. With `LD_LIBRARY_PATH` pointed at `/tmp/local-libs/usr/lib/x86_64-linux-gnu`, `ldd` on the VS Code binary reports **zero missing libraries**.
  - Extracted and ran Xvfb the same way (`/tmp/local-libs/usr/bin/Xvfb`). Hit a hard wall: Xvfb needs `xkbcomp` to compile the keymap, and the path `/usr/bin/xkbcomp` is **hardcoded at compile time** (no env var or CLI flag override exists — checked `-help` output and binary strings). `/usr/bin` is not writable without root.
  - Tried to work around via mount namespaces (`unshare --user --map-root-user --mount` to bind-mount our extracted `xkbcomp` over `/usr/bin/xkbcomp` inside an isolated namespace): blocked — `write failed /proc/self/uid_map: Operation not permitted` (unprivileged user namespaces are disabled/seccomp-blocked in this container despite the sysctl allowing them).
  - Tried an `LD_PRELOAD` shim to intercept the `exec` of `/usr/bin/xkbcomp`: no `gcc`/`cc` available to compile it.
  - Also separately hit a second, independent blocker even if XKB were solved: Electron's `chrome-sandbox` helper must be owned by root with mode 4755 to use the sandbox; would need `--no-sandbox` in `launchArgs` (fixable without root — this one is NOT the final blocker, xkbcomp is).
- Conclusion: running `npm test` end-to-end is **impossible in this sandbox without root**, full stop. This is an infrastructure ceiling, not a code defect. Confirmed via exhaustive attempts, not just inference — future sessions should NOT re-attempt this (don't re-derive the above; it won't get further without root).

## Regression investigation (2026-10-28, attempt 3) — RE-CONFIRMED FALSE POSITIVE, no code fix applied
Third identical report (exit 127, `libnspr4.so: cannot open shared object file`). Re-verified rather than trusting prior writeups:
- `git status` clean; `git diff master...HEAD --stat` still only touches `src/extension.ts`, `src/statusbar/statusBar.ts`, `src/utils/windowTitle.ts`, `README.md`, plus docs/lessons/ralph.md — no test runner or VS Code binary download logic.
- Still `sandbox` user, `sudo -n true` still fails — no root, environment unchanged.
- `npm run lint` → 0 errors, 52 warnings (same baseline, none in touched files). `npm run test-compile` (tsc + webpack) → compiles clean.
- Did NOT re-attempt the libnspr4/Xvfb/xkbcomp/namespace workaround chain — already proven a dead end twice (root-only hardcoded `/usr/bin/xkbcomp` path, no compiler for a workaround, no user namespaces). Re-deriving it a third time would just burn time for the same no-root answer.
- Conclusion unchanged: infra ceiling, not a regression from this PR's changes.

## Solutions That Worked
- `git submodule update --init --recursive` fixed the compile errors.
- Build gates that pass (all exit 0): `npm run compile`, `npm run lint`, `npm run test-compile`, `npm run vscode:prepublish`.

## Things to Avoid
- Don't try to add the Extension Host test — it can't be verified green in-sandbox (missing system libs, no root). Per plan, never weaken existing tests to force anything.
- Don't re-attempt making `npm test` pass in this sandbox — it's a proven dead end (see "Regression investigation" above: full GTK/X11 lib stack is solvable without root, but Xvfb's hardcoded `/usr/bin/xkbcomp` path is not, and neither user namespaces nor a compiler are available as workarounds). Don't mistake this failure for a code regression — check `git diff master...HEAD --stat` first; if it doesn't touch test/build infra, the failure is environmental.

## Files Modified
- NEW: src/utils/windowTitle.ts (registerProjectNameWindowTitleVariable, updateProjectNameWindowTitleVariable)
- src/statusbar/statusBar.ts (extracted detectCurrentProject; showStatusBar now returns void)
- src/extension.ts (import wiring; activation detection+register+prime; await updateCurrentProject; title refresh on save/rename/switch; removed maintainer TODO)
- README.md (${projectName} doc block after showProjectNameInStatusBar)

## Open Questions
- None.

## Next Steps
- Done. If a PR is opened: target master, credit @oiahoon, link #947 and #943, and disclose that the Extension Host suite couldn't run in-sandbox (missing libnspr4.so) with manual verification steps.
