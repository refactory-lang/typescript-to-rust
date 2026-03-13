# Feature Specification: Phase 1 TS-to-Rust Pipeline

**Feature Branch**: `001-ts-pipeline-phase1`
**Created**: 2026-03-13
**Status**: Draft

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Translate a TypeScript file to Rust via the full pipeline (Priority: P1)

As a developer, I want to run the full translation pipeline on a TypeScript-as-Rust profile-compliant TypeScript file and receive compilable Rust output, so that I can migrate TypeScript codebases to Rust without manual rewriting.

**Acceptance Scenarios:**

1. **Given** a TypeScript file conforming to the TS-as-Rust profile (no `any`, no dynamic property access, no prototype manipulation), **when** I run profile validation, **then** it passes with zero violations.
2. **Given** a validated TypeScript file, **when** the 10 normalizers run, **then** TS-specific idioms (Promise, interface, enum, class, union types, optional chaining, nullish coalescing, type assertions, generics, decorators) are rewritten to intermediate forms closer to Rust semantics.
3. **Given** normalized TypeScript, **when** the shadow rewrite runs for the 3 TS shadow libraries, **then** all shadow library imports are rewritten to their Rust crate path equivalents.
4. **Given** shadow-rewritten TypeScript, **when** all 29 tier1/tier2 transforms run, **then** TypeScript syntax, types, error handling, control flow, and module structure are converted to idiomatic Rust.
5. **Given** the final Rust output, **when** I run `cargo build`, `cargo clippy`, and `cargo test`, **then** all three pass with zero errors and zero warnings.

### User Story 2 - Validate TS-as-Rust profile compliance (Priority: P1)

As a developer, I want to validate that my TypeScript source files conform to the TS-as-Rust profile before running the translation pipeline, so that I catch untranslatable patterns early.

**Acceptance Scenarios:**

1. **Given** a TypeScript file that uses `any` type annotations, **when** I run profile validation (ast-grep rules), **then** it reports a violation with the file path, line number, and rule name (`no-any-type`).
2. **Given** a TypeScript file that uses dynamic property access (`obj[dynamicKey]` where `dynamicKey` is not a literal), **when** I run profile validation, **then** it reports a violation (`no-dynamic-property-access`).
3. **Given** a TypeScript file that uses `eval()`, `Function()` constructor, or `with` statements, **when** I run profile validation, **then** it reports violations for each occurrence.
4. **Given** a TypeScript file using only profile-compliant constructs (typed interfaces, typed functions, enums, Result-pattern error handling, explicit generics), **when** I run profile validation, **then** it exits with code 0 and reports no violations.
5. **Given** a TypeScript project directory, **when** I run profile validation recursively, **then** it processes all `.ts` files (excluding `node_modules` and `dist`) and reports a summary of violations.

### User Story 3 - Debug translation failures (Priority: P2)

As a developer, I want clear diagnostics when a translation step fails, so that I can fix my source code or report a transform bug.

**Acceptance Scenarios:**

1. **Given** a TypeScript file that fails during a normalizer, **when** I run the pipeline with verbose output, **then** I see which normalizer failed, the AST node it failed on, and the original TypeScript source at that location.
2. **Given** a translation that produces Rust code failing `cargo build`, **when** I inspect the pipeline output, **then** each output section is annotated with the transform that produced it.
3. **Given** a construct that no transform can handle, **when** the pipeline completes, **then** it is wrapped in a structured prompt comment with the original TypeScript, the reason it was not translated, and a suggested Rust pattern.

### Edge Cases

