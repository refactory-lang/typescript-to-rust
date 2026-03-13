# Quality Checklist: Phase 1 TS Pipeline Requirements

**Feature Branch**: `001-ts-pipeline-phase1`
**Last Updated**: 2026-03-13

## Profile Validation (ast-grep rules)

- [ ] **FR-001** Rule rejects `any` type annotations
- [ ] **FR-001** Rule rejects `unknown` type used without narrowing
- [ ] **FR-001** Rule rejects dynamic property access (`obj[dynamicKey]`)
- [ ] **FR-001** Rule rejects `eval()` calls
- [ ] **FR-001** Rule rejects `Function()` constructor
- [ ] **FR-001** Rule rejects `with` statements
- [ ] **FR-001** Rule rejects prototype manipulation (`X.prototype.y = ...`)
- [ ] **FR-001** Rule rejects `arguments` object usage
- [ ] **FR-001** Rule rejects implicit `this` binding (unbound method references)

## Normalizers (10 total)

- [ ] **FR-002** `normalize/promise`: `Promise<T>` return types to `Result<T, E>`
- [ ] **FR-002** `normalize/promise`: `async/await` to Rust async equivalents
- [ ] **FR-002** `normalize/promise`: `.then()/.catch()` chains to `?` operator chains
- [ ] **FR-003** `normalize/interface`: Interface-as-trait (polymorphic usage) to `trait`
- [ ] **FR-003** `normalize/interface`: Interface-as-shape (data usage) to `struct`
- [ ] **FR-003** `normalize/interface`: Interface extension (`extends`) to trait inheritance or struct composition
- [ ] **FR-004** `normalize/enum`: Numeric enums to Rust enum with explicit discriminants
- [ ] **FR-004** `normalize/enum`: String enums to Rust enum with string-valued variants
- [ ] **FR-004** `normalize/enum`: Const enums inlined at usage sites
- [ ] **FR-005** `normalize/class`: `class` to `struct` + `impl`
- [ ] **FR-005** `normalize/class`: `constructor` to `fn new()`
- [ ] **FR-005** `normalize/class`: `extends` to trait composition or struct embedding
- [ ] **FR-005** `normalize/class`: `private`/`protected`/`public` to Rust visibility (`pub`/non-`pub`)
- [ ] **FR-006** `normalize/union-type`: Union types to Rust enums with variants
- [ ] **FR-006** `normalize/union-type`: `From` trait implementations generated for each variant
- [ ] **FR-006** `normalize/union-type`: Discriminated unions (tag field) to Rust enum with `#[serde(tag = "...")]`
- [ ] **FR-007** `normalize/optional-chaining`: `?.` to `.as_ref().map()` or `?` chains
- [ ] **FR-007** `normalize/optional-chaining`: `??` to `.unwrap_or()` / `.unwrap_or_else()`
- [ ] **FR-007** `normalize/optional-chaining`: `?.()` (optional call) to `.map(|f| f())` pattern
- [ ] **FR-008** `normalize/type-assertion`: `value as Type` to `.into()` / `.try_into()` / cast
- [ ] **FR-008** `normalize/type-assertion`: `<Type>value` (legacy syntax) to same Rust cast
- [ ] **FR-009** `normalize/generics`: `<T extends U>` to `<T: U>`
- [ ] **FR-009** `normalize/generics`: Multiple constraints to `where` clauses
- [ ] **FR-009** `normalize/generics`: Default type parameters to Rust defaults
- [ ] **FR-010** `normalize/decorator`: Decorators with direct Rust macro mappings converted
- [ ] **FR-010** `normalize/decorator`: Unmappable decorators routed to tier3 prompts

## Shadow Rewrite (3 libraries)

- [ ] **FR-011** Shadow library 1: rewrites `import` / `import { ... } from` / `import * as` forms
- [ ] **FR-011** Shadow library 2: rewrites `import` / `import { ... } from` / `import * as` forms
- [ ] **FR-011** Shadow library 3: rewrites `import` / `import { ... } from` / `import * as` forms

## Tier 1 - Syntax Transforms (1-8)

