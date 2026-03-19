# Feature Specification: TypeScript-to-Rust Pipeline Milestone 1

**Feature Branch**: `001-ts-pipeline-phase1`
**Created**: 2026-03-13
**Status**: Draft
**Input**: User description: "Implement Milestone 1 of the TypeScript-to-Rust transformation pipeline - TypeScript-as-Rust profile definition, normalize transforms, shadow library integration, and Stage 1-2 deterministic transforms"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Normalize Idiomatic TypeScript to Profile-Compliant TypeScript (Priority: P1)

A developer writes idiomatic TypeScript (using CommonJS requires, let/var declarations, throw statements, try/catch blocks, mutable arrays, type assertions, mutable classes, default exports, and callback-based pagination). The pipeline runs the 10 Stage 0 normalizers in sequence, rewriting the source into the TS-as-Rust profile subset before any Rust translation begins.

**Why this priority**: Normalization is the foundation of the entire pipeline. Every downstream transform assumes input conforms to the TS-as-Rust profile. Without normalizers, the deterministic transforms produce incorrect or non-compilable output.

**Independent Test**: Feed a TypeScript file containing CommonJS requires, let/var declarations, throw statements, and try/catch blocks into the normalize pipeline. Verify the output uses only ESM imports, const declarations, Result returns, and match-based error handling.

**Acceptance Scenarios**:

1. **Given** a file with `const fs = require('fs')`, **When** commonjs-to-esm runs, **Then** the output is `import fs from 'fs'`
2. **Given** a file with `let x = 5; var y = 10;`, **When** let-var-to-const runs, **Then** the output uses `const x = 5; const y = 10;`
3. **Given** a function with `throw new Error('fail')`, **When** throw-to-err runs, **Then** the function returns `Err('fail')` using a Result wrapper type
4. **Given** a function returning `string | null`, **When** implicit-null-to-option runs, **Then** the return type becomes `Option<string>` using the shadow library Option type
5. **Given** a try/catch block, **When** try-catch-to-result runs, **Then** the block is rewritten to use Result chaining with `.mapErr()`
6. **Given** `arr.push(item); arr.splice(i, 1)`, **When** mutable-array-to-functional runs, **Then** output uses `[...arr, item]` and `arr.filter((_, idx) => idx !== i)`
7. **Given** `const x = value as string`, **When** type-assertion-to-explicit runs, **Then** output uses an explicit type guard or runtime check
8. **Given** a class with mutable fields, **When** mutable-class-to-readonly runs, **Then** all fields are marked `readonly` with builder-pattern mutation methods
9. **Given** `export default function main()`, **When** default-to-named-exports runs, **Then** output is `export function main()`
10. **Given** a callback-pagination pattern `fetchPage(cursor, callback)`, **When** callback-pagination-to-iterator runs, **Then** output uses an async generator yielding pages

---

### User Story 2 - Stage 1 Deterministic Type and Syntax Translate (Priority: P1)

A developer has profile-compliant TypeScript (post-normalization). The pipeline runs 11 Stage 1 transforms that perform direct syntactic translations from TypeScript constructs to their Rust equivalents: primitive types, const bindings, interfaces to structs, type aliases, enums, arrow functions to closures, imports to use statements, template literals to format!, array methods, string methods, and Record to HashMap.

**Why this priority**: Stage 1 transforms are 1:1 mechanical translations with no ambiguity. They form the bulk of translated output and are prerequisite for Stage 2 transforms that handle more complex patterns.

**Independent Test**: Pass a normalized TypeScript file containing interfaces, const declarations, enums, arrow functions, and template literals through the Stage 1 pipeline. Verify each construct maps to its Rust equivalent.

**Acceptance Scenarios**:

