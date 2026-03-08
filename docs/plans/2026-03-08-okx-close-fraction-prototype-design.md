# OKX Close Fraction Prototype Design

## Goal

Prototype a narrow OKX adapter enhancement for NautilusTrader 1.224.0 that allows conditional protective orders in SWAP net-mode to use OKX `closeFraction` through the existing algo-order path, then validate the behavior with local tests and multiple live OKX demo trades on the server.

## Scope

The prototype remains adapter-local inside `nautilus_trader`.

- No Nautilus core order-model changes.
- No new generic order abstraction for attached TP/SL.
- No changes to the regular WebSocket entry-order path.
- The new behavior is opt-in through OKX-specific `SubmitOrder.params`.

## Recommended Approach

Use the existing OKX conditional/algo submission path and add support for venue field `closeFraction`.

The prototype will:

- read an adapter-specific execution param such as `close_fraction` from `SubmitOrder.params`
- apply it only for OKX conditional orders submitted via `place_algo_order(...)`
- serialize it to OKX request field `closeFraction`
- leave all existing behavior unchanged when the param is absent

This is preferred over a first-pass `attachAlgoOrds` implementation because it is materially smaller, stays adapter-local, and avoids introducing a larger design question around how attached venue-native TP/SL should map into Nautilus abstractions and reconciliation.

## Patch Surface

### Rust HTTP model

Extend `crates/adapters/okx/src/http/models.rs`:

- add optional `close_fraction` field to `OKXPlaceAlgoOrderRequest`
- serialize it as `closeFraction`

### Rust HTTP helper

Extend `crates/adapters/okx/src/http/client.rs`:

- add `close_fraction: Option<String>` to `place_algo_order_with_domain_types(...)`
- populate the new request field when present

### PyO3 binding

Extend `crates/adapters/okx/src/python/http.rs` and `nautilus_trader/core/nautilus_pyo3.pyi`:

- expose `close_fraction` on `OKXHttpClient.place_algo_order(...)`

### Python OKX execution adapter

Extend `nautilus_trader/adapters/okx/execution.py`:

- read `command.params.get("close_fraction")` on the algo-order path
- normalize accepted values to strings
- pass the value into `self._http_client.place_algo_order(...)`
- keep the regular WebSocket order path untouched

## Param Contract

For the prototype, keep the contract strict and explicit:

- accepted input: string or numeric values that can be safely normalized to string
- intended demo value: `"1"`
- no param: existing behavior
- invalid param: reject or deny early with a clear error rather than silently guessing

This keeps the prototype easy to reason about and easy to report upstream.

## Validation Bar

Prototype success requires both local and live validation.

### Local validation

1. Python adapter test proves `_submit_order(...)` forwards `params={"close_fraction": "1"}` into the HTTP algo path.
2. Rust serialization test proves the emitted request JSON includes `"closeFraction":"1"` when set and omits it when unset.

### Live validation

After local tests pass, validate on the server with multiple OKX demo/testnet trade cycles using a temporary branch build of the local prototype:

1. enter position
2. place protective conditional TP/SL using the close-fraction path
3. confirm the protective legs are accepted
4. flatten or let one leg trigger
5. repeat multiple times

The prototype is only considered successful if the demo runs avoid the current OKX rejection:

`OKX error 1: Reduce Only is not available.`

## Environment Strategy

Use a local-first workflow:

1. implement and test in the local checkout at `/home/nickonomic/code/nautilus_trader`
2. push the prototype branch to a temporary fork or remote branch
3. pull that branch onto the server in a disposable checkout
4. run multiple live OKX demo validations there

This gives a faster implementation loop locally while still meeting the requirement for realistic server-side demo validation.

## Out of Scope for This Prototype

- `attachAlgoOrds` support on the regular order-submit path
- broad venue-agnostic abstraction work
- documentation or API design beyond what is needed to explain the prototype result upstream

## Decision

Proceed with a `closeFraction` prototype first. If the prototype works in live OKX demo validation, comment on issue `#3693` with concrete evidence and then decide whether to upstream it as a PR.
