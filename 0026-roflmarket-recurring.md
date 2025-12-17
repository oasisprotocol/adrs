# ADR 0026: Recurring Payments for roflmarket (Per-Admin Escrow)

## Component

Oasis SDK (roflmarket module)

## Changelog

- 2025-12-17: Proposed recurring payments with per-admin escrow for roflmarket.

## Status

Proposed

## Context

roflmarket currently supports one-off payments. Providers want subscription-style renewals without user intervention each term, while allowing multiple instances to draw from a single funding pool. The payer model must be predictable and avoid surprise charges to third parties.

## Decision

### Overview

Add recurring (subscription-style) payments with automatic term-by-term renewals. Funds can come from two sources:

1. **Per-instance funding** (`payment_address`): Anyone can fund via InstanceCreate or InstanceTopUp. Used first during renewal.
2. **Per-admin escrow** (fallback): Admin's shared pool across instances. Used when payment_address is insufficient.

### Per-Admin Escrow Model

Each admin has a single escrow address derived from their address. Multiple instances can draw from the same escrow.

```rust
pub fn generate_user_escrow_address(admin: Address) -> [u8; 20] {
    address::generate_custom_eth_address("roflmarket.escrow", admin.as_ref())
}
```

Note: Escrow balances use native account storage via deterministic addresses. No separate
escrow state mapping is required - `Accounts::get_balance()` on the derived address is sufficient.

### New Payment Variant

```rust
pub enum Payment {
    Native { /* existing */ },
    EvmContract { /* existing */ },

    /// Recurring payments with per-instance funding and per-admin escrow fallback.
    ///
    /// Uses a single `term` and `price_per_term` rather than `BTreeMap<Term, u128>` like `Native`.
    /// This ensures deterministic scheduler behavior and predictable renewals. Funds from
    /// InstanceCreate/InstanceTopUp go to payment_address (per-instance). On renewal, payment_address
    /// is checked first; if insufficient, admin's escrow is used as fallback. This allows third
    /// parties to sponsor instances without admin being able to withdraw those funds.
    NativeRecurring {
        denomination: token::Denomination,
        term: Term,
        price_per_term: u128,
    },
}
```

### Instance State Updates

```rust
pub struct Instance {
    // ... existing fields ...
    /// Timestamp of last failed renewal due to insufficient escrow (cleared on success).
    pub renewal_failed_at: Option<u64>, // New field
}
```

### New Transactions

#### EscrowTopUp (`roflmarket.EscrowTopUp`)

User deposits funds to their escrow. Can be used by any instances where they are the admin.

```rust
// Method: roflmarket.EscrowTopUp
pub struct EscrowTopUp {
    pub amount: token::BaseUnits,
}

fn tx_escrow_top_up(ctx, body) -> Result<(), Error> {
    let caller = ctx.tx_caller_address();
    let escrow_address = generate_user_escrow_address(caller);

    // Transfer from caller to their escrow.
    Accounts::transfer(
        caller,
        Address::from_eth(&escrow_address),
        &body.amount,
    )?;

    CurrentState::with(|state| {
        state.emit_event(Event::EscrowToppedUp {
            admin: caller,
            amount: body.amount,
        })
    });

    Ok(())
}
```

#### EscrowWithdraw (`roflmarket.EscrowWithdraw`)

User withdraws funds from their escrow. User is responsible for keeping enough funds for renewals.

```rust
pub struct EscrowWithdraw {
    pub amount: token::BaseUnits,
}

fn tx_escrow_withdraw(ctx, body) -> Result<(), Error> {
    let caller = ctx.tx_caller_address();
    let escrow_address = generate_user_escrow_address(caller);

    // Transfer from escrow to caller.
    Accounts::transfer(
        Address::from_eth(&escrow_address),
        caller,
        &body.amount,
    )?;

    CurrentState::with(|state| {
        state.emit_event(Event::EscrowWithdrawn {
            admin: caller,
            amount: body.amount,
        })
    });

    Ok(())
}
```

#### InstanceProcessRecurring (`roflmarket.InstanceProcessRecurring`)

Scheduler activates the next term by drawing from admin's escrow.