1. **Empty file**: An empty TypeScript file produces a valid empty Rust file or minimal stub.
2. **Re-exports**: `export { Foo } from './bar'` chains are resolved to Rust `pub use` statements with correct module paths.
3. **Union types**: `string | number` maps to a Rust enum with `String` and `i64` variants, or to a trait object where appropriate.
4. **Generic constraints**: `<T extends SomeTrait>` maps to Rust `<T: SomeTrait>` with correct trait bound syntax.
5. **Async/await**: `async function` and `await` expressions map to Rust `async fn` and `.await` with appropriate `Future` traits.
6. **Index signatures**: `{ [key: string]: Value }` maps to `HashMap<String, Value>`.
7. **Tuple types**: `[string, number, boolean]` maps to Rust `(String, i64, bool)`.
8. **Intersection types**: `TypeA & TypeB` maps to a Rust struct implementing both traits or containing both sets of fields.
9. **Reserved word collisions**: TypeScript identifiers that collide with Rust reserved words (`type`, `match`, `loop`, `move`, `ref`, `self`, `crate`, `mod`) are prefixed with `r#`.

## Requirements *(mandatory)*

### Functional Requirements

**FR-001**: The TS-as-Rust profile shall be defined as a set of ast-grep rules that reject: `any` type, `unknown` type used without narrowing, dynamic property access, `eval()`, `Function()` constructor, `with` statements, prototype manipulation, `arguments` object usage, and implicit `this` binding.

**FR-002**: The `normalize/promise` normalizer shall convert `Promise<T>` return types to `Result<T, E>` patterns, `async/await` to Rust async equivalents, and `.then()/.catch()` chains to `?` operator chains.

**FR-003**: The `normalize/interface` normalizer shall convert TypeScript `interface` declarations to Rust `trait` definitions (when used polymorphically) or `struct` definitions (when used as data shapes), based on usage analysis.

**FR-004**: The `normalize/enum` normalizer shall convert TypeScript `enum` declarations to Rust `enum` with explicit discriminants for numeric enums and string-valued variants for string enums.

**FR-005**: The `normalize/class` normalizer shall convert TypeScript `class` declarations to Rust `struct` + `impl` blocks, with `constructor` mapping to `fn new()`, methods mapping to `impl` methods, and `extends` mapping to trait composition or struct embedding.

**FR-006**: The `normalize/union-type` normalizer shall convert TypeScript union types (`A | B | C`) to Rust enums with one variant per union member, generating appropriate `From` trait implementations.

**FR-007**: The `normalize/optional-chaining` normalizer shall convert `?.` optional chaining to Rust `.as_ref().map(|x| x.field)` or `?` operator chains, and `??` nullish coalescing to `.unwrap_or()` or `.unwrap_or_else()`.

**FR-008**: The `normalize/type-assertion` normalizer shall convert `value as Type` assertions to Rust `.into()`, `.try_into()`, or explicit cast operations depending on the types involved.

**FR-009**: The `normalize/generics` normalizer shall convert TypeScript generic type parameters and constraints (`<T extends U>`) to Rust generic parameters with trait bounds (`<T: U>`), including `where` clauses for complex bounds.

**FR-010**: The `normalize/decorator` normalizer shall convert TypeScript decorators to Rust attribute macros where a direct mapping exists, and route unmappable decorators to tier3 prompt markers.

**FR-011**: The shadow rewrite package shall rewrite imports for the 3 supported TS shadow libraries. Each shadow library mirrors a subset of Node.js or TS runtime APIs with Rust crate equivalents (e.g., `fs` operations to `std::fs`, `path` operations to `std::path`, `crypto` to a Rust crypto crate).

