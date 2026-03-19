<!-- codemod-skill-discovery:begin -->
## Codemod Skill Discovery
This section is managed by `codemod` CLI.

- Core skill: `.agents/skills/codemod/SKILL.md`
- Package skills: `.agents/skills/<package-skill>/SKILL.md`
- List installed Codemod skills: `npx codemod agent list --harness antigravity --format json`

<!-- codemod-skill-discovery:end -->

## Project: @refactory/typescript-to-rust

Codemod package that translates TypeScript into compilable Rust. Part of the [refactory-lang](https://github.com/refactory-lang) organization. The pipeline normalizes idiomatic TypeScript into a constrained subset, then applies deterministic JSSG transforms (Stage 1 syntax, Stage 2 semantics), followed by LLM-assisted resolve (Stage 3).

### Architecture

- **Normalize** (`normalize/`): 10 pre-processing transforms that rewrite idiomatic TS into the constrained "TypeScript-as-Rust" subset (e.g., `let`/`var` to `const`, `throw` to `Err`, `try-catch` to `Result`)
- **Profile** (`profile/rules/`): ast-grep validation rules that enforce the constrained TS subset
- **Transforms — Stage 1** (`transforms/stage1-*.ts`): 11 deterministic syntax transforms (imports, structs, enums, primitives, string/array methods, closures, etc.)
- **Transforms — Stage 2** (`transforms/stage2-*.ts`): 8 semantic transforms (async/await, class→struct+impl, destructuring, optional chaining, nullish coalescing, try-catch→match, Result chains, object spread)
- **Transforms — Stage 3** (`transforms/stage3-prompts/`): LLM prompt templates for Rust→Rust resolve (lifetimes, generics, traits, async patterns)
- **Shadow Rewrite** (`typescript-shadow-rewrite/rules/`): ast-grep rules that rewrite TS API calls to shadow library equivalents
- **Mappings** (`mappings/`): Type and API mapping data
- **Tests** (`tests/`): e2e, normalize, shadow-rewrite, stage1, stage2 test suites
- **Specs** (`specs/`): Implementation specifications

### Running

```bash
# Run a Stage 1 transform on a TypeScript file
npx codemod jssg run transforms/stage1-import-to-use.ts --language typescript --target <file.ts> --allow-dirty --no-interactive

# Run a normalize transform
npx codemod jssg run normalize/throw-to-err.ts --language typescript --target <file.ts> --allow-dirty --no-interactive
```

### Key Files

| File | Purpose |
|------|---------|
| `transforms/stage1-interface-to-struct.ts` | Interface/type → Rust struct transform |
| `transforms/stage1-import-to-use.ts` | ES import → Rust `use` declaration |
| `transforms/stage1-type-primitives.ts` | TS primitive types → Rust types (`number→f64`, `string→String`, etc.) |
| `transforms/stage2-class-to-struct-impl.ts` | Class → struct + impl block |
| `transforms/stage2-async-await.ts` | async/await → tokio async |
| `transforms/stage2-try-catch-to-match.ts` | try/catch → match on Result |
| `normalize/throw-to-err.ts` | Rewrite `throw` to `return Err(...)` |
| `normalize/try-catch-to-result.ts` | Rewrite try/catch to Result pattern |
| `normalize/commonjs-to-esm.ts` | CommonJS → ES module syntax |
| `profile/rules/` | ast-grep rules enforcing constrained TS subset |