1. **Given** `const x: number = 42`, **When** type-primitives runs, **Then** output is `let x: i64 = 42` (number -> i64, string -> String, boolean -> bool)
2. **Given** `const name: string = "hello"`, **When** const transform runs, **Then** output is `let name: String = String::from("hello")`
3. **Given** `interface User { name: string; age: number; }`, **When** interface-to-struct runs, **Then** output is `struct User { name: String, age: i64 }`
4. **Given** `type ID = string`, **When** type-alias runs, **Then** output is `type ID = String;`
5. **Given** `enum Color { Red, Green, Blue }`, **When** enum transform runs, **Then** output is `enum Color { Red, Green, Blue }`
6. **Given** `const add = (a: number, b: number): number => a + b`, **When** arrow-to-closure runs, **Then** output is `let add = |a: i64, b: i64| -> i64 { a + b };`
7. **Given** `import { readFile } from 'fs'`, **When** import-to-use runs, **Then** output is `use fs::readFile;`
8. **Given** `` `Hello ${name}, you are ${age}` ``, **When** template-literal runs, **Then** output is `format!("Hello {}, you are {}", name, age)`
9. **Given** `arr.map(x => x * 2).filter(x => x > 5)`, **When** array-methods runs, **Then** output is `arr.iter().map(|x| x * 2).filter(|x| *x > 5).collect::<Vec<_>>()`
10. **Given** `str.includes("hello")`, **When** string-methods runs, **Then** output is `str.contains("hello")`
11. **Given** `const m: Record<string, number> = {}`, **When** record-to-hashmap runs, **Then** output is `let m: HashMap<String, i64> = HashMap::new()`

---

### User Story 3 - Shadow Library Rewrite for TS Module Compatibility (Priority: P2)

After normalization, the pipeline replaces TypeScript standard library and third-party module imports with shadow library equivalents that mirror Rust standard library semantics (via napi-rs bindings). This allows the normalized TypeScript to reference types like Option, Result, Vec, and HashMap that have direct Rust counterparts.

**Why this priority**: Shadow libraries bridge the semantic gap between TypeScript's standard library and Rust's. Without them, normalizers produce Option/Result types that have no TypeScript definition, and transforms cannot map them to Rust types.

**Independent Test**: Pass a normalized file that imports from `@ts-rust/std` (Option, Result) and `@ts-rust/collections` (Vec, HashMap) through the shadow rewrite. Verify imports are rewritten to the shadow library paths used during Rust code generation.

**Acceptance Scenarios**:

1. **Given** a file importing `Option` from the TS-as-Rust profile types, **When** shadow rewrite runs, **Then** the import path points to the shadow library implementing `Option<T>` with `Some(T)` and `None` variants
2. **Given** a file importing `Result` from the TS-as-Rust profile types, **When** shadow rewrite runs, **Then** the import path points to the shadow library implementing `Result<T, E>` with `Ok(T)` and `Err(E)` variants
3. **Given** a file importing `HashMap` from collections, **When** shadow rewrite runs, **Then** the import maps to `std::collections::HashMap` in the Rust output

---

### User Story 4 - Stage 2 Deterministic Pattern Translate (Priority: P2)

A developer has TypeScript code that uses async/await, classes, destructuring, nullish coalescing, object spread, optional chaining, Result chains, and try/catch. The pipeline applies 8 Stage 2 transforms that translate these patterns into idiomatic Rust equivalents.

**Why this priority**: Stage 2 transforms handle patterns that are common in real-world TypeScript but require structural changes (not just syntactic substitution). They depend on Stage 1 type mappings being in place.

**Independent Test**: Pass a file containing a TypeScript class with async methods, destructuring assignments, optional chaining, and nullish coalescing through the Stage 2 pipeline. Verify each pattern translates to idiomatic Rust.

**Acceptance Scenarios**:

1. **Given** `async function fetchData(): Promise<string> { return await fetch(url); }`, **When** async-await runs, **Then** output is `async fn fetch_data() -> Result<String, Box<dyn Error>> { fetch(url).await }`
2. **Given** a class `class User { constructor(public name: string) {} greet() { return this.name; } }`, **When** class-to-struct-impl runs, **Then** output is a `struct User` with `impl User` block containing methods
3. **Given** `const { name, age } = user;`, **When** destructuring runs, **Then** output is `let User { name, age } = user;`
4. **Given** `const val = x ?? defaultVal;`, **When** nullish-coalescing runs, **Then** output is `let val = x.unwrap_or(default_val);`
5. **Given** `const merged = { ...defaults, ...overrides };`, **When** object-spread runs, **Then** output uses a struct merge pattern or builder
6. **Given** `const name = user?.profile?.name;`, **When** optional-chaining runs, **Then** output is `let name = user.as_ref().and_then(|u| u.profile.as_ref()).map(|p| p.name.clone());`
7. **Given** a chain of Result operations, **When** result-chain runs, **Then** output uses the `?` operator for error propagation
8. **Given** a try/catch block with typed errors, **When** try-catch-to-match runs, **Then** output uses `match result { Ok(v) => ..., Err(e) => ... }`

---

### User Story 5 - End-to-End Pipeline Execution (Priority: P3)

