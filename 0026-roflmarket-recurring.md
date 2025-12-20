# ADR 0026: Recurring Payments for roflmarket (Per-Admin Deposit)

## Component

Oasis SDK (roflmarket module)

## Changelog

- 2025-12-20: Proposed recurring payments with per-admin deposit for roflmarket.

## Status

Proposed

## Context

Roflmarket currently supports one-off payments where users must commit to a specific term upfront and manually extend before expiration. This differs from the cloud-like experience users expect, where instances run continuously without manual payment intervention.

Providers want subscription-style renewals without user intervention each term, while allowing multiple instances to draw from a single funding pool. The payer model must be predictable and avoid surprise charges to third parties.

## Decision

### Overview

Add recurring (subscription-style) payments with automatic term-by-term renewals.

**Funding model:**

1. **Per-instance funding** (`payment_address`): Existing model, unchanged. Anyone can fund via `InstanceCreate` or `InstanceTopUp`, extending `paid_until` immediately.
2. **Per-admin deposit** (new): Admin's shared pool across instances. Used by `InstanceRenew` when `paid_until` is reached.

**Renewal flow:**

1. User creates instance with `auto_renew: true` (or enables later via `InstanceSetAutoRenew`)
2. User funds their deposit via `DepositTopUp`
3. When `paid_until` is reached, scheduler calls `InstanceRenew`
4. `InstanceRenew` transfers one term's payment from deposit to `payment_address`, extending `paid_until`
5. If deposit is insufficient, `renewal_failed_at` is set and instance stops

**Key properties:**

- Only `Payment::Native` supports recurring (not `EvmContract`)
- Admin can toggle auto-renewal on existing instances
- Admin can withdraw from deposit at any time via `DepositWithdraw`
- Third-party top-ups via `InstanceTopUp` take priority (extend `paid_until` immediately)

### Per-Admin Deposit Model

Each admin has a single deposit address derived from their address. Multiple instances can draw from the same deposit.

```rust
pub fn generate_admin_deposit_address(admin: Address) -> [u8; 20] {
    address::generate_custom_eth_address("roflmarket.admin_deposit", admin.as_ref())
}
```

Note: Deposit balances use native account storage via deterministic addresses. No separate
deposit state mapping is required - `Accounts::get_balance()` on the derived address is sufficient.

### Payment Type

No changes to the `Payment` enum. Recurring behavior is controlled per-instance, not per-offer.

### Instance State Updates

```rust
pub struct Instance {
    // ... existing fields ...

    /// If true, scheduler will auto-renew when paid_until is reached.
    /// Set at instance creation or updated via InstanceSetAutoRenew.
    pub auto_renew: Option<bool>,

    /// Term used for auto-renewal (set when auto_renew is true).
    /// Price is looked up from payment.terms.
    pub renewal_term: Option<Term>,

    /// Timestamp of last failed renewal due to insufficient funds (cleared on success).
    pub renewal_failed_at: Option<u64>,
}
```

When `auto_renew` is `false`: existing behavior (prepaid, manual top-up via InstanceTopUp).

When `auto_renew` is `true`:
- Scheduler calls `InstanceRenew` when `paid_until` is reached
- Uses `renewal_term` price from `payment.terms`
- Pulls funds from admin's deposit to `payment_address`
- Third parties can fund via InstanceTopUp (funds locked in `payment_address`)

### New Transactions

#### DepositTopUp (`roflmarket.DepositTopUp`)

Admin deposits funds to their own deposit address. Only the caller's own deposit address can be funded (derived from caller's address). For third-party sponsoring, use `InstanceTopUp` instead which funds the instance's `payment_address` directly.

```rust
// Method: roflmarket.DepositTopUp
pub struct DepositTopUp {
    pub amount: token::BaseUnits,
}

fn tx_deposit_top_up(ctx, body) -> Result<(), Error> {
    let caller = ctx.tx_caller_address();
    let deposit_address = generate_admin_deposit_address(caller);

    // Transfer from caller to their deposit.
    Accounts::transfer(
        caller,
        Address::from_eth(&deposit_address),
        &body.amount,
    )?;

    CurrentState::with(|state| {
        state.emit_event(Event::DepositToppedUp {
            admin: caller,
            amount: body.amount,
        })
    });

    Ok(())
}
```

#### DepositWithdraw (`roflmarket.DepositWithdraw`)

Admin withdraws funds from their deposit.