**FR-012**: The 29 tier1/tier2 transforms shall cover the following categories:
- **Tier 1 - Syntax** (transforms 1-8): braces/semicolons (already present in TS, normalize), `function`/`const fn` to `fn`, `let`/`const` to `let`/`let mut`, arrow functions to closures, template literals to `format!()`, destructuring to Rust pattern matching, spread operator to iterator chains, type annotations syntax.
- **Tier 1 - Errors** (transforms 9-12): `throw` to `return Err()`, `try/catch` to `match` on `Result`, error class hierarchies to error enums with `thiserror`, `finally` blocks to `Drop` or explicit cleanup.
- **Tier 1 - Types** (transforms 13-18): `string`->`String`, `number`->`i64`/`f64` (based on usage analysis), `boolean`->`bool`, `Array<T>`->`Vec<T>`, `Map<K,V>`->`HashMap<K,V>`, `Set<T>`->`HashSet<T>`.
- **Tier 2 - Control** (transforms 19-24): `for...of` to `for x in iter`, `for...in` to `for (k, v) in map.iter()`, `switch/case` to `match`, `if/else` chains to `match` where applicable, `while`/`do-while` to `loop`/`while`, iterator methods (`.map`/`.filter`/`.reduce`) to Rust iterator chains.
- **Tier 2 - Modules** (transforms 25-29): `import/export` to `mod`/`use`/`pub`, `default export` to `pub fn`/`pub struct`, barrel files (`index.ts`) to `mod.rs`, namespace imports to module aliases, re-exports to `pub use`.

**FR-013**: Tier 3 prompt markers shall be emitted for any construct that no mechanical transform can handle. Each marker shall contain: the original TypeScript source, the surrounding Rust context, the list of transforms that were attempted, and a suggested Rust pattern.

**FR-014**: The end-to-end pipeline shall execute stages in order: profile validation -> normalizers (10 stages) -> shadow rewrite -> tier1 transforms -> tier2 transforms -> tier3 prompts -> cargo build verification. Each stage shall be independently runnable.

**FR-015**: The pipeline shall produce a translation report (JSON) listing: files processed, transforms applied per file, normalizers applied, tier3 prompts generated, and cargo build/clippy/test results.

### Key Entities

| Entity | Description |
|---|---|
| **TS-as-Rust Profile** | A set of ast-grep rules defining the subset of TypeScript that is mechanically translatable to Rust. Files must pass validation before entering the pipeline. |
| **Normalizer** | One of 10 pre-processing transforms that rewrite TS-specific idioms (Promise, interface, enum, class, union types, optional chaining, nullish coalescing, type assertions, generics, decorators) to intermediate forms. |
| **Shadow Library** | A TypeScript package mirroring a runtime API but mapping to a Rust crate. 3 shadow libraries supported. |
| **Transform** | A codemod rule (ast-grep or programmatic) converting one syntactic or semantic pattern from TypeScript to Rust. 29 transforms across tier1 and tier2. |
| **Tier** | Ordering level: Tier 0: normalizers, Tier 0.5: shadow rewrite, Tier 1: syntax/errors/types, Tier 2: control/modules, Tier 3: AI prompts. |
| **Pipeline** | The orchestrator running all tiers in sequence on input `.ts` files, producing `.rs` output and a translation report. |
| **Translation Report** | Structured JSON summarizing transforms applied, successes, failures, and tier3 prompt locations. |

## Success Criteria *(mandatory)*

### Measurable Outcomes

**SC-001**: All 10 normalizers pass their unit test suites with at least 8 test cases each covering normal, boundary, and error conditions.

**SC-002**: The shadow rewrite package correctly rewrites imports for all 3 supported TS shadow libraries, verified by a test suite covering `import X from 'lib'`, `import { Y } from 'lib'`, and `import * as Z from 'lib'` forms.

**SC-003**: All 29 tier1/tier2 transforms pass their unit test suites with at least 3 test cases each.

**SC-004**: At least 3 real-world TypeScript files (minimum 100 lines each, using interfaces, classes, async/await, generics, and error handling) translate end-to-end to Rust that passes `cargo build` with zero errors.

**SC-005**: Translated Rust output from profile-compliant TypeScript input requires zero manual editing to pass `cargo build` and `cargo clippy -- -D warnings`.

**SC-006**: The pipeline processes a 500-line TypeScript file in under 30 seconds on a standard development machine.

**SC-007**: Tier 3 prompt markers contain sufficient context that an LLM can resolve them to valid Rust in a single pass at least 80% of the time.

**SC-008**: The ast-grep profile rules catch at least 95% of untranslatable TypeScript patterns, measured against a curated test corpus of 50+ violation examples.