```rust
pub struct InstanceProcessRecurring {
    pub provider: Address,
    pub id: InstanceId,
}

fn tx_instance_process_recurring(ctx, body) -> Result<(), Error> {
    // Only scheduler app can call.
    let provider = get_provider(body.provider)?;
    ensure_caller_is_scheduler_app(&provider)?;

    let mut instance = get_instance(body.provider, body.id)?;

    let Payment::NativeRecurring { denomination, term, price_per_term } = &instance.payment else {
        return Err(Error::InvalidArgument);
    };

    // Only accepted instances can renew; cancelled/not-accepted must not charge escrow.
    if instance.status != InstanceStatus::Accepted {
        return Err(Error::InvalidInstanceState);
    }

    // Only renew if current period has ended.
    if ctx.now() < instance.paid_until {
        return Err(Error::RenewalNotDue);
    }

    let payment_address = Address::from_eth(&instance.payment_address);
    let instance_balance = Accounts::get_balance(payment_address, denomination.clone())?;

    // First, check if payment_address already has funds (from InstanceTopUp).
    // If not, fall back to admin's escrow.
    if instance_balance < *price_per_term {
        let escrow_address = Address::from_eth(&generate_user_escrow_address(instance.admin));
        let escrow_balance = Accounts::get_balance(escrow_address, denomination.clone())?;

        if escrow_balance < *price_per_term {
            // Neither instance nor escrow has sufficient funds.
            instance.renewal_failed_at = Some(ctx.now());
            set_instance(instance.clone());

            CurrentState::with(|state| {
                state.emit_event(Event::EscrowDepleted {
                    provider: body.provider,
                    id: body.id,
                    admin: instance.admin,
                })
            });
            return Err(Error::InsufficientEscrow);
        }

        // Transfer from admin's escrow to instance payment_address.
        Accounts::transfer(
            escrow_address,
            payment_address,
            &token::BaseUnits::new(*price_per_term, denomination.clone()),
        )?;
    }
    // If instance_balance >= price_per_term, funds are already in payment_address.
    // No transfer needed; provider will claim from there.

    // Extend paid_until by adding term to the previous paid_until (not ctx.now()).
    // If renewal is late, the provider bears the gap risk - the unfunded period
    // between old paid_until and ctx.now() is not charged to the escrow.
    instance.paid_until += term.as_secs();
    instance.updated_at = ctx.now();
    instance.renewal_failed_at = None;

    let new_paid_until = instance.paid_until;
    set_instance(instance);

    CurrentState::with(|state| {
        state.emit_event(Event::TermRenewed {
            provider: body.provider,
            id: body.id,
            paid_until: new_paid_until,
        })
    });

    Ok(())
}
```

### Updated Transactions

#### InstanceCreate (`roflmarket.InstanceCreate`)

For `NativeRecurring`, `term_count` specifies how many terms to fund. Funds go directly to the instance's `payment_address`, not to escrow. This allows anyone (not just admin) to create and fund instances. The first term is activated immediately; remaining funds stay in `payment_address` for future renewals.

```rust
// In InstanceCreate handler for NativeRecurring:
let Payment::NativeRecurring { denomination, term, price_per_term } = &offer.payment else {
    // ... handle other payment types
};

if body.term_count == 0 {
    return Err(Error::InvalidArgument);
}

let caller = ctx.tx_caller_address();
let total_amount = price_per_term
    .checked_mul(body.term_count as u128)
    .ok_or(Error::InvalidArgument)?;

// Transfer from caller directly to instance payment_address.
// Funds here are used for renewals before falling back to admin's escrow.
Accounts::transfer(
    caller,
    Address::from_eth(&instance.payment_address),
    &token::BaseUnits::new(total_amount, denomination.clone()),
)?;

// Set initial payment period (1 term activated, rest stays in payment_address for renewals).
instance.paid_from = ctx.now();
instance.paid_until = ctx.now() + term.as_secs();
```

#### InstanceTopUp (`roflmarket.InstanceTopUp`)

For `NativeRecurring`, anyone can top up the instance's `payment_address` directly. These funds are used first during renewal (before falling back to admin's escrow). This allows third parties to sponsor specific instances without the admin being able to withdraw those funds.

```rust
fn pay(&self, ctx, instance, term, term_count) -> Result<(), Error> {
    match self {
        Payment::Native { .. } => { /* existing */ },
        Payment::EvmContract { .. } => { /* existing */ },
        Payment::NativeRecurring { denomination, term: offer_term, price_per_term } => {
            // Validate term matches offer (recurring has single fixed term).
            if term != *offer_term {
                return Err(Error::InvalidArgument);
            }

            let caller = CurrentState::with_env(|env| env.tx_caller_address());
            let amount = price_per_term
                .checked_mul(term_count as u128)
                .ok_or(Error::InvalidArgument)?;

            // Transfer from caller directly to instance payment_address.
            // These funds are used first during renewal, before escrow.
            Accounts::transfer(
                caller,
                Address::from_eth(&instance.payment_address),
                &token::BaseUnits::new(amount, denomination.clone()),
            )?;
        }
    }
    // Note: paid_until is NOT extended here. Renewal extends it when due.
    Ok(())
}
```

#### InstanceClaimPayment (`roflmarket.InstanceClaimPayment`)

No changes needed. Provider claims from instance's `payment_address` as usual. The funds were moved there during `InstanceProcessRecurring`.

