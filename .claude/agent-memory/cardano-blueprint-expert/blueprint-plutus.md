---
name: blueprint-plutus
description: Cardano Blueprint Plutus/UPLC documentation — syntax, CEK machine semantics, built-ins, serialization
type: reference
---

# Cardano Blueprint — Plutus (UPLC)

## Source Files

- `src/plutus/README.md` — Overview and resources
- `src/plutus/syntax.md` — UPLC concrete syntax, versioning, de Bruijn indices
- `src/plutus/builtin.md` — Built-in types and functions
- `src/plutus/cek.md` — CEK machine operational semantics (full transition table)
- `src/plutus/serialization.md` — UPLC serialization

## What Plutus Is

Untyped Plutus Core (UPLC) — Cardano's on-chain smart contract execution language:
- Untyped lambda calculus (Turing-complete)
- Executed by validating nodes (CEK machine)
- Extended with built-in types and functions for efficiency
- Compiled to from higher-level languages (Plinth/Haskell, Aiken, etc.)

## UPLC Syntax

```text
L, M, N ∈ Term ::=
    x                           ; variable
  | (con T c)                   ; constant c with type T
  | (builtin b)                 ; built-in function
  | (lam x M)                   ; lambda abstraction
  | [M N]                       ; application
  | (delay M)                   ; delayed execution
  | (force M)                   ; force execution
  | (constr k M₁ … Mₘ)         ; constructor tag k, m args (m ≥ 0) — since v1.1.0
  | (case M N₁ … Nₘ)           ; case analysis, m alternatives — since v1.1.0
  | (error)                     ; error

P ∈ Program ::= (program v M)  ; versioned program
```

Constants carry type tags; variables do not. Constants must be of a built-in type.

## Version Numbers

`v` in `(program v M)` is the **Plutus Core language version** (x.y.z form).

Distinct from **Plutus ledger language version** (PlutusV1, V2, V3).

For details: https://plutus.cardano.intersectmbo.org/docs/essential-concepts/versions

## De Bruijn Indices

Variables can be textual strings OR de Bruijn indices. **Serialized scripts always use de Bruijn indices.**

- Binder `x` in `(lam x M)` is irrelevant with de Bruijn; use `0` conventionally
- Implementation recommendation: use de Bruijn indices for CEK machine

## Constructor Tags

- Principle: any natural number
- Practice: limited to 64 bits (enforced in binary format)
- Haskell uses `Word64`

## CEK Machine

### Key Structures

```text
Σ ∈ State ::=
    s; ρ ⊳ M    ; Computing M under env ρ with stack s
  | s ⊲ V       ; Returning value V to stack s
  | ⬥           ; Error state
  | ◻V          ; Final state (success) with value V

s ∈ Stack ::= f*   ; zero or more frames

V ∈ CEK value ::=
    〈con T c〉           ; constant
  | 〈delay M ρ〉         ; delayed computation + env
  | 〈lam x M ρ〉         ; lambda + env
  | 〈constr i V*〉        ; constructor, all args are values
  | 〈builtin b V* η〉     ; builtin, partial application with expected args

ρ ∈ Environment ::= [] | ρ[x ↦ V]

f ∈ Frame ::=
    (force _)              ; awaiting forced value
  | [_ (M, ρ)]             ; application, awaiting function (arg=term)
  | [_ V]                  ; application, awaiting function (arg=value)
  | [V _]                  ; application, awaiting argument
  | (constr i V* _ (M*, ρ)) ; constructor, awaiting argument
  | (case _ (M*, ρ))        ; case, awaiting scrutinee
```

### Evaluation

Start state: `[]; [] ⊳ M` for program term M.

Terminal states:
- `◻V` → success, return term corresponding to V
- `⬥` or stuck → failure

### Transition Rules (key subset)

| Rule | From State | To State | Condition |
|------|-----------|----------|-----------|
| 1 | `s; ρ ⊳ x` | `s ⊲ ρ[x]` | x bound in ρ |
| 2 | `s; ρ ⊳ (con T c)` | `s ⊲ 〈con T c〉` | |
| 3 | `s; ρ ⊳ (lam x M)` | `s ⊲ 〈lam x M ρ〉` | |
| 4 | `s; ρ ⊳ (delay M)` | `s ⊲ 〈delay M ρ〉` | |
| 5 | `s; ρ ⊳ (force M)` | `(force _)·s; ρ ⊳ M` | |
| 6 | `s; ρ ⊳ [M N]` | `[_ (N, ρ)]·s; ρ ⊳ M` | |
| 11 | `s; ρ ⊳ (error)` | `⬥` | |
| 12 | `[] ⊲ V` | `◻V` | |

Full table in `src/plutus/cek.md` — 30+ rules covering all term forms, stack frames, and builtin applications.

## Cost Model

Two kinds of cost (briefly mentioned in README but not detailed in Blueprint):
1. **Step charges**: fixed charge per CEK machine step (variable lookup, lambda, etc.)
2. **Builtin function charges**: calculated by costing function based on argument sizes

Costing functions derived empirically (R statistical fitting on benchmarks).

## Resources (from Blueprint)

- **Plinth User Guide**: https://plutus.cardano.intersectmbo.org/docs/ — covers UPLC, essential concepts, glossary
- **Plutus Core Spec**: https://plutus.cardano.intersectmbo.org/resources/plutus-core-spec.pdf — authoritative reference
- **Haskell implementation**: https://plutus.cardano.intersectmbo.org/haddock/latest/plutus-core/UntypedPlutusCore-Evaluation-Machine-Cek.html
- **CEK machine Wikipedia**: https://en.wikipedia.org/wiki/CEK_Machine

## Blueprint Coverage Status

- Syntax: fully documented
- CEK machine transitions: fully documented with complete table
- Built-in types/functions: documented in `builtin.md` (contents not fully captured)
- Serialization: documented in `serialization.md` (contents not fully captured)
- Cost model: only mentioned conceptually, not detailed

## Gaps in Blueprint

- No script context format documentation (what's passed to scripts)
- No Plutus version compatibility matrix (language version vs ledger version)
- No documentation of execution budget limits per protocol parameter
- No documentation of how scripts are triggered (spending, minting, staking, voting)
- Cost model costing functions not documented
