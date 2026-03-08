# OKX Close Fraction Prototype Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a narrow OKX adapter prototype that forwards `SubmitOrder.params["close_fraction"]` into OKX algo-order field `closeFraction`, then validate it with local tests and multiple live OKX demo trades on the server.

**Architecture:** Keep the change adapter-local. Extend the OKX Rust request model and Python bindings, then have the Python OKX execution adapter read `command.params` on the existing conditional-order path and pass the value through unchanged when valid. Validate first with targeted Python and Rust tests, then build the package and run repeated live demo smokes from a disposable server checkout.

**Tech Stack:** Python adapter code, Rust OKX adapter crate, PyO3 bindings, `pytest`, `cargo test`, `uv`, OKX demo/testnet.

---

### Task 1: Add the failing Python adapter test

**Files:**
- Modify: `tests/integration_tests/adapters/okx/test_execution.py`
- Test: `tests/integration_tests/adapters/okx/test_execution.py`

**Step 1: Write the failing test**

Add a new async test near the existing `_submit_order` adapter tests that:

- builds an OKX execution client with mocked HTTP client
- creates a `StopMarketOrder` or `MarketIfTouched` order
- wraps it in `SubmitOrder(..., params={"close_fraction": "1"})`
- calls `await client._submit_order(command)`
- asserts `mock_http_client.place_algo_order.assert_awaited_once()`
- asserts the awaited call includes `close_fraction="1"`

Example assertion shape:

```python
call = http_client.place_algo_order.await_args
assert call.kwargs["close_fraction"] == "1"
```

**Step 2: Run test to verify it fails**

Run:

```bash
cd /home/nickonomic/code/nautilus_trader
uv run --active --no-sync pytest tests/integration_tests/adapters/okx/test_execution.py -k close_fraction -v
```

Expected: FAIL because `close_fraction` is not yet accepted by the mocked call path.

**Step 3: Commit the failing test**

```bash
cd /home/nickonomic/code/nautilus_trader
git add tests/integration_tests/adapters/okx/test_execution.py
git commit -m "test: cover OKX close fraction algo forwarding"
```

### Task 2: Add the failing Rust serialization test

**Files:**
- Modify: `crates/adapters/okx/src/http/models.rs`
- Test: `crates/adapters/okx/src/http/models.rs`

**Step 1: Write the failing test**

Add a Rust test next to the existing `OKXPlaceAlgoOrderRequest` serialization tests that builds a request with:

```rust
close_fraction: Some("1".to_string()),
```

and asserts:

```rust
assert!(json.contains("\"closeFraction\":\"1\""));
```

Also keep the existing omission behavior test when `None`.

**Step 2: Run test to verify it fails**

Run:

```bash
cd /home/nickonomic/code/nautilus_trader
cargo test -p nautilus-okx test_algo_order_request_serializes_close_fraction -- --exact
```

Expected: FAIL because the request struct does not yet have `close_fraction`.

**Step 3: Commit the failing Rust test**

```bash
cd /home/nickonomic/code/nautilus_trader
git add crates/adapters/okx/src/http/models.rs
git commit -m "test: cover OKX closeFraction serialization"
```

### Task 3: Implement Rust request and helper support

**Files:**
- Modify: `crates/adapters/okx/src/http/models.rs`
- Modify: `crates/adapters/okx/src/http/client.rs`

**Step 1: Extend the request model**

Add the new optional field to `OKXPlaceAlgoOrderRequest`:

```rust
#[serde(rename = "closeFraction", skip_serializing_if = "Option::is_none")]
pub close_fraction: Option<String>,
```

**Step 2: Extend the domain helper signature**

Update `place_algo_order_with_domain_types(...)` in `crates/adapters/okx/src/http/client.rs` to accept:

```rust
close_fraction: Option<String>,
```

**Step 3: Populate the request**

When constructing `OKXPlaceAlgoOrderRequest`, pass:

```rust
close_fraction,
```

and leave all other fields unchanged.

**Step 4: Run the Rust test to verify it passes**

Run:

```bash
cd /home/nickonomic/code/nautilus_trader
cargo test -p nautilus-okx test_algo_order_request_serializes_close_fraction -- --exact
```

Expected: PASS.

**Step 5: Commit the Rust implementation**

```bash
cd /home/nickonomic/code/nautilus_trader
git add crates/adapters/okx/src/http/models.rs crates/adapters/okx/src/http/client.rs
git commit -m "feat: add OKX closeFraction algo request support"
```

### Task 4: Extend the PyO3 surface

**Files:**
- Modify: `crates/adapters/okx/src/python/http.rs`
- Modify: `nautilus_trader/core/nautilus_pyo3.pyi`

**Step 1: Extend the Python binding signature**

Add `close_fraction=None` to the `#[pyo3(signature = ...)]` list for `place_algo_order`.

**Step 2: Extend the Rust binding function parameters**

Add:

```rust
close_fraction: Option<String>,
```

to `py_place_algo_order(...)`.

**Step 3: Pass the new argument through**

Update the `place_algo_order_with_domain_types(...)` call to forward `close_fraction`.

**Step 4: Update the Python stub**

Add:

```python
close_fraction: str | None = None,
```

to `OKXHttpClient.place_algo_order(...)` in `nautilus_trader/core/nautilus_pyo3.pyi`.

**Step 5: Commit the binding changes**