A developer runs the full pipeline on a TypeScript source file. The pipeline executes all stages in order: normalize (Stage 0) -> shadow rewrite (Stage 0.5) -> Stage 1 transforms -> Stage 2 transforms. The final output is a syntactically valid Rust source file that compiles with `rustc` or `cargo build`.

**Why this priority**: E2E validation proves the pipeline stages compose correctly. Individual transforms may each be correct but produce incompatible intermediate states if not properly ordered.

**Independent Test**: Create a TypeScript file that exercises at least one construct from each stage. Run the full pipeline and verify the Rust output compiles without errors using `cargo check`.

**Acceptance Scenarios**:

1. **Given** a TypeScript file with CommonJS imports, let declarations, an interface, const bindings, template literals, and optional chaining, **When** the full pipeline runs, **Then** the output is a `.rs` file that passes `cargo check`
2. **Given** a multi-file TypeScript project with cross-module imports, **When** the pipeline runs on all files, **Then** each output `.rs` file has correct `use` statements referencing sibling modules
3. **Given** a TypeScript file with constructs outside the TS-as-Rust profile, **When** the pipeline runs, **Then** it emits a diagnostic error identifying the unsupported construct and its location

---

### Edge Cases

- What happens when a normalizer encounters code that is already profile-compliant? The normalizer must be idempotent and leave compliant code unchanged.
- How does the pipeline handle circular imports between TypeScript modules? The import-to-use transform must detect cycles and emit a diagnostic rather than generating invalid Rust `use` statements.
- What happens when type inference is required but no explicit type annotation exists? The type-primitives transform must either infer the Rust type from usage context or emit a `// TODO: type annotation required` comment.
- How does the pipeline handle union types that do not map to Option or Result? Unions like `string | number` must be flagged as unsupported with a diagnostic, since Rust has no implicit union type.
- What happens when a class uses inheritance (`extends`)? The class-to-struct-impl transform must detect inheritance and either flatten single-level inheritance into trait composition or emit a diagnostic for multi-level hierarchies.
- How does the pipeline handle generic type parameters? `interface Container<T>` must translate to `struct Container<T>` with appropriate trait bounds.
- What happens when async functions contain multiple await points with error handling? The async-await transform must correctly chain `?` operators and preserve error propagation order.

## Requirements *(mandatory)*

### Functional Requirements

#### TS-as-Rust Profile Definition

- **FR-001**: System MUST define the TS-as-Rust profile as a set of rules specifying which TypeScript constructs are permitted in profile-compliant code (stored in `profile/rules/`)
- **FR-002**: Profile MUST require all variable declarations to use `const` (no `let` or `var`)
- **FR-003**: Profile MUST require all imports to use ESM named imports (no default imports, no CommonJS require)
- **FR-004**: Profile MUST require all error handling to use Result<T, E> return types (no throw statements, no try/catch)
- **FR-005**: Profile MUST require all nullable values to use Option<T> (no implicit null/undefined unions)
- **FR-006**: Profile MUST require all arrays to be treated as immutable (functional operations only, no push/pop/splice)
- **FR-007**: Profile MUST require all class fields to be readonly with explicit type annotations
- **FR-008**: Profile MUST require explicit type annotations on all function parameters and return types
- **FR-009**: Profile MUST prohibit `any`, `unknown` (except in constrained generics), and type assertions

#### Stage 0 Normalizers

- **FR-010**: System MUST implement `commonjs-to-esm` normalizer that rewrites `require()` calls to ESM `import` statements and `module.exports` to `export`
- **FR-011**: System MUST implement `let-var-to-const` normalizer that rewrites all `let` and `var` declarations to `const`, introducing new bindings where reassignment occurs
- **FR-012**: System MUST implement `throw-to-err` normalizer that replaces `throw` statements with `return Err(...)` and updates function signatures to return `Result<T, E>`
- **FR-013**: System MUST implement `implicit-null-to-option` normalizer that replaces `T | null`, `T | undefined`, and `T | null | undefined` return/parameter types with `Option<T>`
- **FR-014**: System MUST implement `try-catch-to-result` normalizer that rewrites try/catch blocks into Result-based control flow using `.map()` and `.mapErr()`
- **FR-015**: System MUST implement `mutable-array-to-functional` normalizer that replaces array mutation methods (push, pop, splice, shift, unshift) with immutable alternatives (spread, filter, slice, concat)
- **FR-016**: System MUST implement `type-assertion-to-explicit` normalizer that replaces `as` type assertions with runtime type guards or explicit conversion functions
- **FR-017**: System MUST implement `mutable-class-to-readonly` normalizer that marks all class fields as `readonly` and introduces builder-pattern setter methods that return new instances
- **FR-018**: System MUST implement `default-to-named-exports` normalizer that converts `export default` to named exports
- **FR-019**: System MUST implement `callback-pagination-to-iterator` normalizer that converts callback-based pagination into async generator functions using `yield*`
- **FR-020**: Each normalizer MUST be idempotent: running a normalizer on already-compliant code MUST produce identical output