- [ ] **FR-012** Transform 1: Braces/semicolons normalization
- [ ] **FR-012** Transform 2: `function` / `const fn` to `fn`
- [ ] **FR-012** Transform 3: `let`/`const` to `let`/`let mut`
- [ ] **FR-012** Transform 4: Arrow functions to closures (`|args| { body }`)
- [ ] **FR-012** Transform 5: Template literals to `format!()`
- [ ] **FR-012** Transform 6: Destructuring to Rust pattern matching
- [ ] **FR-012** Transform 7: Spread operator to iterator chains
- [ ] **FR-012** Transform 8: Type annotation syntax (`x: Type` stays, generics `<T>` adjusted)

## Tier 1 - Error Transforms (9-12)

- [ ] **FR-012** Transform 9: `throw` to `return Err()`
- [ ] **FR-012** Transform 10: `try/catch` to `match` on `Result`
- [ ] **FR-012** Transform 11: Error class hierarchies to error enums with `thiserror`
- [ ] **FR-012** Transform 12: `finally` blocks to `Drop` or explicit cleanup

## Tier 1 - Type Transforms (13-18)

- [ ] **FR-012** Transform 13: `string` -> `String`
- [ ] **FR-012** Transform 14: `number` -> `i64`/`f64` (usage-based)
- [ ] **FR-012** Transform 15: `boolean` -> `bool`
- [ ] **FR-012** Transform 16: `Array<T>` / `T[]` -> `Vec<T>`
- [ ] **FR-012** Transform 17: `Map<K,V>` -> `HashMap<K,V>`
- [ ] **FR-012** Transform 18: `Set<T>` -> `HashSet<T>`

## Tier 2 - Control Transforms (19-24)

- [ ] **FR-012** Transform 19: `for...of` to `for x in iter`
- [ ] **FR-012** Transform 20: `for...in` to `for (k, v) in map.iter()`
- [ ] **FR-012** Transform 21: `switch/case` to `match`
- [ ] **FR-012** Transform 22: `if/else` chains to `match` where applicable
- [ ] **FR-012** Transform 23: `while`/`do-while` to `loop`/`while`
- [ ] **FR-012** Transform 24: `.map()`/`.filter()`/`.reduce()` to Rust iterator chains

## Tier 2 - Module Transforms (25-29)

- [ ] **FR-012** Transform 25: `import`/`export` to `mod`/`use`/`pub`
- [ ] **FR-012** Transform 26: `default export` to `pub fn`/`pub struct`
- [ ] **FR-012** Transform 27: Barrel files (`index.ts`) to `mod.rs`
- [ ] **FR-012** Transform 28: Namespace imports to module aliases
- [ ] **FR-012** Transform 29: Re-exports to `pub use`

## Tier 3 - AI Prompts

- [ ] **FR-013** Untranslatable constructs wrapped in structured prompt comments
- [ ] **FR-013** Prompt contains original TypeScript source
- [ ] **FR-013** Prompt contains surrounding Rust context
- [ ] **FR-013** Prompt contains list of attempted transforms
- [ ] **FR-013** Prompt contains suggested Rust pattern

## Pipeline Integration

- [ ] **FR-014** Stages execute in order: validate -> normalize (10) -> shadow -> tier1 -> tier2 -> tier3 -> cargo verify
- [ ] **FR-014** Each stage independently runnable
- [ ] **FR-014** Each stage independently testable with fixture inputs/outputs
- [ ] **FR-015** Translation report lists files processed
- [ ] **FR-015** Translation report lists transforms applied per file
- [ ] **FR-015** Translation report lists normalizers applied
- [ ] **FR-015** Translation report lists tier3 prompts generated
- [ ] **FR-015** Translation report includes cargo build/clippy/test results

## Success Criteria Verification

- [ ] **SC-001** Each normalizer has 8+ unit tests (normal, boundary, error)
- [ ] **SC-002** Shadow rewrite tested for all 3 libs, all 3 import forms
- [ ] **SC-003** Each of 29 transforms has 3+ unit tests
- [ ] **SC-004** 3+ real TS files (100+ lines each) translate to compilable Rust
- [ ] **SC-005** Profile-compliant input produces Rust needing zero manual edits
- [ ] **SC-006** 500-line file processes in under 30 seconds
- [ ] **SC-007** Tier3 prompts resolvable by LLM 80%+ of the time
- [ ] **SC-008** Profile rules catch 95%+ of untranslatable patterns (50+ test corpus)
