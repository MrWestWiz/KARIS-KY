# Escrow Funding Parameters & Liability Floor Invariant

> **Contract version:** `SCHEMA_VERSION = 6` | `INTERFACE_VERSION = 1`  
> **Target audience:** Operators, integrators, protocol developers, and auditors

This document describes the relationship between funding target updates (`update_funding_target`) and 
dust sweep restrictions (`sweep_terminal_dust`), emphasizing the **shared liability floor invariant** 
that ensures consistency across both endpoints.

---

## Overview: The Funding Target and Dust Sweep Ecosystem

The escrow contract manages two distinct phases of fund movement:

1. **Funding Phase** (status 0 = open, status 1 = funded): Investors deposit principal; the target may be adjusted.
2. **Terminal Phase** (status 4 = cancelled): Investors can refund; dust sweeps are gated by outstanding liabilities.

The contract maintains a critical invariant: **the liability floor must remain consistent across both phases**.

---

## Key Concepts

### Funded Amount (`funded_amount`)

The **cumulative principal** deposited by all investors across all `fund` and `fund_with_commitment` calls.
Once an investor contributes, the escrow's `funded_amount` increases immediately.

| State | Funded Amount | Mutable? |
|-------|---------------|----------|
| Open (0) | Grows as investors fund | Increases only on each deposit |
| Funded (1) | Fixed at settlement/cancellation time | Immutable from status 1 onward |
| Cancelled (4) | Same as at cancellation | Immutable |

### Distributed Principal (`distributed_principal`)

The **cumulative principal refunded to investors** via the `refund()` endpoint in a cancelled escrow.
Only meaningful in status 4 (cancelled) state; remains 0 in settled (2) or withdrawn (3) states.

| Endpoint | Increments `distributed_principal`? |
|----------|--------------------------------------|
| `refund()` (status 4 only) | Yes — by investor's contribution |
| `settle()` (status 2) | No — disbursement is off-chain |
| `withdraw()` (status 3) | No — disbursement is off-chain |
| `sweep_terminal_dust()` | No — but reads it to enforce floor |

### Outstanding Investor Liabilities

The amount of funded principal that investors have **not yet received**:

```
outstanding = funded_amount - distributed_principal
```

In status 4 (cancelled):
- If `distributed_principal = 0`, **all** funded amount is outstanding.
- If `distributed_principal = funded_amount`, **no** outstanding liabilities (all refunds complete).
- Otherwise, **partial** outstanding liabilities exist.

---

## The Liability Floor Invariant

### Definition

The contract enforces a single, unified **liability floor** across both funding and terminal phases:

```
liability_floor = funded_amount - distributed_principal
```

**Invariant:** 
- `update_funding_target` must reject targets **below** `liability_floor`.
- `sweep_terminal_dust` must reject sweeps that would reduce balance **below** `liability_floor`.

### Why This Matters

If we allowed a target to be set below `liability_floor`, the protocol could inadvertently
create a scenario where the balance cannot legally cover all outstanding investor claims,
violating the promise that refunds will always succeed.

### Status-Specific Behavior

#### Status 0 (Open)