#### Stage 0.5 Shadow Library Rewrite

- **FR-021**: System MUST implement shadow rewrite rules (in `typescript-shadow-rewrite/rules/rewrite-ts-shadows.yml`) that map TS-as-Rust profile types to shadow library import paths
- **FR-022**: Shadow library MUST provide TypeScript type definitions for `Option<T>` with `Some(value)` and `None` constructors
- **FR-023**: Shadow library MUST provide TypeScript type definitions for `Result<T, E>` with `Ok(value)` and `Err(error)` constructors
- **FR-024**: Shadow library MUST provide TypeScript type definitions for `Vec<T>` wrapping Array with Rust-compatible method signatures (iter, map, filter, collect)
- **FR-025**: Shadow rewrite MUST replace standard TypeScript array/object types in import positions with shadow library equivalents

#### Stage 1 Transforms (Deterministic 1:1 Syntax Mapping)

- **FR-026**: System MUST implement `type-primitives` transform mapping: `number` -> `i64`, `string` -> `String`, `boolean` -> `bool`, `void` -> `()`, `never` -> `!`, `bigint` -> `i128`
- **FR-027**: System MUST implement `const` transform converting `const x: T = v` to `let x: RustT = v` with appropriate Rust type
- **FR-028**: System MUST implement `interface-to-struct` transform converting TypeScript interfaces to Rust structs with `#[derive(Debug, Clone)]`
- **FR-029**: System MUST implement `type-alias` transform converting `type X = Y` to `type X = Y;`
- **FR-030**: System MUST implement `enum` transform converting TypeScript string/numeric enums to Rust enums with `#[derive(Debug, Clone, PartialEq)]`
- **FR-031**: System MUST implement `arrow-to-closure` transform converting arrow function expressions to Rust closures
- **FR-032**: System MUST implement `import-to-use` transform converting ESM imports to Rust `use` statements
- **FR-033**: System MUST implement `template-literal` transform converting template literals to `format!()` macro calls
- **FR-034**: System MUST implement `array-methods` transform mapping: `.map()` -> `.iter().map().collect()`, `.filter()` -> `.iter().filter().collect()`, `.reduce()` -> `.iter().fold()`, `.find()` -> `.iter().find()`, `.some()` -> `.iter().any()`, `.every()` -> `.iter().all()`, `.forEach()` -> `.iter().for_each()`, `.includes()` -> `.contains()`
- **FR-035**: System MUST implement `string-methods` transform mapping: `.includes()` -> `.contains()`, `.startsWith()` -> `.starts_with()`, `.endsWith()` -> `.ends_with()`, `.indexOf()` -> `.find()`, `.slice()` -> index range syntax, `.toUpperCase()` -> `.to_uppercase()`, `.toLowerCase()` -> `.to_lowercase()`, `.trim()` -> `.trim()`, `.split()` -> `.split()`, `.replace()` -> `.replace()`, `.length` -> `.len()`
- **FR-036**: System MUST implement `record-to-hashmap` transform converting `Record<K, V>` to `HashMap<K, V>` with `use std::collections::HashMap`

#### Stage 2 Transforms (Deterministic Pattern Mapping)

- **FR-037**: System MUST implement `async-await` transform converting async functions to Rust async fn with appropriate Future return types
- **FR-038**: System MUST implement `class-to-struct-impl` transform converting classes to struct + impl blocks, mapping constructors to `fn new()`, methods to impl methods, and static methods to associated functions
- **FR-039**: System MUST implement `destructuring` transform converting object/array destructuring to Rust pattern matching in let bindings
- **FR-040**: System MUST implement `nullish-coalescing` transform converting `??` to `.unwrap_or()` or `.unwrap_or_else()`
- **FR-041**: System MUST implement `object-spread` transform converting `{...a, ...b}` to struct update syntax or manual field merge
- **FR-042**: System MUST implement `optional-chaining` transform converting `?.` chains to `.as_ref().and_then()` or `if let Some()` chains
- **FR-043**: System MUST implement `result-chain` transform converting sequential Result operations to use the `?` operator
- **FR-044**: System MUST implement `try-catch-to-match` transform converting remaining try/catch patterns to `match` expressions on Result values