```rust
pub struct DepositWithdraw {
    pub amount: token::BaseUnits,
}

fn tx_deposit_withdraw(ctx, body) -> Result<(), Error> {
    let caller = ctx.tx_caller_address();
    let deposit_address = generate_admin_deposit_address(caller);

    // Transfer from deposit to caller.
    Accounts::transfer(
        Address::from_eth(&deposit_address),
        caller,
        &body.amount,
    )?;

    CurrentState::with(|state| {
        state.emit_event(Event::DepositWithdrawn {
            admin: caller,
            amount: body.amount,
        })
    });

    Ok(())
}
```

#### InstanceRenew (`roflmarket.InstanceRenew`)

Scheduler renews the instance by drawing from admin's deposit.

```rust
pub struct InstanceRenew {
    pub provider: Address,
    pub id: InstanceId,
}

fn tx_instance_renew(ctx, body) -> Result<(), Error> {
    // Only scheduler app can call.
    let provider = get_provider(body.provider)?;
    ensure_caller_is_scheduler_app(&provider)?;

    let mut instance = get_instance(body.provider, body.id)?;

    // Only process if auto_renew is enabled.
    if !instance.auto_renew.unwrap_or_default() {
        return Err(Error::InvalidArgument);
    }

    let renewal_term = instance.renewal_term.ok_or(Error::InvalidArgument)?;
    let Payment::Native { denomination, terms, .. } = &instance.payment else {
        return Err(Error::InvalidArgument);  // Only Native supports recurring
    };

    let price_per_term = terms.get(&renewal_term).ok_or(Error::InvalidArgument)?;

    // Only accepted instances can renew; cancelled/not-accepted must not charge deposit.
    if instance.status != InstanceStatus::Accepted {
        return Err(Error::InvalidInstanceState);
    }

    // Only renew if current period has ended.
    if ctx.now() < instance.paid_until {
        return Err(Error::RenewalNotDue);
    }

    // Pull from admin's deposit.
    let deposit_address = Address::from_eth(&generate_admin_deposit_address(instance.admin));
    let deposit_balance = Accounts::get_balance(deposit_address, denomination.clone())?;

    if deposit_balance < *price_per_term {
        instance.renewal_failed_at = Some(ctx.now());
        set_instance(instance.clone());
        return Err(Error::InsufficientDeposit);
    }

    // Transfer from admin's deposit to instance payment_address.
    let payment_address = Address::from_eth(&instance.payment_address);
    Accounts::transfer(
        deposit_address,
        payment_address,
        &token::BaseUnits::new(*price_per_term, denomination.clone()),
    )?;

    // Extend paid_until by adding term to the previous paid_until (not ctx.now()).
    // If renewal is late, the provider bears the gap risk - the unfunded period
    // between old paid_until and ctx.now() is not charged to the deposit.
    instance.paid_until += renewal_term.as_secs();
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

Updated to accept `auto_renew` flag from user. When `auto_renew` is enabled and admin has sufficient deposit, payment is drawn from deposit instead of caller's balance.

```rust
pub struct InstanceCreate {
    // ... existing fields ...
    pub auto_renew: Option<bool>,  // New field: user chooses recurring or one-time
}

// In InstanceCreate handler:
instance.auto_renew = body.auto_renew;
instance.renewal_term = if body.auto_renew.unwrap_or_default() {
    // Validate only Native supports recurring.
    if !matches!(offer.payment, Payment::Native { .. }) {
        return Err(Error::InvalidArgument);
    }
    Some(body.term)
} else {
    None
};

// For auto_renew with Native payment, try to use deposit first.
if body.auto_renew.unwrap_or_default() {
    if let Payment::Native { denomination, terms, .. } = &offer.payment {
        let price_per_term = terms.get(&body.term).ok_or(Error::InvalidArgument)?;
        let total_price = price_per_term * body.term_count;
        let deposit_addr = Address::from_eth(&generate_admin_deposit_address(caller));
        let deposit_balance = Accounts::get_balance(deposit_addr, denomination.clone())?;

        if deposit_balance >= total_price {
            // Pay from deposit.
            let payment_addr = Address::from_eth(&instance.payment_address);
            Accounts::transfer(
                deposit_addr,
                payment_addr,
                &token::BaseUnits::new(total_price, denomination.clone()),
            )?;
            instance.paid_until = ctx.now() + body.term.as_secs() * body.term_count;
            // Skip payment.pay() - deposit covered the payment.
            // Continue with instance creation...
        }
    }
}

// Fallback: use existing payment logic if deposit wasn't used.
offer.payment.pay(ctx, &mut instance, body.term, body.term_count)?;
```

#### InstanceSetAutoRenew (`roflmarket.InstanceSetAutoRenew`)

Allows admin to enable or disable auto-renewal on an existing instance.

```rust
pub struct InstanceSetAutoRenew {
    pub provider: Address,
    pub id: InstanceId,
    pub auto_renew: bool,
    pub renewal_term: Option<Term>,  // Required when enabling, ignored when disabling
}

