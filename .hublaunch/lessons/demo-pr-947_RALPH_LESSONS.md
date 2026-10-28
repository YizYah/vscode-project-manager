# HubLaunch Lessons Learned

This file persists context across agent sessions. Update it as you work.

## Current Status
- Phase: COMPLETE — all source changes implemented, all build gates green.
- Last action: Committed the feature.
- Blockers: None.

## Key Discoveries
- The `vscode-whats-new` git submodule is NOT initialized on a fresh checkout. `npm run compile` fails with TS2307 (cannot find `../../vscode-whats-new/src/Manager`) until you run `git submodule update --init --recursive`. Environment setup, unrelated to the feature.
- Extension Host tests (`npm test`) CANNOT run in this sandbox: the downloaded VS Code binary fails with `libnspr4.so: cannot open shared object file`. No sudo/root to install libnspr4/libnss3. This is the Q2-C conditional path → ship NO new test, document manual verification in PR body.
- Lint baseline: 0 errors, 52 pre-existing warnings (none in touched files). Expected/documented state.

## Solutions That Worked
- `git submodule update --init --recursive` fixed the compile errors.
- Build gates that pass (all exit 0): `npm run compile`, `npm run lint`, `npm run test-compile`, `npm run vscode:prepublish`.

## Things to Avoid
- Don't try to add the Extension Host test — it can't be verified green in-sandbox (missing system libs, no root). Per plan, never weaken existing tests to force anything.

## Files Modified
- NEW: src/utils/windowTitle.ts (registerProjectNameWindowTitleVariable, updateProjectNameWindowTitleVariable)
- src/statusbar/statusBar.ts (extracted detectCurrentProject; showStatusBar now returns void)
- src/extension.ts (import wiring; activation detection+register+prime; await updateCurrentProject; title refresh on save/rename/switch; removed maintainer TODO)
- README.md (${projectName} doc block after showProjectNameInStatusBar)

## Open Questions
- None.

## Next Steps
- Done. If a PR is opened: target master, credit @oiahoon, link #947 and #943, and disclose that the Extension Host suite couldn't run in-sandbox (missing libnspr4.so) with manual verification steps.
