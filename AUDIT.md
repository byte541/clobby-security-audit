# Security Audit Report: Clobby Orderbook

**Target Repository:** https://github.com/VK-RED/clobby  
**Audit Date:** 2026-02-14  
**Auditor:** byte541  
**Severity:** CRITICAL  

---

## Executive Summary

A critical vulnerability was discovered in the Clobby on-chain orderbook that allows attackers to drain funds from the market vault. The bug exists in the partial order fill logic where the wrong amount is recorded in fill events, leading to inflated token credits for makers.

---

## Vulnerability Details

### CVE-PENDING: Incorrect Base Amount in Partial Fill Events

**File:** `programs/clobby/src/instructions/place_order.rs`  
**Lines:** 119-128  
**Severity:** CRITICAL  
**CVSS Score:** 9.8 (Critical)  

#### Description

When a maker's order is partially filled by a taker, the code creates a `Fill` event to record the match. However, the `base_amount` field in this event is incorrectly set to `order.base_amount_to_set` (the **remaining** unfilled amount) instead of the actual amount that was matched.

#### Vulnerable Code

```rust
// place_order.rs, lines 119-128
for (index, order) in orders_to_edit.iter().enumerate() {
    let order_idx = full_orders_matched + index;
    let partial_matched_order = &mut opposing_side.orders[order_idx];

    if partial_matched_order.order_id == order.order_id {
        partial_matched_order.base_amount = order.base_amount_to_set;

        market_events.add_event(EventParams{
            base_amount: order.base_amount_to_set,  // BUG: This is REMAINING, not MATCHED!
            order_id: partial_matched_order.order_id,
            maker: partial_matched_order.order_authority,
            quote_amount: order.total_quote_amount,
            event_type: EventType::Fill,
            side: event_type_ops_side,
        });
    }
}
```

The issue is that `base_amount_to_set` represents how much of the order is **left unfilled**, not how much was **just matched**. This value is calculated as:

```rust
let base_amount_to_set = opposing_order.base_amount - base_amount_eaten;
```

So if a 10,000 token order has 100 tokens matched, `base_amount_to_set = 9,900` (remaining), but the event records `9,900` instead of `100` (matched).

#### Root Cause Analysis

Looking at the `EditOrders` struct:

```rust
struct EditOrders {
    pub order_id: u64,
    pub base_amount_to_set: u64,    // Remaining amount (for updating the order)
    pub total_quote_amount: u64      // Quote amount for the matched portion
}
```

The struct stores `base_amount_to_set` (remaining) for updating the order in the orderbook, but this same value is incorrectly used for the fill event. The struct should also track `base_amount_eaten` for event recording.

#### Impact

When `consume_events` is called, the fill event is processed:

```rust
// consume_events.rs
EventType::Fill => {
    match event.get_side_in_enum()? {
        Side::Bid => {
            maker_balance_account.base_amount += event.base_amount;  // Credits REMAINING instead of MATCHED!
        },
        Side::Ask => {
            maker_balance_account.quote_amount += event.quote_amount;
        }
    }
}
```

For Bid orders (makers wanting to buy base tokens), the maker's balance account is credited with the **remaining** amount instead of the **matched** amount.

---

## Exploit Scenario

### Attack Steps

1. **Attacker (Maker)** places a large Bid order for 10,000 base tokens at price X
2. **Accomplice (Taker)** places a small Ask order for 1 base token at price X
3. The orderbook matches: 1 base token is traded
4. Internally:
   - `base_amount_eaten = 1`
   - `base_amount_to_set = 10,000 - 1 = 9,999`
   - Fill event records: `base_amount: 9,999` (WRONG!)
5. Market operator calls `consume_events`
6. Attacker's balance account is credited with 9,999 base tokens (instead of 1)
7. Attacker calls `settle_user_balance` and withdraws 9,999 base tokens
8. **Result:** Attacker profits 9,998 base tokens per attack

### Damage Amplification

- The attack can be repeated as long as there are funds in the vault
- Each iteration drains: `(original_order_size - 1)` tokens
- Multiple attackers can coordinate to rapidly drain the entire vault
- Legitimate users lose their deposited funds

### Prerequisites