fn tx_instance_set_auto_renew<C: Context>(ctx: &C, body: InstanceSetAutoRenew) -> Result<(), Error> {
    let mut instance = get_instance(body.provider, body.id)?;
    Self::ensure_caller_is_instance_admin(&instance)?;

    if body.auto_renew {
        // Enabling: validate renewal_term exists in payment.terms.
        let term = body.renewal_term.ok_or(Error::InvalidArgument)?;
        let Payment::Native { terms, .. } = &instance.payment else {
            return Err(Error::InvalidArgument);
        };
        if !terms.contains_key(&term) {
            return Err(Error::InvalidArgument);
        }
        instance.renewal_term = Some(term);
    } else {
        instance.renewal_term = None;
    }

    instance.auto_renew = Some(body.auto_renew);
    instance.updated_at = ctx.now();
    set_instance(instance);

    Ok(())
}
```

#### InstanceTopUp (`roflmarket.InstanceTopUp`)

No changes needed. Existing behavior applies to both recurring and non-recurring instances:
- Anyone can call to top up
- Caller pays → funds go to `payment_address`
- `paid_until` extends immediately

This allows third parties to sponsor instances by prepaying additional terms. For recurring instances, this means they won't need deposit-funded renewals until the prepaid time runs out.

#### InstanceClaimPayment (`roflmarket.InstanceClaimPayment`)

No changes needed. Provider claims from instance's `payment_address` as usual. The existing proration logic applies regardless of whether `auto_renew` is set. Renewal funds from admin deposit flow through `payment_address`, so claiming works the same way.

#### InstanceRemove (`roflmarket.InstanceRemove`)

No changes needed. Existing refund behavior applies - refunds go to `refund_data` address (the last payer via InstanceCreate or InstanceTopUp). Admin deposit is not involved in refunds; it's only used as a funding source for renewals.

## Scheduler Changes

The rofl-scheduler app needs updates to call `InstanceRenew` for auto-renewing instances.

### Current Behavior

When `paid_until < now`:
1. Stop the instance
2. Schedule removal (after delay)

### New Behavior

When an instance's `paid_until` is reached:
1. Check if renewal should be attempted: `auto_renew == true` AND `renewal_failed_at` is not set
2. If yes: pass `try_renewal: true` to stop_instance, which calls `InstanceRenew` first
   - On success: return early (don't stop), next cycle will see extended `paid_until`
   - On failure: `renewal_failed_at` is set on-chain, proceed with stop
3. If no: existing behavior (stop and schedule removal)

The `renewal_failed_at` check prevents infinite retry loops - once renewal fails, it won't retry until the field is cleared (by a successful renewal or manual intervention).

### Implementation

**1. In reconcile loop (manager.rs):**

```rust
// When paid_until < now:
let try_renew_first = instance.auto_renew.unwrap_or_default()
    && instance.renewal_failed_at.is_none();

if local_state.running.contains_key(&instance.id) {
    local_state.pending_stop.push((
        instance.id,
        StopRequest { wipe_storage: false, try_renew_first },
    ));
}

// Only schedule removal if not trying renewal.
if !try_renew_first {
    local_state.maybe_remove.push((instance.id, instance.paid_until));
}
```

**2. In stop_instance (manager.rs):**

```rust
async fn stop_instance(
    self: Arc<Self>,
    instance_id: InstanceId,
    wipe_storage: bool,
    try_renewal: bool,
) -> Result<()> {
    if try_renewal {
        match self.client.renew_instance(instance_id).await {
            Ok(_) => {
                slog::info!(self.logger, "instance renewed successfully"; "id" => ?instance_id);
                return Ok(()); // Don't stop - renewal succeeded.
            }
            Err(e) => {
                slog::warn!(self.logger, "instance renewal failed, proceeding with stop";
                    "id" => ?instance_id,
                    "err" => ?e,
                );
                // Fall through to stop logic
            }
        }
    }
    // Existing stop logic...
}
```

**3. Add to MarketClient (client.rs):**

```rust
pub async fn renew_instance(&self, id: InstanceId) -> Result<()> {
    let tx = self.env.app().new_transaction(
        "roflmarket.InstanceRenew",
        market::types::InstanceRenew {
            provider: self.provider,
            id,
        },
    );
    self.env
        .client()
        .sign_and_submit_tx(self.env.signer(), tx)
        .await?
        .ok()?;
    Ok(())
}
```

## Queries

### DepositBalance

Query a user's deposit balance.

```rust
pub struct DepositBalanceQuery {
    pub admin: Address,
    pub denomination: token::Denomination,
}