```rust
fn claim(&self, ctx, provider, instance) -> Result<(), Error> {
    match self {
        Payment::Native { .. } => { /* existing */ },
        Payment::EvmContract { .. } => { /* existing */ },
        Payment::NativeRecurring { denomination, .. } => {
            // Same proration logic as Native - see payment.rs for details.
            // Claims proportional amount based on (claimable_time / paid_time) * balance.
        }
    }
}
```

#### InstanceRemove (`roflmarket.InstanceRemove`)

For `NativeRecurring`, refund instance `payment_address` balance to admin's escrow (not directly to user).

```rust
fn refund(&self, ctx, instance) -> Result<(), Error> {
    match self {
        Payment::Native { .. } => { /* existing - refund to refund_data address */ },
        Payment::EvmContract { .. } => { /* existing */ },
        Payment::NativeRecurring { denomination, .. } => {
            // Refund to current admin's escrow (derived from instance.admin, not refund_data).
            // Note: If admin was changed after the last renewal, the new admin receives the
            // refund even though the previous admin may have funded that term. This is a
            // known trade-off for simplicity; parties should coordinate timing of admin transfers.
            let payment_address = Address::from_eth(&instance.payment_address);
            let escrow_address = Address::from_eth(&generate_user_escrow_address(instance.admin));

            let amount = Accounts::get_balance(payment_address, denomination)?;

            Accounts::transfer(
                payment_address,
                escrow_address,
                &token::BaseUnits::new(amount, denomination.clone()),
            )?;
        }
    }
}
```

### New Events

```rust
/// User topped up their escrow.
EscrowToppedUp {
    admin: Address,
    amount: token::BaseUnits,
},

/// User withdrew from their escrow.
EscrowWithdrawn {
    admin: Address,
    amount: token::BaseUnits,
},

/// Instance term was renewed via recurring payment.
TermRenewed {
    provider: Address,
    id: InstanceId,
    paid_until: u64,
},

/// Instance renewal failed due to insufficient escrow.
EscrowDepleted {
    provider: Address,
    id: InstanceId,
    admin: Address,
},
```

### New Errors

```rust
/// Renewal is not yet due (paid_until not reached).
RenewalNotDue,

/// Insufficient funds for renewal (neither payment_address nor escrow has enough).
InsufficientEscrow,
```

## Queries

### EscrowBalance

Query a user's escrow balance.

```rust
pub struct EscrowBalanceQuery {
    pub admin: Address,
    pub denomination: token::Denomination,
}

pub struct EscrowBalanceResponse {
    pub balance: u128,
}

fn query_escrow_balance(ctx, query) -> Result<EscrowBalanceResponse, Error> {
    let escrow_address = Address::from_eth(&generate_user_escrow_address(query.admin));
    let balance = Accounts::get_balance(escrow_address, query.denomination)?;
    Ok(EscrowBalanceResponse { balance })
}
```

Note: To query which instances draw from an admin's escrow, use the Nexus indexer. An on-chain `InstancesByAdmin` query could be added later if needed.

## Edge Cases

### Cancelling a recurring instance

To stop future renewals, don't add more funds. The current term continues until `paid_until`
(already committed). When payment_address and escrow are both insufficient, renewal fails
with `EscrowDepleted`. The provider can then remove the instance via `InstanceRemove`.

### Multiple instances, insufficient escrow

If an admin has multiple instances but not enough escrow for all renewals:
- Renewals are processed first-come-first-served (based on scheduler call order)
- Instances that can't renew emit `EscrowDepleted` event
- User should monitor events and keep escrow funded; `renewal_failed_at` is persisted to help UIs show suspended/expired state.

### Instance cancelled with funds in payment_address

- Provider claims for time served
- Remaining balance in payment_address is NOT auto-refunded
- Provider should call `InstanceRemove` to refund remainder to admin's escrow

### Future work

- Separate billing/admin roles (allow a finance account to fund while ops administer) can be added later if needed, with explicit payer consent.

## Consequences

### Positive

- Two-tier funding: per-instance (anyone can fund) + per-admin escrow (shared fallback).
- Third parties can sponsor instances without admin being able to withdraw those funds.
- Shared escrow per admin enables multiple instances to renew from one pool.
- Deterministic escrow address enables easy balance monitoring.
- Reuses existing payment infrastructure (payment_address, claim logic, proration).

### Negative

- No split billing role (finance vs ops) in this iteration.
- Provider bears gap risk if scheduler renewal is late.
- Added complexity in roflmarket module (escrow lifecycle, scheduler renewal path).
- No grace period for failed renewals (immediate `EscrowDepleted` on insufficient funds).

### Neutral

- Multiple instances compete for escrow at renewal time (user responsibility to keep funded).

## Alternatives Considered

- Escrow-only funding (no per-instance top-ups): rejected because it would allow admin to withdraw third-party contributions.
- Split billing/admin roles with third-party payer consent: deferred to future work to avoid added flows in initial rollout.

## References

- None.
