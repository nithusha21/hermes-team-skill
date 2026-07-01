# TASKS.md — example

A task file is a list of independently-implementable units. Each task declares an ID, a
title, the files it touches, its dependencies, verification commands, and a commit message.
The skill reads `deps` to order tasks into waves.

> This example happens to be Rust, but the skill is language-agnostic: each task supplies its
> own `Verify:` commands, and the skill auto-detects the project's build/test commands
> (see Step 7 of the skill). Swap in `go test`, `pytest`, `npm test`, etc. as appropriate.

---

## A1 — Add `Money` value type

- **Files:** CREATE `src/money.rs`, MODIFY `src/lib.rs`
- **Deps:** none
- **Details:** Add a `Money { cents: i64, currency: Currency }` struct with checked
  `add`/`sub` returning `Result<Money, MoneyError>` on currency mismatch. Derive
  `Debug, Clone, Copy, PartialEq, Eq`.
- **Verify:**
  ```
  cargo test money
  cargo clippy -- -D warnings
  ```
- **Commit:** `feat(A1): add Money value type`

## A2 — Add `Currency` enum

- **Files:** CREATE `src/currency.rs`, MODIFY `src/lib.rs`
- **Deps:** none
- **Details:** `enum Currency { USD, EUR, GBP }` with `symbol()` and `FromStr`.
- **Verify:**
  ```
  cargo test currency
  ```
- **Commit:** `feat(A2): add Currency enum`

## B1 — Wallet balance using Money

- **Files:** CREATE `src/wallet.rs`, MODIFY `src/lib.rs`
- **Deps:** A1, A2
- **Details:** `Wallet` holding a `Money` balance; `deposit`/`withdraw` reject currency
  mismatches and overdrafts.
- **Verify:**
  ```
  cargo test wallet
  ```
- **Commit:** `feat(B1): add Wallet with Money-backed balance`
```

Waves derived from the above:

- **Wave 1:** A1, A2 (no deps — run in parallel)
- **Wave 2:** B1 (depends on A1 + A2)
