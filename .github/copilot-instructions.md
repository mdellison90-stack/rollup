# GitHub Copilot Agent Instructions for Rollup

Keep instructions concise, only add non-obvious information. Proactively update .github/copilot-instructions.md to prevent future mistakes.

## Architecture

- TypeScript + Rust hybrid: Rust code in `rust/` (bindings_napi, bindings_wasm, parse_ast crates) called via `native.js` and `native.wasm.js`
- Tests run against full artifact only—no unit tests to allow easy refactoring of internal APIs
- Test cases in `test/*/samples/` are configured via `_config.js` files; focus tests with `solo: true`
- See CONTRIBUTING.md "How to write tests" for test type selection (function/form/chunking-form/cli/etc.)
- AST parsing happens in Rust, converted to binary buffer, then decoded in TypeScript
  - To extend AST: update `scripts/ast-types.js`, write encoder in `rust/parse_ast/src/convert_ast/converter.rs`, create TS classes in `src/ast/nodes`
  - Auto-generated files: `src/utils/bufferToAst.ts` and `src/ast/bufferParsers.ts` (regenerate with `npm run build:ast-converters`)

## Development Workflow

- Quick iteration: `npm run update:js` (TS changes) or `npm run update:napi` (Rust changes), then `npm run test:only`
- Rust only: `npm run build:napi`
- Fast build for testing: `npm run build:quick` (CommonJS only, some tests will fail - browser and CLI tests)
- Full build: `npm run build` (includes Rust release build, WASM, and all JS formats)
- Local REPL development: `npm run dev` (starts Vite dev server with hot reload for Rollup changes)

## Code Organization

- Core logic in `src/`: `Graph.ts`, `Module.ts`, `ModuleLoader.ts`, `Chunk.ts`, `Bundle.ts`
- Build phase: module graph generation, tree-shaking (in `Graph.build()`)
- Generate phase: chunk assignment, rendering (in `Bundle.generate()`)
- Entry points: `src/node-entry.ts` (Node.js), `src/browser-entry.ts` (browser)
- CLI in `cli/` folder, config loading in `cli/run/loadConfigFile.ts`

## Code Style & Conventions

- Tabs for indentation (see `.prettierrc.json`)
- Single quotes, no trailing commas
- `arrowParens: avoid` - omit parens for single-arg arrow functions
- TypeScript: use `interface` not `type`, consistent type imports (`import type`)
- Class members ordered alphabetically
- Unused vars/args prefixed with `_`
- Test descriptions: clear, descriptive language - `description: 'does X when Y happens'`

## Testing

- Always test edge cases, especially in core logic or build/test infrastructure
- Test names and descriptions use clear, descriptive language of the expected behavior, e.g. "description: 'does X when Y happens"
- To focus a single test: add `solo: true` to `_config.js`, then `npm run build:quick && npm run test:quick`
- Test types (see CONTRIBUTING.md):
  - `test/function/` - bundle + run code, best for most cases
  - `test/form/` - compare bundled output without running
  - `test/chunking-form/` - multi-file outputs
  - `test/cli/` - CLI behavior
  - `test/browser/` - browser build

## Build System & Artifacts

- Native modules: platform-specific `.node` files in `dist/` (auto-added as optionalDependencies during publish)
- WASM artifacts: `wasm/` (browser), `wasm-node/` (Node.js WASM fallback)
- Build outputs: `dist/rollup.js` (CommonJS), `dist/es/rollup.js` (ESM), browser build
- `rollup.config.ts` orchestrates all builds
- ESLint ignores test samples (`test/*/samples/**/*.*`) except `_config.js` files

## Debugging & Tools

- VS Code launch config in `.vscode/launch.json` for Mocha tests
- Git symlinks required on Windows: enable Developer Mode and `git config core.symlinks true`
- Rust toolchain version in `rust-toolchain.toml` (auto-managed by rustup)

## Common Pitfalls

- Don't add unit tests - only integration tests via full build
- Don't fix style violations in test sample files (except `_config.js`)
- Don't remove working code unless absolutely necessary
- Generated files require regeneration after AST changes: `npm run build:ast-converters`
- First Rust build is slow (~minutes), subsequent builds are faster

## Code Review Focus

- **Ignore style/linting issues** in test sample files (`test/*/samples/`) except for `_config.js` files
- Test samples intentionally violate best practices to test edge cases—do not flag style violations in these files
- Focus reviews on production code quality

## Delegate to Background Agent

When working on complex tasks, use the `task` tool to delegate work to specialized agents:
- Use `explore` agent for codebase exploration, finding files, searching code patterns
- Use `task` agent for running builds, tests, linters with verbose output
- Use `general-purpose` agent for complex multi-step tasks requiring full toolset
- Provide complete context in agent prompts - each agent starts with fresh context
- After delegating, trust agent results on success; refine prompt on failure
