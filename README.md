# Clobby Security Audit

This repository contains a security audit of [VK-RED/clobby](https://github.com/VK-RED/clobby), an on-chain orderbook built on Solana.

## Vulnerability Summary

**Severity:** CRITICAL  
**Type:** Vault Drain / Theft of Funds  

A critical vulnerability was discovered in the partial order fill logic that allows attackers to drain funds from the market vault. When a maker's order is partially filled, the Fill event incorrectly records the **remaining** order amount instead of the **matched** amount, causing makers to be credited more tokens than they're owed.

## Impact

- Attackers can claim ~10x more tokens than legitimately traded
- Repeated exploitation can drain the entire market vault
- All user funds in affected markets are at risk

## Repository Contents

```
├── AUDIT.md              # Detailed vulnerability writeup
├── clobby/               # Cloned vulnerable repository
├── fix/
│   ├── place_order.rs    # Patched version of vulnerable file
│   └── CHANGES.diff      # Diff showing exact changes
└── tests/
    └── exploit.ts        # Proof of concept demonstrating the vulnerability
```

## Quick Links

- [Full Audit Report](AUDIT.md)
- [Vulnerable Code](clobby/programs/clobby/src/instructions/place_order.rs) (lines 119-128)
- [Fixed Code](fix/place_order.rs)
- [Exploit PoC](tests/exploit.ts)

## Fix Summary

The fix adds a `base_amount_matched` field to the `EditOrders` struct to track the actual matched amount, and uses this value when recording Fill events instead of `base_amount_to_set` (the remaining amount).

```rust
// Before (BUG)
market_events.add_event(EventParams{
    base_amount: order.base_amount_to_set,  // WRONG: remaining amount
    ...
});

// After (FIX)
market_events.add_event(EventParams{
    base_amount: order.base_amount_matched,  // CORRECT: matched amount
    ...
});
```

## Disclosure

This audit was conducted as part of responsible security research. A PR with the fix has been submitted to the original repository.

---

*Audit conducted by byte541 | 2026-02-14*