pub struct DepositBalanceResponse {
    pub balance: u128,
}

fn query_deposit_balance(ctx, query) -> Result<DepositBalanceResponse, Error> {
    let deposit_address = Address::from_eth(&generate_admin_deposit_address(query.admin));
    let balance = Accounts::get_balance(deposit_address, query.denomination)?;
    Ok(DepositBalanceResponse { balance })
}
```

Note: To query which instances draw from an admin's deposit, use the Nexus indexer. An on-chain `InstancesByAdmin` query could be added later if needed.

## Usage Scenarios

### Instance without auto-renewal (existing behavior)

Instances with `auto_renew: false` (or unset) behave exactly as before:
- Initial term is paid via `InstanceCreate`
- Additional terms can be prepaid via `InstanceTopUp`
- When `paid_until` is reached, instance stops
- No deposit is involved

This change is backwards compatible - existing instances and workflows are unaffected.

### Enabling auto-renewal on a new instance

1. Optionally fund your deposit via `DepositTopUp`
2. Call `InstanceCreate` with `auto_renew: true` and desired `term`
3. If deposit has sufficient funds, initial payment is drawn from deposit; otherwise caller pays
4. Before `paid_until` is reached, ensure deposit is funded via `DepositTopUp`
5. When `paid_until` is reached, scheduler automatically renews from deposit

### Enabling auto-renewal on an existing instance

1. Call `InstanceSetAutoRenew` with `auto_renew: true` and `renewal_term`
2. Before `paid_until` is reached, fund your deposit via `DepositTopUp`
3. When `paid_until` is reached, scheduler automatically renews from deposit

### Disabling auto-renewal

To stop future renewals:
- Call `InstanceSetAutoRenew` with `auto_renew: false`, or
- Withdraw/don't add funds to deposit (renewal will fail when insufficient)

The current term continues until `paid_until` (already committed). When auto_renew is disabled or deposit is insufficient, renewal fails and the instance stops. The provider can then remove the instance via `InstanceRemove`.

### Multiple instances sharing a deposit

If an admin has multiple auto-renewing instances:
- All instances draw from the same deposit
- Renewals are processed first-come-first-served (based on scheduler call order)
- If deposit is insufficient for all renewals, some instances will have `renewal_failed_at` set and stop
- Admin should keep deposit funded; `renewal_failed_at` helps UIs show suspended/expired state

## Possible Improvements

Ideas still under consideration that could be added to this ADR:

- Separate billing/admin roles (allow a finance account to fund while ops administer). Would require signed ownership proof from the billing account to authorize the admin to draw from their deposit.
- Recurring payments for `Payment::EvmContract`. Not included since EVM contracts are rarely used in practice. Would require changes: the roflmarket module would need to call a renewal method on the contract (instead of direct transfer), and the contract itself would need to implement deposit/renewal logic. Not possible with current EVM contract interface.

## Consequences

### Positive

- Enables cloud-like usage: users don't need to commit to a specific term upfront and manually extend - auto-renewal handles continuity automatically.
- User flexibility: same offer can be used for one-time or recurring.
- Admin can toggle auto-renewal on existing instances.
- Third parties can sponsor instances without admin being able to withdraw those funds.
- Shared deposit per admin enables multiple instances to renew from one pool.
- Admin with pre-funded deposit doesn't need separate balance for instance creation.
- No changes to Payment enum - recurring controlled per-instance.
- Reuses existing payment infrastructure (payment_address, claim logic, proration).

### Negative

- Added complexity in roflmarket module (deposit lifecycle, scheduler renewal path).
- Added complexity for users: one more payment decision (recurring vs one-time) and deposit management. CLI and rofl.app should monitor admin deposit balances and alert when funds are low.
- No grace period for failed renewals (renewal fails immediately on insufficient funds).

### Neutral

- Multiple instances compete for deposit at renewal time (user responsibility to keep funded).

## Alternatives Considered

- **Separate `NativeRecurring` payment type**: Add a new Payment variant with single `term` and `price_per_term` fields. Rejected to keep Payment enum unchanged and give users flexibility to choose recurring per-instance.
- **`auto_renew` on Payment/Offer instead of Instance**: Provider would decide if offer is recurring. Rejected to give users flexibility - same offer can be one-time or recurring depending on user's choice at instance creation.
- **Deposit-only funding** (no per-instance top-ups): rejected because it would allow admin to withdraw third-party contributions.

## References

- None.
