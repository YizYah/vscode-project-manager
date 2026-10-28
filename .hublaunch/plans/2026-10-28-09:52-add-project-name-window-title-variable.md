# Add `${projectName}` window-title variable (carry forward PR #947)

## Plan Summary

- **What / why:** Carry forward stalled upstream PR [#947](https://github.com/alefragnani/vscode-project-manager/pull/947) (by @oiahoon, closes #943) as a clean branch off `master`. Expose a `${projectName}` variable for VS Code's built-in `window.title`, backed by a context key (`projectManager.projectName`) that stays in sync with the detected current project across activation, save, rename, and project switch.
- **Key decision (safer alternative):** Instead of re-ordering `showStatusBar`'s internals (as the original PR did), extract current-project detection into a small reusable helper that both `showStatusBar` (for rendering) and the activation/window-title path call. This directly discharges the maintainer's existing `TODO` at `src/extension.ts:165-169` rather than growing the function it flags.
- **Host-version guard:** `registerWindowTitleVariable` only exists on VS Code ≥ 1.93; the declared engine floor is `^1.78.0` (`package.json:8`). Registration is gated behind a `commands.getCommands()` check so older hosts are unaffected and no engine bump is needed.
- **Most important files:** new `src/utils/windowTitle.ts`; `src/statusbar/statusBar.ts` (extract `detectCurrentProject`); `src/extension.ts` (wire registration + sync on save/rename/switch); `README.md` (one doc block). No new dependencies, no `package.json` `contributes` change.
- **Priority / complexity:** Medium priority, low–medium complexity. One logical change, ~4 source files + README (+ optional test). PR body must credit @oiahoon and link #947 / #943.

## Problem Statement

VS Code lets users customize the editor title bar through the built-in `window.title` setting using template variables (e.g. `${activeEditorShort}`, `${separator}`). Project Manager already knows which saved/detected project the current window belongs to (it renders it in the status bar), but it does not surface that project name to `window.title`. Issue #943 requests a `${projectName}` variable so users can show the active project in the title bar.

Upstream PR #947 implemented this but has gone stale: it is in a `dirty`/conflicting state (+75/-28 across 4 files). The conflict is **mechanical, not behavioral** — `master` added `import { ProjectManagerApiImpl } from "./api/apiImpl";` immediately after the `import { Project } from "./core/project";` line the PR anchored its new import to (`src/extension.ts:37-38`), and `master` now ends `deactivate()` with a trailing newline, so the PR's EOF fix-up hunk is already satisfied. The feature logic still applies to current `master`.

This plan reproduces the same user-facing feature, rebased onto current `master`, but adopts the reviewer-preferred shape: extracting detection into a helper instead of reworking `showStatusBar` in place.

#### Planning Context

**Key requirements**
- A `${projectName}` variable is usable in `window.title`, resolving to the current project's name (or empty when there is no current project).
- The backing context key `projectManager.projectName` is kept in sync on: activation, project save (new favorite), project rename, and workspace/project switch (`updateCurrentProject`).
- The variable name (`projectName`) and context key (`projectManager.projectName`) are preserved verbatim from PR #947 — they are the user-facing `${projectName}` contract for issue #943 and must not change.
- Registration must be skipped on hosts where `registerWindowTitleVariable` is unavailable (VS Code < 1.93), preserving the `^1.78.0` engine compatibility with no manifest change.
- Current-project detection must work even when the status-bar display (`projectManager.showProjectNameInStatusBar`) is disabled.

**Decisions made (with rationale)**
- **Adopt the safer alternative (helper extraction), not the original in-place rework.** Decision from the confirmed understanding. The original PR moved the detection cascade ahead of the `showStatusConfig` early-return *inside* `showStatusBar`, enlarging a function the maintainer already flagged with a `TODO` (`src/extension.ts:165-169`) asking for that detection to be *extracted*. Extracting `detectCurrentProject` satisfies the request (detection works regardless of status-bar visibility) while honoring the TODO and keeping each function single-purpose.
- **Gate registration at runtime via `commands.getCommands()`**, reusing `commands.executeCommand("setContext", …)` — the established way this codebase publishes state to VS Code (see the new util, matching the `setContext` idiom already used across the extension). No `engines.vscode` bump; no new `contributes` entry (the variable is registered at runtime).
- **No new dependencies** — the feature uses only the built-in `vscode` `commands` API.
- **Testing is conditional (Q2 answer C):** add the version-guarded Extension Host test only if it runs cleanly in the sandbox; otherwise ship no new test and document manual verification in the PR body. Never weaken or skip existing tests to make it pass.

**Out of scope**
- No `.hublaunch/goals.md` or any file outside the feature (Q3 answer C).
- No CI / workflow / release-config / license / funding / security-file edits (binding fork rules).
- No `CHANGELOG.md` entry (see Assumptions — the repo's CHANGELOG is release-version-keyed and curated by the maintainer; the original PR shipped none).
- No refactors/renames/formatting sweeps beyond the detection extraction required by the chosen design.
- No change to `updateCurrentProject`'s existing (deliberately lighter) detection logic beyond adding the title refresh — it is preserved as the original author left it.

## Detailed Requirements

**Functional**
1. After activation, `${projectName}` in `window.title` resolves to the current project's name on hosts ≥ 1.93; on older hosts the setting is simply not registered and behaves as today.
2. Saving a new favorite project for the current window updates `projectManager.projectName`.
3. Renaming the current project updates both the in-memory `Container.currentProject.name` and `projectManager.projectName`.
4. Switching workspace/current project (via `updateCurrentProject`) updates `projectManager.projectName`.
5. The current project is detected (and `Container.currentProject` populated) even when `projectManager.showProjectNameInStatusBar` is `false`.

**Technical**
- New module `src/utils/windowTitle.ts` exporting `registerProjectNameWindowTitleVariable()` and `updateProjectNameWindowTitleVariable(project?)`, both `async`, both using `vscode` `commands`.
- `src/statusbar/statusBar.ts` exports a new `detectCurrentProject(projectStorage, locators)` helper containing the remote/local locator cascade; `showStatusBar` consumes it for rendering.
- `src/extension.ts` sets `Container.currentProject` from `detectCurrentProject(...)` directly (not from `showStatusBar`'s return), then registers and updates the window-title variable.
- Follows TypeScript/ESLint conventions, GPLv3 file header on the new file, and the small single-purpose `src/utils/*` module shape (e.g. `src/utils/remote.ts:1-24`, `src/utils/icons.ts`).

**Non-Functional**
- No regression to status-bar behavior when enabled; no new behavior when the status bar is disabled other than `Container.currentProject` now being populated (already consumed, null-guarded, by `src/sidebar/decoration.ts:14`).
- Success bar: `npm run compile`, `npm run lint` (0 new errors), `npm run test-compile`, `npm run vscode:prepublish` all pass (scripts at `package.json:1134-1142`). Existing Extension Host tests pass unchanged.

## Proposed Solution

Introduce a dedicated window-title util and reuse it from the extension lifecycle. Extract the current-project detection cascade out of `showStatusBar` into an exported helper so detection is independent of status-bar rendering and visibility, discharging the maintainer TODO.

```
activate()
  │
  ├─ detectCurrentProject(storage, locators) ──► Container.currentProject   (statusBar.ts helper; works even if status bar disabled)
  ├─ showStatusBar(storage, locators)          ──► renders using detectCurrentProject() internally
  ├─ registerProjectNameWindowTitleVariable()  ──► windowTitle.ts (guarded: only if `registerWindowTitleVariable` exists)
  └─ updateProjectNameWindowTitleVariable(cur) ──► windowTitle.ts → commands.executeCommand("setContext", "projectManager.projectName", name)

save  ─► updateCurrentProject() ─► updateProjectNameWindowTitleVariable(found)
rename─► mutate Container.currentProject.name ─► updateProjectNameWindowTitleVariable(current)
```

The window-title util mirrors PR #947's `src/utils/windowTitle.ts` verbatim (same function names, variable name `projectName`, context key `projectManager.projectName`, guard logic). The divergence from #947 is confined to *where detection lives*: a helper rather than a reordered `showStatusBar`.

## Implementation Steps

### Phase 1 — Add the window-title util (new file)

Create `src/utils/windowTitle.ts`, reproducing PR #947's module exactly (it is a pure addition into the existing `src/utils/` folder; no conflict). Follow the GPLv3 header + single-purpose shape of `src/utils/remote.ts:1-6`.

```ts
/*---------------------------------------------------------------------------------------------
*  Copyright (c) Alessandro Fragnani. All rights reserved.
*  Licensed under the GPLv3 License. See License.md in the project root for license information.
*--------------------------------------------------------------------------------------------*/

import { commands } from "vscode";
import { Project } from "../core/project";

const PROJECT_NAME_CONTEXT_KEY = "projectManager.projectName";
const REGISTER_WINDOW_TITLE_VARIABLE_COMMAND = "registerWindowTitleVariable";

export async function registerProjectNameWindowTitleVariable(): Promise<void> {
    const availableCommands = await commands.getCommands();
    if (!availableCommands.includes(REGISTER_WINDOW_TITLE_VARIABLE_COMMAND)) {
        return;
    }

    await commands.executeCommand(
        REGISTER_WINDOW_TITLE_VARIABLE_COMMAND,
        "projectName",
        PROJECT_NAME_CONTEXT_KEY
    );
}

export async function updateProjectNameWindowTitleVariable(project?: Project): Promise<void> {
    await commands.executeCommand("setContext", PROJECT_NAME_CONTEXT_KEY, project?.name);
}
```

- `Project` is imported from `../core/project` (`src/core/project.ts:6`); `name` is a mutable `string`, which the rename flow relies on.

### Phase 2 — Extract detection in `src/statusbar/statusBar.ts`

Current state: `showStatusBar` (`src/statusbar/statusBar.ts:16-78`) early-returns on `!showStatusConfig || !currentProjectPath` (line 29) and runs the locator cascade inline (lines 52-72), returning the found `Project`.

1. Add an exported `detectCurrentProject` helper holding the workspace resolution + remote/local locator cascade (relocating, **not rewriting**, the logic currently at lines 24-27 and 52-72):

```ts
export function detectCurrentProject(projectStorage: ProjectStorage, locators: Locators): Project {
    const workspace0 = workspace.workspaceFile ? workspace.workspaceFile :
        workspace.workspaceFolders ? workspace.workspaceFolders[ 0 ].uri :
            undefined;
    const currentProjectPath = workspace0 ? workspace0.fsPath : undefined;
    if (!workspace0 || !currentProjectPath) { return undefined; }

    if (isRemoteUri(workspace0)) {
        return projectStorage.existsRemoteWithRootPath(workspace0);
    }

    let foundProject = projectStorage.existsWithRootPath(currentProjectPath, true);
    if (!foundProject) { foundProject = locators.vscLocator.existsWithRootPath(currentProjectPath); }
    if (!foundProject) { foundProject = locators.gitLocator.existsWithRootPath(currentProjectPath); }
    if (!foundProject) { foundProject = locators.mercurialLocator.existsWithRootPath(currentProjectPath); }
    if (!foundProject) { foundProject = locators.svnLocator.existsWithRootPath(currentProjectPath); }
    if (!foundProject) { foundProject = locators.anyLocator.existsWithRootPath(currentProjectPath); }
    return foundProject;
}
```

2. Rewire `showStatusBar` to keep its `showStatusConfig`/`currentProjectPath` early-return and its `projectName` fast-path, but call `detectCurrentProject(projectStorage, locators)` for the search case. `showStatusBar` becomes rendering-only; its return value is no longer needed by callers (the two save-flow callers at `src/extension.ts:328,358` already ignore it), so it may return `void`:

```ts
export function showStatusBar(projectStorage: ProjectStorage, locators: Locators, projectName?: string): void {
    const showStatusConfig = workspace.getConfiguration("projectManager").get("showProjectNameInStatusBar");
    const workspace0 = /* unchanged resolution */;
    const currentProjectPath = workspace0 ? workspace0.fsPath : undefined;
    if (!showStatusConfig || !currentProjectPath) { return; }

    // ...unchanged statusItem creation / text / tooltip / command (lines 31-43)...

    if (projectName) {
        statusItem.text += projectName;
        statusItem.show();
        return;
    }

    const foundProject = detectCurrentProject(projectStorage, locators);
    if (foundProject) {
        statusItem.text += foundProject.name;
        statusItem.show();
    }
}
```

- Keep the existing imports (`Project` at `src/statusbar/statusBar.ts:12`, `isRemoteUri` at line 10); they are now used by the helper.
- Note: `detectCurrentProject` resolves `workspace0` itself, so the status-bar path and the activation path share one detection source of truth.

### Phase 3 — Wire lifecycle sync in `src/extension.ts`

1. **Import** (resolves the mechanical conflict from #947): add
   `import { registerProjectNameWindowTitleVariable, updateProjectNameWindowTitleVariable } from "./utils/windowTitle";`
   alongside the existing imports (place it near the other `./utils`/feature imports; the original PR anchored to the `import { Project }` line now at `src/extension.ts:37`, which is immediately followed by `import { ProjectManagerApiImpl } from "./api/apiImpl";` at line 38 — add the new import after these rather than splicing into that hunk). Also import `detectCurrentProject` from `./statusbar/statusBar` (extend the existing `import { showStatusBar, updateStatusBar }` at `src/extension.ts:17`).

2. **Activation** (`src/extension.ts:170-171`): set `Container.currentProject` from the helper (so it is populated regardless of status-bar visibility), render the status bar, then register + prime the title variable. The maintainer TODO comment at `src/extension.ts:165-169` about extracting detection from `showStatusBar` can be removed, since this change performs that extraction (keep the ordering note if any remains relevant):

```ts
const currentProject = detectCurrentProject(projectStorage, locators);
Container.currentProject = currentProject;
showStatusBar(projectStorage, locators);
await registerProjectNameWindowTitleVariable();
await updateProjectNameWindowTitleVariable(currentProject);
```

3. **Save flow** (`saveProjectInternal`, `src/extension.ts:327-330`): make `updateCurrentProject` `async` and `await` it:

```ts
if (!node) {
    showStatusBar(projectStorage, locators, projectName);
    await updateCurrentProject();
}
```

4. **`updateCurrentProject`** (`src/extension.ts:436-449`): convert to `async`; keep its existing (lighter, intentional) detection unchanged; append the title refresh:

```ts
async function updateCurrentProject() {
    // ...unchanged detection (lines 437-447)...
    Container.currentProject = foundProject;
    await updateProjectNameWindowTitleVariable(foundProject);
}
```

5. **Rename flow** (`renameProject`, `src/extension.ts:539-559`): make the `showInputBox(...).then(...)` callback `async`; after `updateStatusBar(...)` (line 555), refresh the current project name + variable when the renamed project is the current one:

```ts
vscode.window.showInputBox(ibo).then(async newName => {
    // ...unchanged...
    updateStatusBar(oldName, node.command.arguments[0], newName);
    if (Container.currentProject?.name.toLowerCase() === oldName.toLowerCase()) {
        Container.currentProject.name = newName;
        await updateProjectNameWindowTitleVariable(Container.currentProject);
    }
    // ...
});
```

### Phase 4 — README documentation

In `README.md`, insert the `${projectName}` block immediately after the `projectManager.showProjectNameInStatusBar` entry (`README.md:298`) and before the `openInNewWindowWhenClickingInStatusBar` entry (`README.md:304`), matching PR #947's wording and the surrounding list/code-block style:

```md
* Display the Project Name in the Window Title (VS Code 1.93 and newer)

Project Manager exposes the `${projectName}` variable for use in the built-in `window.title` setting.

```json
    "window.title": "${projectName}${separator}${activeEditorShort}"
```
```

### Phase 5 — Test (conditional — Q2 answer C)

Attempt a version-guarded Extension Host test, then keep it **only if it runs cleanly in the sandbox**; otherwise delete it and rely on documented manual verification.

- Add to `src/test/suite/` following the existing style — activate via `vscode.extensions.getExtension('alefragnani.project-manager')` and assert against the real host (`src/test/suite/extension.test.ts:17-34`).
- Guard on capability, not raw version: at suite/test start call `vscode.commands.getCommands()` and **skip** (`this.skip()`) when `registerWindowTitleVariable` is absent (host < 1.93), so the test passes on the 1.78 floor.
- Assert that `projectManager.projectName` tracks save and rename. Because a `setContext` value is not directly readable back, assert via observable behavior the test can reach (e.g. drive the public API / save+rename flow used in `extension.test.ts:36-45` and assert `Container.currentProject?.name` reflects the change) rather than reading the context key directly. If a clean, non-flaky assertion cannot be achieved in-sandbox, drop the test.
- If kept, run `npm run test` (`package.json:1142`) and confirm green. If dropped, note in the PR body that the window-title API (≥1.93) could not be exercised in the sandbox and list the manual verification steps from PR #947.

### Phase 6 — Branch, verify, PR

- Branch from `master` (current commit `aefcf11b422e2a260faeb63a7dfdd6adbf3b1e03`), one logical change.
- Run `npm run compile`, `npm run lint`, `npm run test-compile`, `npm run vscode:prepublish`.
- Commit in repo style (conventional-ish subject, reference the issue), e.g. `Add ${projectName} window title variable (#943)`.
- PR targets `master`, credits @oiahoon, and links #947 and #943.

<details><summary><b>Implementation Detail</b></summary>

### Edge Cases
- **Host < 1.93:** `registerProjectNameWindowTitleVariable` returns early (command absent); `updateProjectNameWindowTitleVariable` still runs `setContext` harmlessly (the key simply isn't consumed). No error, no behavior change.
- **No current project / no workspace:** `detectCurrentProject` returns `undefined`; `updateProjectNameWindowTitleVariable(undefined)` sets the context key to `undefined` → `${projectName}` resolves to empty. Matches PR #947.
- **Status bar disabled:** `Container.currentProject` is now populated from `detectCurrentProject` at activation; `src/sidebar/decoration.ts:14` already null-guards its use, so no regression.
- **Rename of a non-current project:** the `name.toLowerCase()` guard ensures only the current project's name + variable are refreshed.
- **Remote/virtual workspaces:** handled by the `isRemoteUri` branch inside `detectCurrentProject` (relocated from the original inline cascade).
- **Save when `node` is set:** unchanged — the save flow only calls `updateCurrentProject` when `!node` (`src/extension.ts:327`).

### Error Handling
- `commands.executeCommand`/`getCommands` rejections propagate as before; the registration guard prevents calling a missing command. No new try/catch is introduced (none in PR #947; consistent with surrounding code).

### Performance
- Detection runs the same locator cascade as today, just relocated; cost is unchanged. It now also runs once at activation when the status bar is disabled (previously skipped) — a single detection pass, negligible.

### Security
- No new inputs, network, filesystem, or dependency surface. Uses only built-in `vscode` commands.

### Testing (detail)
- See Phase 5. Existing suites (`src/test/suite/*.test.ts`) must pass unchanged; do not alter mocks or existing assertions.

### Rollout
- Pure additive feature behind a runtime capability check; no migration, no setting default change, no manifest change.

### Dependencies & Related Work
- Carries forward [PR #947](https://github.com/alefragnani/vscode-project-manager/pull/947) (@oiahoon); closes [#943](https://github.com/alefragnani/vscode-project-manager/issues/943).
- Touches: new `src/utils/windowTitle.ts`; `src/statusbar/statusBar.ts`; `src/extension.ts`; `README.md`; optional test under `src/test/suite/`.
- Related consumers of `Container.currentProject`: `src/sidebar/decoration.ts:14` (null-guarded), `src/core/container.ts:32-38`.

</details>

## Technical Considerations

- **Dependencies:** none added. `engines.vscode` stays `^1.78.0` (`package.json:8`).
- **Config / manifest:** no `contributes` entry — the title variable is registered at runtime via the `registerWindowTitleVariable` command.
- **Conflict reconciliation:** the only mechanical conflicts from #947 are (a) the `extension.ts` import hunk shifted by `import { ProjectManagerApiImpl }` (`src/extension.ts:38`) and (b) the already-satisfied EOF-newline fix-up on `deactivate()`. Both are resolved by placing the new import among current imports and omitting the newline hunk. Behavioral hunks still match current `master`.
- **Security:** no CI/license/funding/security-file edits (binding fork rules).

## Testing Strategy

- **Unit:** none — the repo has no pure unit tests for this layer; all tests are Extension Host integration tests.
- **Integration (conditional, Q2-C):** version-guarded Extension Host test under `src/test/suite/` asserting `projectManager.projectName` tracks save/rename, skipped where `registerWindowTitleVariable` is absent (< 1.93). Kept only if it runs cleanly in-sandbox; existing tests must never be weakened/skipped to make it pass.
- **Manual (from PR #947, document in PR body if the test is dropped):**
  - Set `"window.title": "${projectName}${separator}${activeEditorShort}"`, open a saved project, confirm the title shows the project name.
  - Save a new favorite → title/context updates.
  - Rename the current project → title updates to the new name.
  - Disable `projectManager.showProjectNameInStatusBar` → current project still detected (sidebar highlight + `${projectName}` still work).
  - On a host < 1.93, confirm no error and unchanged behavior.
- **Build gates:** `npm run compile`, `npm run lint` (expect 0 new errors; repo has 52 pre-existing warnings), `npm run test-compile`, `npm run vscode:prepublish`.

## Documentation Updates

- `README.md`: one block documenting `${projectName}` and the VS Code 1.93 requirement, inserted after `showProjectNameInStatusBar` (`README.md:298`). No `CHANGELOG.md` entry (see Assumptions).

## Acceptance Criteria

1. `src/utils/windowTitle.ts` exists and exports `registerProjectNameWindowTitleVariable()` and `updateProjectNameWindowTitleVariable(project?)` with the GPLv3 header, variable name `projectName`, and context key `projectManager.projectName`.
2. `src/statusbar/statusBar.ts` exports `detectCurrentProject(projectStorage, locators)`; `showStatusBar` uses it and renders identically when enabled; the maintainer TODO about extracting detection is discharged.
3. On a host ≥ 1.93, `${projectName}` in `window.title` resolves to the current project and updates on activation, save, rename, and project switch.
4. On a host < 1.93, the extension activates without error and behaves as before (no registration).
5. The current project is detected and `Container.currentProject` populated even when `projectManager.showProjectNameInStatusBar` is `false`.
6. `README.md` documents the feature after the `showProjectNameInStatusBar` entry.
7. `npm run compile`, `npm run lint`, `npm run test-compile`, `npm run vscode:prepublish` pass; existing tests pass unchanged. Any inability to run the full Extension Host suite in-sandbox is disclosed in the PR body, not worked around by weakening tests.
8. Change is a single logical commit on a branch off `master`, targeting `master`, crediting @oiahoon and linking #947 and #943; no new dependencies; no CI/license/funding/security/`contributes`/`.hublaunch` edits.

## Assumptions

- **Repo Snapshot matches checkout:** HEAD is `aefcf11b422e2a260faeb63a7dfdd6adbf3b1e03` (the snapshot SHA); no drift was observed in the files this plan touches. `src/statusbar/statusBar.ts` matches PR #947's "before" verbatim; `README.md:298-304` matches; `src/extension.ts:37-38` confirms the import-ordering conflict cause.
- **Final diff = PR #947's intent, re-shaped:** the four user-facing artifacts (README block, `src/utils/windowTitle.ts`, title-sync wiring in `src/extension.ts`, detection reachable regardless of status-bar visibility) are reproduced, with the detection delivered via an extracted `detectCurrentProject` helper (safer alternative, Q1-B) instead of re-ordering `showStatusBar`.
- **Names preserved verbatim:** variable `projectName` and context key `projectManager.projectName` are kept exactly (changing them breaks the `${projectName}` contract from #943).
- **`updateCurrentProject` detection left as-is:** only the title refresh is added; its deliberately lighter detection (remote/`existsWithRootPath` only) is preserved to honor the original author's logic and keep the change minimal.
- **`showStatusBar` return type:** since all remaining callers either ignore the return or are replaced by the direct `detectCurrentProject` call, `showStatusBar` returns `void`. If preferred for minimal churn, it may instead return `detectCurrentProject(...)`; the plan chooses `void` for clarity, with activation calling the helper directly.
- **Maintainer TODO comment (`src/extension.ts:165-169`):** removed because this change performs the extraction it requests; any still-relevant ordering note is kept.
- **Testing:** version-guarded Extension Host test is added only if it runs cleanly in-sandbox (Q2-C); otherwise no new test ships and the PR body documents manual verification. Existing tests are never weakened or skipped.
- **No `CHANGELOG.md` entry:** `CHANGELOG.md` is keyed by released version numbers (e.g. `## [13.1.1] - 2026-03-31`) and curated by the maintainer at release time; PR #947 added none. Following the smallest-change rule and Q3-C (no files outside the fix), none is added. The reviewer may add one at release.
- **No `.hublaunch/goals.md`** and no other files outside the feature (Q3-C).
- **Commit/PR style:** conventional-ish subject referencing the issue, one logical change; PR targets `master`, credits @oiahoon, links #947 and #943; no CI/license/funding/security edits.