```bash
cd /home/nickonomic/code/nautilus_trader
git add crates/adapters/okx/src/python/http.rs nautilus_trader/core/nautilus_pyo3.pyi
git commit -m "feat: expose OKX closeFraction in Python bindings"
```

### Task 5: Implement Python adapter param handling

**Files:**
- Modify: `nautilus_trader/adapters/okx/execution.py`
- Test: `tests/integration_tests/adapters/okx/test_execution.py`

**Step 1: Add a small extraction helper or inline normalization**

In the OKX execution client, read:

```python
close_fraction = command.params.get("close_fraction") if command.params else None
```

Normalize accepted values:

- `str` -> use as-is
- `int`/`float` -> `str(value)`
- everything else -> deny or reject with a clear error

**Step 2: Pass the value only on the algo path**

Update the `self._http_client.place_algo_order(...)` call to include:

```python
close_fraction=close_fraction,
```

Do not change the WebSocket submit path.

**Step 3: Run the Python test to verify it passes**

Run:

```bash
cd /home/nickonomic/code/nautilus_trader
uv run --active --no-sync pytest tests/integration_tests/adapters/okx/test_execution.py -k close_fraction -v
```

Expected: PASS.

**Step 4: Commit the adapter change**

```bash
cd /home/nickonomic/code/nautilus_trader
git add nautilus_trader/adapters/okx/execution.py tests/integration_tests/adapters/okx/test_execution.py
git commit -m "feat: forward OKX close_fraction on algo orders"
```

### Task 6: Run targeted verification

**Files:**
- Modify: none
- Test: `tests/integration_tests/adapters/okx/test_execution.py`
- Test: `crates/adapters/okx/src/http/models.rs`

**Step 1: Run the targeted Python adapter tests**

Run:

```bash
cd /home/nickonomic/code/nautilus_trader
uv run --active --no-sync pytest tests/integration_tests/adapters/okx/test_execution.py -k "close_fraction or quote_quantity" -v
```

Expected: PASS.

**Step 2: Run the targeted Rust OKX tests**

Run:

```bash
cd /home/nickonomic/code/nautilus_trader
cargo test -p nautilus-okx test_algo_order_request_serializes_close_fraction -- --exact
```

Expected: PASS.

**Step 3: Build the package locally**

Run:

```bash
cd /home/nickonomic/code/nautilus_trader
make build-debug
```

Expected: build completes successfully with the updated PyO3 surface.

**Step 4: Commit any follow-up fixes**

```bash
cd /home/nickonomic/code/nautilus_trader
git add .
git commit -m "chore: fix OKX close fraction prototype verification issues"
```

Only create this commit if verification required follow-up changes.

### Task 7: Prepare a temporary remote branch for server validation

**Files:**
- Modify: none

**Step 1: Push the prototype branch**

Run one of:

```bash
cd /home/nickonomic/code/nautilus_trader
git remote -v
git push -u <your-fork-remote> nick/okx-close-fraction-prototype
```

or, if no fork exists yet, create one first and then push the same branch name.

**Step 2: Record the exact commit**

Run:

```bash
cd /home/nickonomic/code/nautilus_trader
git rev-parse HEAD
```

Expected: save the commit hash used for server validation.

### Task 8: Validate with repeated live OKX demo trades on the server

**Files:**
- Create or modify in disposable checkout only

**Step 1: Clone the prototype branch on the server**

Run on the server:

```bash
source ~/.ssh/agent-env && ssh -i ~/.ssh/id_ed25519 nickonomic@185.253.7.75
mkdir -p ~/code
cd ~/code
git clone <fork-url> nautilus_trader-close-fraction
cd nautilus_trader-close-fraction
git checkout nick/okx-close-fraction-prototype
```

**Step 2: Install dependencies and build**

Run:

```bash
cd ~/code/nautilus_trader-close-fraction
uv sync --active --all-groups --all-extras --inexact
make build-debug
```

Expected: successful local build on the server.

**Step 3: Run repeated demo trade cycles**

Use the same OKX demo/testnet credentials and smoke pattern that reproduced the issue:

1. open a SWAP net-mode position
2. submit protective `STOP_MARKET` / `MARKET_IF_TOUCHED` with `params={"close_fraction": "1"}`
3. confirm OKX accepts the legs
4. flatten or trigger the exit
5. repeat several times across both long and short cases if practical

**Step 4: Capture evidence**

For each cycle, record:

- entry side and symbol
- protective order params
- whether `Reduce Only is not available` appears
- whether OKX returns accepted algo IDs
- whether the position closed as intended

**Step 5: Commit or note any server-only fixes**

Only if the server run reveals a genuine prototype bug, bring the fix back to the local branch and repeat local verification before re-running the server smoke.

### Task 9: Decide whether to comment on issue #3693

**Files:**
- Modify: none

**Step 1: If live validation succeeds, prepare the issue comment**

Include:

- tested commit hash
- exact OKX demo environment
- number of successful demo cycles
- confirmation that `closeFraction` avoids the previous rejection
- note that the prototype is a narrow adapter-local change

**Step 2: If live validation fails, prepare a failure note instead**

Include:

- what was attempted
- what still failed
- whether the blocker is OKX behavior, adapter plumbing, or a larger API mismatch

**Step 3: Decide on next action**

Choose one:

- open a focused PR for the `closeFraction` path
- report prototype findings on the issue without PR
- abandon this path and revisit `attachAlgoOrds`