- `distributed_principal = 0` (always; refunds haven't begun)
- `liability_floor = funded_amount - 0 = funded_amount`
- **`update_funding_target` check:** `new_target >= funded_amount`
- Dust sweep: Not applicable (not terminal)

#### Status 1 (Funded)

- `distributed_principal = 0` (always; refunds haven't begun)
- `liability_floor = funded_amount - 0 = funded_amount`
- **`update_funding_target`:** Not callable (error `TargetUpdateNotOpen = 73`)
- Dust sweep: Not applicable (not terminal)

#### Status 4 (Cancelled)

- `distributed_principal >= 0` (may have partial or full refunds)
- `liability_floor = funded_amount - distributed_principal` (can be > 0 if refunds incomplete)
- **`update_funding_target`:** Not callable (error `TargetUpdateNotOpen = 73`)
- **`sweep_terminal_dust` check:** `balance_after_sweep >= liability_floor` or error `SweepExceedsLiabilityFloor = 42`

---

## Cross-Reference: `update_funding_target` and `sweep_terminal_dust`

### `update_funding_target` (Admin-Only)

**Location:** `src/lib.rs::update_funding_target`

**Callable in:** Status 0 (open) only.

**Validation:**
```rust
// Liability floor: target must remain >= funded_amount - distributed_principal.
// In status 0 (open), distributed_principal is always 0, so this reduces to >= funded_amount.
// This invariant aligns with the floor used by sweep_terminal_dust in cancelled (status 4) escrows.
let distributed: i128 = env.storage().instance().get(&DataKey::DistributedPrincipal).unwrap_or(0);
let liability_floor = escrow.funded_amount.saturating_sub(distributed);
ensure(&env, new_target >= liability_floor, EscrowError::TargetBelowFundedAmount);
```

**Effect:** Prevents lowering the target below `funded_amount`, ensuring any future cancellation + refund cycle
has sufficient balance to cover all outstanding claims.

### `sweep_terminal_dust` (Treasury-Only)

**Location:** `src/lib.rs::sweep_terminal_dust`

**Callable in:** Status 2 (settled), 3 (withdrawn), 4 (cancelled), 5 (archived).

**Validation (status 4 only):**
```rust
// Liability floor (cancelled escrows only): sweep must not reduce the balance below
// principal still owed to investors who have not yet called refund().
// outstanding = funded_amount - distributed_principal
// Invariant: balance - sweep_amt >= outstanding
if escrow.status == 4 {
    let distributed: i128 = env.storage().instance().get(&DataKey::DistributedPrincipal).unwrap_or(0);
    let outstanding = escrow.funded_amount.saturating_sub(distributed);
    let balance_after_sweep = balance - sweep_amt;
    ensure(&env, balance_after_sweep >= outstanding, EscrowError::SweepExceedsLiabilityFloor);
}
```

**Effect:** Prevents sweeping treasury dust in a way that would strand outstanding investor refund obligations.

---

## Scenario: Partial Refund Then Dust Sweep

This scenario illustrates how the invariant protects investors:

**Setup:**
- Invoice target: 10,000 tokens
- Investor A: 5,000 tokens
- Investor B: 5,000 tokens
- Balance: 10,000 tokens (all funded)
- Escrow status: 4 (cancelled)

**Step 1: Investor A refunds**
- Call `refund(&investor_a)`
  - `funded_amount = 10_000`
  - `distributed_principal = 5_000`
  - `outstanding = 10_000 - 5_000 = 5_000` ← Investor B still owed
  - `balance = 10_000 - 5_000 = 5_000`

**Step 2: Attempt to sweep excess dust**
- Call `sweep_terminal_dust(1_000)` on treasury
  - Dust sweep floor: `balance_after_sweep = 5_000 - 1_000 = 4_000`
  - Outstanding: `5_000`
  - Check: `4_000 >= 5_000`? **NO** → Reject with `SweepExceedsLiabilityFloor = 42`
  - Treasury cannot sweep yet

**Step 3: Investor B refunds**
- Call `refund(&investor_b)`
  - `funded_amount = 10_000`
  - `distributed_principal = 10_000`
  - `outstanding = 0` ← No liabilities remain
  - `balance = 0`

**Step 4: Retry dust sweep**
- Since `distributed_principal == funded_amount`, `outstanding = 0`
- Sweep is now **allowed** (no constraints)

---

## Scenario: Pre-Cancellation Target Update

This scenario shows why `update_funding_target` must respect the liability floor:

**Setup:**
- Invoice target: 10,000 tokens
- Investor A: 3,000 tokens
- Investor B: 2,000 tokens
- `funded_amount = 5_000`
- Balance: 5,000 tokens
- Escrow status: 0 (open)

**Attempt: Lower target below funded amount**
```
admin.update_funding_target(3_000)  // Try to set target to 3,000
```

**Check:**
- `liability_floor = funded_amount - distributed_principal = 5_000 - 0 = 5_000`
- `new_target = 3_000`
- Condition: `3_000 >= 5_000`? **NO** → Reject with `TargetBelowFundedAmount = 74`

**Why reject?**
- If we allowed this, the target becomes 3,000 but balance holds 5,000.
- Later, if the escrow is cancelled, Investors A and B have claims for 5,000 total.
- The balance (5,000) would be just enough to refund them.
- But the target (3,000) would be misleading to future participants.
- The invariant prevents this inconsistency.

---

## Common Questions

### Q: Why does `update_funding_target` read `distributed_principal` if it's always 0 in status 0?

**A:** For **documentation and consistency**. The check explicitly computes `liability_floor` the same way as `sweep_terminal_dust`, making the shared invariant visible in the code and future-proofing against refactoring. It also makes the intent clear: we are enforcing a **floor based on liabilities**, not just a business rule.

### Q: Can an admin lower the target after cancellation?

**A:** No. `update_funding_target` requires status 0 (open). Once the escrow is funded (status 1) or cancelled (status 4), the target becomes immutable. This is intentional: the target is a governance lever for the initial open period only.

### Q: What if balance dips below `liability_floor` due to external loss (e.g., token accounting error)?

**A:** The contract assumes the token contract is non-defective. If a token has a bug (e.g., silent loss), Soroban's balance checks would detect it at transfer time and reject the transaction. The escrow contract itself does not recompute balances; it relies on SEP-41 contract invariants.

### Q: Can I set the target to exactly equal `funded_amount`?

**A:** Yes. `update_funding_target(funded_amount)` is valid. The check is `new_target >= liability_floor`, not `>`.

### Q: How does `sweep_terminal_dust` apply to settled (status 2) or withdrawn (status 3) escrows?

**A:** The liability floor check **only applies to cancelled (status 4) escrows**. In settled or withdrawn states:
- Disbursement is off-chain.
- `distributed_principal` remains 0 (unused).
- No on-chain refund mechanism exists.
- Treasury can sweep any excess balance without constraints.

This is by design: in settled/withdrawn cases, the SME (issuer) has already received the full amount,
and dust is truly residual (rounding or yield distributions).

### Q: Can a dust sweep strand investors if `distributed_principal` is not properly tracked?

**A:** No. If a bug caused `distributed_principal` to be undercounted, the invariant would still protect investors:
- The check would compute a **lower** `outstanding` than actual.
- The check would **allow** a sweep that should have been blocked.
- **However**, the token transfer would succeed only if balance actually existed.
- Investors calling `refund()` would then fail at the token transfer step.

To prevent this, the contract ensures `distributed_principal` is **incremented atomically** with each successful `refund()` transfer. The two are always in sync.

---

## Implementation Notes for Developers

### Adding a New Endpoint that Affects Balance

If adding a new endpoint that moves funds (e.g., a new refund variant):
1. **Increment `distributed_principal` atomically** when principal leaves the contract.
2. **Update the invariant check** in any endpoints that read the balance.
3. **Add a test** that exercises partial distributions followed by sweep, verifying the floor holds.

### Storage Keys

- `DataKey::Escrow` — Contains `funded_amount`
- `DataKey::DistributedPrincipal` — Tracks cumulative refunds (or 0 if not yet written)

Both must be read together when computing `liability_floor`.

---

## Related Documents

- [Escrow Lifecycle & State Machine](escrow-lifecycle.md)
- [Escrow Init Parameters](escrow-init-parameters.md)
- [Escrow Error Messages](escrow-error-messages.md)
- [Escrow Data Model](escrow-data-model.md)
- [Escrow External Calls & Token Integrations](ESCROW_TOKEN_INTEGRATION_CHECKLIST.md)

---

## Audit Checklist

When auditing the escrow contract, verify:

- [ ] `update_funding_target` reads `distributed_principal` and computes `liability_floor` correctly.
- [ ] `sweep_terminal_dust` (status 4 path) reads the same `distributed_principal` key.
- [ ] Both endpoints reject with appropriate error codes (`TargetBelowFundedAmount = 74`, `SweepExceedsLiabilityFloor = 42`).
- [ ] `refund()` increments `distributed_principal` atomically with token transfer.
- [ ] No code path mutates `distributed_principal` except `refund()` and initialization (default 0).
- [ ] Test coverage includes: partial refund → attempted sweep → blocked; then remaining refund → sweep allowed.
- [ ] `liability_floor` computation uses `saturating_sub` to avoid underflow in edge cases.
