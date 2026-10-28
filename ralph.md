# Project Manager (vscode-project-manager)

A VS Code extension written in TypeScript that lets users save and switch between projects. It's bundled with webpack, linted with ESLint, and tested via the VS Code extension test runner (`@vscode/test-electron`).

## Setup

Install dependencies:

    npm ci

## Verification

Type-check with `tsc` and lint with ESLint, build the extension bundle with webpack, and run the VS Code extension test suite.

<!-- RALPH_CHECK_COMMANDS
npm run compile
npm run lint
RALPH_CHECK_COMMANDS_END -->

<!-- RALPH_BUILD_COMMANDS
npm run build
RALPH_BUILD_COMMANDS_END -->

<!-- RALPH_REGRESSION_COMMANDS
npm test
RALPH_REGRESSION_COMMANDS_END -->
