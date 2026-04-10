# Business Logic Vulnerabilities - Complete Solutions

---

## Task 1: Coupon Abuse (Discount Stacking)

**Tab:** Checkout

**Vulnerability:** The coupon code `SAVE50` applies a 50% discount on the *current* total each time, and there is no server-side check preventing repeated application of the same coupon.

**Steps to Exploit:**

1. Navigate to the **Checkout** tab.
2. Enter `SAVE50` in the coupon code field.
3. Click **Apply**.
4. The total drops from `$230.00` to `$115.00` (50% off).
5. Without changing the code, click **Apply** again.
6. The total drops from `$115.00` to `$57.50`.
7. Click **Apply** again: `$57.50` → `$28.75`.
8. Click **Apply** again: `$28.75` → `$14.38`.
9. Click **Apply** once more: `$14.38` → `$7.19`.
10. Once the total goes below **$10.00**, the flag is revealed.

**Root Cause:** The application does not track whether a coupon has already been applied. Each application recalculates 50% off the *current* (already discounted) total, allowing infinite stacking.

**Flag:** `Flag{Scaler_C0up0n_St4ck3d}`

**Real-World Impact:** Attackers could purchase items for near-zero cost by repeatedly applying the same discount coupon, causing direct financial loss to the business.

---

## Task 2: Negative Quantity Manipulation

**Tab:** Cart

**Vulnerability:** The quantity input field accepts negative numbers. The application calculates the total as `unit_price * quantity` without validating that quantity is a positive integer, causing the total to go negative.

**Steps to Exploit:**

1. Navigate to the **Cart** tab.
2. Notice the +/- stepper buttons next to each product's quantity. Clicking the minus button below 0 shows an error: *"Quantity cannot be less than 0."* -- this is the client-side validation.
3. However, the input field itself is still editable. Click directly into the quantity input for **Mechanical Keyboard (RGB)** ($150.00).
4. Select the current value and **manually type** `-5` using your keyboard.
5. The line total becomes `-$750.00` and the cart total goes negative.
6. Once the total goes below **$0.00**, the flag is revealed.

**Alternative Approach:**

- Type `-3` into the Keyboard qty and `-3` into the Mouse Pad qty.
- Any combination that results in a negative grand total works.

**Key Insight:** The +/- buttons have proper validation and will not go below 0, but the raw input field has no such restriction. Typing a negative number directly bypasses the button-level check.

**Root Cause:** The application validates quantity only in the button click handler (client-side UI guard), but not in the actual calculation logic. The input field itself accepts any typed value, and the total is computed as `price * quantity` without checking for positive values. This is a common pattern where UI-level guards give a false sense of security while the underlying logic remains exploitable.

**Flag:** `Flag{Scaler_N3g4t1v3_C4rt}`

**Real-World Impact:** Attackers could manipulate cart quantities to produce a negative balance, potentially receiving a refund or store credit instead of being charged.

---

## Task 3: Coupon Reuse via Race Condition

**Tab:** Promotions

**Vulnerability:** The promo code `SPRING25` is intended for one-time use. The system checks if the promo has been used before applying it, but marks it as "used" asynchronously after a 1-second delay. During this validation window, multiple rapid submissions can apply the same promo multiple times.

**Steps to Exploit:**

1. Navigate to the **Promotions** tab.
2. Enter `SPRING25` in the promo code field.
3. Click **Redeem** rapidly **3 or more times** within 1 second.
   - First click: Starts the validation, applies $25 credit, begins 1-second "mark as used" timer.
   - Subsequent clicks within the 1-second window: The promo is still in "validating" state and has not yet been marked as used, so each click applies another $25 credit.
4. After 1 second, the promo is finally marked as "used" and further attempts show an error.
5. Once the promo has been applied **3 or more times**, the flag is revealed.

**Key Timing:** All clicks must happen within approximately 1 second of the first click. After 1 second, the promo is locked.

**Root Cause:** The application has a classic TOCTOU (Time-of-Check to Time-of-Use) race condition in its promo redemption logic. The "is this promo already used?" check and the "mark promo as used" action are not atomic. There is an async gap (simulating a database write or API call) between validating and marking, during which concurrent requests can slip through.

**Flag:** `Flag{Scaler_Pr0m0_R4c3d}`

**Real-World Impact:** In real payment systems, race conditions in coupon/promo redemption can allow users to apply one-time discounts multiple times by sending concurrent API requests, leading to unauthorized discounts or credits.

---

## Summary Table

| # | Tab | Coupon/Action | Vulnerability Type | Flag |
|---|-----|--------------|-------------------|------|
| 1 | Checkout | `SAVE50` | Missing duplicate coupon check | `Flag{Scaler_C0up0n_St4ck3d}` |
| 2 | Cart | Type negative qty via keyboard | Missing input validation | `Flag{Scaler_N3g4t1v3_C4rt}` |
| 3 | Promotions | `SPRING25` (rapid clicks) | TOCTOU / Race condition | `Flag{Scaler_Pr0m0_R4c3d}` |
