ShellCheck — Notes for AI coding agents

Be concise, operate on the code, and prefer small, testable changes. This file lists the project's architecture, developer workflows, conventions and examples an AI agent should use when producing edits or tests.

High-level architecture
- Language: Haskell (Haskell98 style in many files).
- Core layers:
  - CLI/runner: `shellcheck.hs` — parses arguments, selects formatters, wires IO via `SystemInterface`.
  - Interface types: `src/ShellCheck/Interface.hs` — canonical data shapes (CheckSpec, CheckResult, SystemInterface).
  - Parser/AST: `src/ShellCheck/Parser.hs` and `src/ShellCheck/AST.hs` — tokenization and AST shapes used by analyzers.
  - Analyzer: `src/ShellCheck/Analyzer.hs` and `src/ShellCheck/AnalyzerLib.hs` — composes checks and runs them to produce TokenComments.
  - Checks: `src/ShellCheck/Checks/*` — individual check modules (Commands.hs, ControlFlow.hs, Custom.hs, ShellSupport.hs).
  - Formatters: `src/ShellCheck/Formatter/*` — output formats (TTY, JSON, GCC, Diff, CheckStyle).

Key integration points and flows
- The runner creates a `SystemInterface` and `CheckSpec`, then calls `checkScript` (see `src/ShellCheck/Checker.hs`).
- `SystemInterface` functions (`siReadFile`, `siFindSource`, `siGetConfig`) are used throughout: formatters call `siReadFile` to fetch content; parser uses `siReadFile` to resolve `source` includes.
- Analysis is driven by `analyzeScript :: AnalysisSpec -> AnalysisResult` in `src/ShellCheck/Analyzer.hs`. Add new checks by composing them into `checkers`.

Build & test workflow (what to run)
- This project uses Cabal/Stack. Quick local build/test commands:

  - Build and install with Cabal:
    cabal install

  - Using Stack (resolver in `stack.yaml`):
    stack build

  - Run unit tests:
    cabal test

Notes: building requires a Haskell toolchain (GHC/Cabal or Stack) and ~2GB RAM. The `ShellCheck.cabal` exposes the library modules and the `shellcheck` executable entrypoint `shellcheck.hs`.

Project-specific conventions
- Prefer pure functions and keep IO in `SystemInterface` and the CLI (`shellcheck.hs`). Tests and debug helpers use `mockedSystemInterface` (see `src/ShellCheck/Interface.hs`).
- Optional checks: modules expose `optionalChecks` (e.g. `ShellCheck.Checks.Commands.optionalChecks`) and the CLI/Checker merges directives from the script with `csOptionalChecks`.
- Token- and position-based fixes: use `Position`, `Replacement`, and `Fix` types from `Interface.hs` when producing suggestions/fixes.

When changing behavior
- Update `optionalChecks` in the relevant check module if adding opt-in behavior. Add tests under `test/` or small unit-style runners in the check module (`runTests` functions are common).
- If you change public library APIs, update `ShellCheck.cabal` exposed modules.

Examples to reference in edits
- To read files respecting external-sources and caching, check `ioInterface` in `shellcheck.hs` — replicate `siReadFile` usage when adding IO.
- To produce comments, create `TokenComment` / `PositionedComment` using `newTokenComment` / `newPositionedComment` and fill `tcComment` / `pcComment`.

Testing and debugging notes
- Use `mockedSystemInterface` to unit test checkers without touching the filesystem.
- `src/ShellCheck/Debug.hs` contains helpers that run checks using `runIdentity`.

If you need more context
- Look at `README.md` for usage examples and `ShellCheck.cabal`/`stack.yaml` for build constraints.

If something isn't discoverable (external CI secrets, platform-specific packaging), ask the maintainer for the missing detail instead of guessing.