#### Pipeline Orchestration

- **FR-045**: System MUST execute transforms in strict stage order: Stage 0 (normalize) -> Stage 0.5 (shadow rewrite) -> Stage 1 -> Stage 2
- **FR-046**: Within each stage, transforms MUST execute in a defined, deterministic order
- **FR-047**: System MUST preserve source location information through all transforms to enable diagnostic error reporting with original file/line references
- **FR-048**: System MUST emit structured diagnostic messages (errors, warnings) when encountering constructs that cannot be translated, including the source file path, line number, and a description of the unsupported pattern
- **FR-049**: System MUST produce output files with `.rs` extension in a configurable output directory, mirroring the input directory structure

### Key Entities

- **Transform**: A single AST-to-AST or AST-to-text rewriting function that takes a TypeScript AST node tree and produces a modified tree (normalize/stage1/stage2) or Rust source text (final emission). Each transform has a stage, an execution order within that stage, and a set of input/output contracts.
- **Profile**: A set of rules defining the TS-as-Rust subset. Used to validate that code conforms to the expected input shape before Stage 1+ transforms run. Stored as declarative rules in `profile/rules/`.
- **Shadow Library**: TypeScript type definition packages that provide Rust-equivalent types (Option, Result, Vec, HashMap) as TypeScript interfaces and classes. These exist only during the TS phase and are stripped during Rust emission.
- **Pipeline**: The ordered composition of all transforms across stages, taking a TypeScript source file as input and producing a Rust source file as output.
- **Diagnostic**: A structured message emitted when the pipeline encounters code it cannot translate, containing severity (error/warning), source location, and description.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: All 10 Stage 0 normalizers pass their individual test suites with at least 3 test cases each (positive, negative, idempotent)
- **SC-002**: All 11 Stage 1 transforms pass their individual test suites with at least 3 test cases each covering the documented type/syntax mappings
- **SC-003**: All 8 Stage 2 transforms pass their individual test suites with at least 2 test cases each covering the documented pattern mappings
- **SC-004**: Shadow rewrite rules correctly rewrite all TS-as-Rust profile type imports to shadow library paths in the test suite
- **SC-005**: The e2e test suite contains at least 3 TypeScript source files that pass through the full pipeline and produce Rust output that compiles with `cargo check` without errors
- **SC-006**: Pipeline emits actionable diagnostic errors for at least 5 known unsupported TypeScript patterns (e.g., union types other than Option, class inheritance beyond one level, dynamic property access, eval, Proxy)
- **SC-007**: Running any normalizer twice on the same input produces byte-identical output (idempotency verification)
- **SC-008**: Full pipeline execution on a 500-line TypeScript file completes in under 10 seconds on standard hardware

---

## v0.3 Addendum: Milestone 1 Track B Validation Gate

*Added 2026-03-16 to align with master spec v0.3 §9.4*

### 10-Node Translation Benchmark

This spec corresponds to **Milestone 1 Track B** (TypeScript Profile Bootstrap). The exit criterion is a 10-node benchmark:

- **10 n8n nodes** (Simple + Medium tier) must translate end-to-end: idiomatic TS → constrained TS (Normalize-Det/LLM) → Rust (S1/S2/S3), compile, tests pass.

### Dual Automation Rate Measurement

The benchmark MUST measure two automation rates **separately**:

1. **Normalize-Det/LLM automation rate** — what percentage of idiomatic TS code converts to constrained TS via deterministic transforms (Normalize-Det) without requiring LLM assistance (Normalize-LLM)
2. **Stages 1–3 translation rate** — what percentage of profile-compliant constrained TS converts to Rust via deterministic transforms (S1/S2) without requiring LLM-assisted stubs (S3)

### Fail Condition

> If Normalize-LLM on Simple-tier nodes requires **>40% LLM assistance**, the batch translation economics weaken significantly. In this case, invest in more deterministic Normalize-Det rules before proceeding to batch translation in Milestone 2.

### Additional Success Criteria

- **SC-009**: Normalize-LLM automation rate ≥60% on Simple-tier n8n nodes (≤40% requires LLM assistance)
- **SC-010**: Stages 1–2 deterministic translation rate ≥80% on profile-compliant TS input (stub-free output)
- **SC-011**: Stage 3 stub count ≤5 per translated node on average
- **SC-012**: All 10 benchmark nodes compile and pass tests