- Attacker needs a funded account to place the initial order
- Attacker needs an accomplice or second account to place opposing order
- Market must have base tokens in the vault (from other users' orders)

---

## Proof of Concept

See `tests/exploit.ts` for a complete working exploit demonstration.

### Simplified Attack Flow

```typescript
// 1. Attacker places large Bid order
await placeOrder({
    side: Side.Bid,
    baseLots: 10000,
    quoteAmount: 1_000_000,  // price per lot
    ioc: false
});

// 2. Accomplice places minimal Ask order  
await placeOrder({
    side: Side.Ask,
    baseLots: 1,
    quoteAmount: 1_000_000,
    ioc: false
});

// 3. Orders match - 1 lot traded but 9999 recorded

// 4. Consume events (credits wrong amount)
await consumeEvents([attackerBalanceAccount]);

// 5. Settle and drain vault
await settleUserBalance();
// Attacker receives 9999 base tokens instead of 1!
```

---

## Recommended Fix

### Option 1: Track Matched Amount Separately (Recommended)

Modify the `EditOrders` struct to include the matched amount:

```rust
struct EditOrders {
    pub order_id: u64,
    pub base_amount_to_set: u64,     // Remaining (for order update)
    pub base_amount_matched: u64,    // NEW: Amount actually matched (for event)
    pub total_quote_amount: u64
}
```

Update the creation of `EditOrders`:

```rust
orders_to_edit.push(EditOrders { 
    order_id: opposing_order.order_id, 
    base_amount_to_set,
    base_amount_matched: base_amount_eaten,  // NEW: Store matched amount
    total_quote_amount
});
```

Update the event creation:

```rust
market_events.add_event(EventParams{
    base_amount: order.base_amount_matched,  // FIX: Use matched amount
    order_id: partial_matched_order.order_id,
    maker: partial_matched_order.order_authority,
    quote_amount: order.total_quote_amount,
    event_type: EventType::Fill,
    side: event_type_ops_side,
});
```

### Option 2: Calculate On-The-Fly

Calculate the matched amount when creating the event:

```rust
let base_amount_matched = partial_matched_order.base_amount - order.base_amount_to_set;

// Wait, this doesn't work because base_amount is already updated!
```

**Note:** Option 2 won't work because `partial_matched_order.base_amount` is already set to `order.base_amount_to_set` on the line before. Option 1 is the correct fix.

---

## Fix Implementation

See `fix/place_order.rs` for the complete patched file.

### Key Changes

```diff
 struct EditOrders {
     pub order_id : u64,
     pub base_amount_to_set: u64,
+    pub base_amount_matched: u64,
     pub total_quote_amount: u64
 }

 // In the matching loop:
 orders_to_edit.push(EditOrders { 
     order_id: opposing_order.order_id, 
     base_amount_to_set,
+    base_amount_matched: base_amount_eaten,
     total_quote_amount
 });

 // In the event creation:
 market_events.add_event(EventParams{
-    base_amount: order.base_amount_to_set,
+    base_amount: order.base_amount_matched,
     order_id: partial_matched_order.order_id,
     ...
 });
```

---

## Verification Steps

1. Run the exploit test before applying fix - should succeed (demonstrates vulnerability)
2. Apply the fix to `place_order.rs`
3. Run the exploit test again - should fail (attacker can't drain excess tokens)
4. Run all existing tests - should still pass (no regression)

---

## Additional Recommendations

1. **Add Invariant Checks:** After each trade, verify that total credits equal total deposits
2. **Add Balance Cap:** Limit how much can be credited in a single event to prevent massive drains
3. **Formal Verification:** Consider formal verification for critical accounting logic
4. **Audit Coverage:** This codebase would benefit from a professional security audit

---

## Disclosure Timeline

| Date | Action |
|------|--------|
| 2026-02-14 | Vulnerability discovered during security research |
| 2026-02-14 | Audit report and fix prepared |
| 2026-02-14 | PR submitted to repository |
| TBD | Maintainer response |

---

## References

- [Clobby Repository](https://github.com/VK-RED/clobby)
- [Anchor Book - Security](https://book.anchor-lang.com/anchor_in_depth/security.html)
- [Solana Security Best Practices](https://github.com/coral-xyz/sealevel-attacks)

---

*Report prepared by byte541 as part of responsible security disclosure.*
