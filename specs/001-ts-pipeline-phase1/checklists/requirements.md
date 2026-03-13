# Requirements Checklist: TypeScript-to-Rust Pipeline Phase 1

**Purpose**: Track implementation completeness of all functional requirements for the TS-to-Rust Phase 1 pipeline
**Created**: 2026-03-13
**Feature**: [spec.md](../spec.md)

## TS-as-Rust Profile Definition

- [ ] CHK001 Profile rules directory (`profile/rules/`) contains declarative rule definitions for all permitted TS constructs (FR-001)
- [ ] CHK002 Profile rule enforces const-only variable declarations, rejecting let and var (FR-002)
- [ ] CHK003 Profile rule enforces ESM named imports only, rejecting default imports and CommonJS require (FR-003)
- [ ] CHK004 Profile rule enforces Result<T, E> error handling, rejecting throw and try/catch (FR-004)
- [ ] CHK005 Profile rule enforces Option<T> for nullable values, rejecting T | null and T | undefined unions (FR-005)
- [ ] CHK006 Profile rule enforces immutable array operations, rejecting push/pop/splice/shift/unshift (FR-006)
- [ ] CHK007 Profile rule enforces readonly class fields with explicit type annotations (FR-007)
- [ ] CHK008 Profile rule enforces explicit type annotations on all function parameters and return types (FR-008)
- [ ] CHK009 Profile rule prohibits any, unknown (except constrained generics), and type assertions (FR-009)

## Tier 0 Normalizers

- [ ] CHK010 commonjs-to-esm normalizer rewrites require() to ESM import and module.exports to export (FR-010)
- [ ] CHK011 let-var-to-const normalizer rewrites let/var to const, introducing new bindings for reassignment (FR-011)
- [ ] CHK012 throw-to-err normalizer replaces throw with return Err() and updates function signatures (FR-012)
- [ ] CHK013 implicit-null-to-option normalizer replaces T|null, T|undefined unions with Option<T> (FR-013)
- [ ] CHK014 try-catch-to-result normalizer rewrites try/catch to Result-based flow with .map()/.mapErr() (FR-014)
- [ ] CHK015 mutable-array-to-functional normalizer replaces array mutations with immutable alternatives (FR-015)
- [ ] CHK016 type-assertion-to-explicit normalizer replaces `as` casts with type guards or conversions (FR-016)
- [ ] CHK017 mutable-class-to-readonly normalizer marks fields readonly, adds builder-pattern setters (FR-017)
- [ ] CHK018 default-to-named-exports normalizer converts export default to named exports (FR-018)
- [ ] CHK019 callback-pagination-to-iterator normalizer converts callbacks to async generators (FR-019)
- [ ] CHK020 All normalizers are idempotent: running on compliant code produces identical output (FR-020)

## Tier 0.5 Shadow Library Rewrite

- [ ] CHK021 Shadow rewrite rules YAML maps profile types to shadow library import paths (FR-021)
- [ ] CHK022 Shadow library provides Option<T> with Some(value) and None constructors (FR-022)
- [ ] CHK023 Shadow library provides Result<T, E> with Ok(value) and Err(error) constructors (FR-023)
- [ ] CHK024 Shadow library provides Vec<T> wrapping Array with Rust-compatible methods (FR-024)
- [ ] CHK025 Shadow rewrite replaces standard TS types in imports with shadow equivalents (FR-025)

## Tier 1 Transforms

- [ ] CHK026 type-primitives maps number->i64, string->String, boolean->bool, void->(), never->!, bigint->i128 (FR-026)
- [ ] CHK027 const transform converts const x: T = v to let x: RustT = v (FR-027)
- [ ] CHK028 interface-to-struct converts interfaces to structs with #[derive(Debug, Clone)] (FR-028)
- [ ] CHK029 type-alias converts type X = Y to type X = Y; (FR-029)
- [ ] CHK030 enum converts TS enums to Rust enums with #[derive(Debug, Clone, PartialEq)] (FR-030)
- [ ] CHK031 arrow-to-closure converts arrow functions to Rust closures (FR-031)
- [ ] CHK032 import-to-use converts ESM imports to Rust use statements (FR-032)
- [ ] CHK033 template-literal converts template literals to format!() macro calls (FR-033)
- [ ] CHK034 array-methods maps map/filter/reduce/find/some/every/forEach/includes to Rust iterator equivalents (FR-034)
- [ ] CHK035 string-methods maps includes/startsWith/endsWith/indexOf/slice/toUpperCase/toLowerCase/trim/split/replace/length to Rust equivalents (FR-035)
- [ ] CHK036 record-to-hashmap converts Record<K,V> to HashMap<K,V> with use std::collections::HashMap (FR-036)

## Tier 2 Transforms

- [ ] CHK037 async-await converts async functions to Rust async fn with Future return types (FR-037)
- [ ] CHK038 class-to-struct-impl converts classes to struct + impl blocks with fn new() constructors (FR-038)
- [ ] CHK039 destructuring converts object/array destructuring to Rust pattern matching (FR-039)
- [ ] CHK040 nullish-coalescing converts ?? to .unwrap_or() or .unwrap_or_else() (FR-040)
- [ ] CHK041 object-spread converts {...a, ...b} to struct update syntax or field merge (FR-041)
- [ ] CHK042 optional-chaining converts ?. chains to .as_ref().and_then() or if let Some() (FR-042)
- [ ] CHK043 result-chain converts sequential Result operations to use the ? operator (FR-043)
- [ ] CHK044 try-catch-to-match converts try/catch to match on Result values (FR-044)

## Pipeline Orchestration

- [ ] CHK045 Pipeline executes transforms in strict tier order: Tier 0 -> Tier 0.5 -> Tier 1 -> Tier 2 (FR-045)
- [ ] CHK046 Within each tier, transforms execute in a defined deterministic order (FR-046)
- [ ] CHK047 Source location information is preserved through all transforms for diagnostics (FR-047)
- [ ] CHK048 Structured diagnostics emitted for untranslatable constructs with file path, line, and description (FR-048)
- [ ] CHK049 Output files use .rs extension in configurable output directory mirroring input structure (FR-049)

## Success Criteria Verification

- [ ] CHK050 All 10 Tier 0 normalizers have test suites with >= 3 test cases each (SC-001)
- [ ] CHK051 All 11 Tier 1 transforms have test suites with >= 3 test cases each (SC-002)
- [ ] CHK052 All 8 Tier 2 transforms have test suites with >= 2 test cases each (SC-003)
- [ ] CHK053 Shadow rewrite rules pass test suite validating all profile type imports (SC-004)
- [ ] CHK054 E2E test suite has >= 3 TS files that produce Rust output passing cargo check (SC-005)
- [ ] CHK055 Pipeline emits diagnostics for >= 5 unsupported TS patterns (SC-006)
- [ ] CHK056 Idempotency verified: all normalizers produce byte-identical output on double run (SC-007)
- [ ] CHK057 Full pipeline on 500-line TS file completes in under 10 seconds (SC-008)

## Notes

- Check items off as completed: `[x]`
- Each CHK item references its corresponding FR or SC requirement from spec.md
- Items are ordered by pipeline execution tier for implementation sequencing
