# Task 2 – Discount Code Test Case Checklist 

This document contains positive and negative test cases for the **Discount Code** feature, based on the product specification for the "Awesome Gadget" product page.

All test cases are only for the **Discount Code** feature, including requirements 2.1, 2.3, 2.6, and 3.4._

---

## **Legend: Valid Behaviour (Discount Code Logic)**
This legend describes the expected valid behaviour used as a reference in the test cases below.
| Parameter            | Description                                                    | Expected Behaviour                                             |
|----------------------|----------------------------------------------------------------|----------------------------------------------------------------|
| **Valid Codes**      | `SAVE10`, `SAVE20`                                             | `SAVE10` → 10% off subtotal; `SAVE20` → 20% off subtotal       |
| **Subtotal**         | `BasePrice * Quantity`                                         | Calculated before shipping, gift wrap, or discount             |
| **DiscountValue**    | `% of Subtotal`                                                | e.g. 10% = 0.1 × Subtotal                                      |
| **Gift Wrap**        | Fixed +$5.00                                                   | Not affected by discount                                       |
| **Shipping Options** | Standard = $0.00, Priority = $7.50, Express = $15.00           | Not affected by discount                                       |
| **Total Formula**    | `(BasePrice * Quantity) + Shipping + GiftWrap − DiscountValue` | Used in all price recalculations                               |
| **UI Update Rule**   | Recalculation must be instant (<100ms)                         | Applies on any change (Quantity, Gift Wrap, Shipping, Discount)|
| **Error States**     | “Invalid discount code”, “Please enter a code”                 | Block order until resolved                                     |
| **One Active Code**  | Only one discount can apply at a time                          | New valid code replaces previous one                           |

---

## 0. Test Case Checklist

| **ID**     | **Type** | **Scenario / What to Verify**                   | **Expected Result (Short)**                                 |
|------------|----------|-------------------------------------------------|-------------------------------------------------------------|
| **DC-P01** | Positive | Apply `SAVE10` with default settings            | 10% discount applied, total recalculated correctly          |
| **DC-P02** | Positive | Apply `SAVE20` with default settings            | 20% discount applied, success message shown                 |
| **DC-P03** | Positive | Case-insensitive & trimmed code (` save10 `)    | Code accepted, 10% discount applied                         |
| **DC-P04** | Positive | `SAVE10` with quantity > 1                      | Discount = 10% of (BasePrice × Quantity)                    |
| **DC-P05** | Positive | `SAVE20` with quantity > 1                      | Discount = 20% of subtotal, total lower                     |
| **DC-P06** | Positive | Discount + Gift Wrap + Priority shipping        | Discount applies to subtotal only; shipping/wrap unaffected |
| **DC-P07** | Positive | Change quantity after discount applied          | Discount recalculated instantly; no error                   |
| **DC-P08** | Positive | Change shipping after discount applied          | Total increases by shipping only; discount unchanged        |
| **DC-P09** | Positive | Replace one valid discount code with another    | New discount overrides previous; total recalculated         |
| **DC-P10** | Positive | Clear discount code to remove discount          | Discount removed; total returns to full price               |
| **DC-N01** | Negative | Empty discount code                             | Error “Please enter a code”; no discount                    |
| **DC-N02** | Negative | Code with only spaces                           | Error message shown; no discount                            |
| **DC-N03** | Negative | Completely invalid code                         | “Invalid discount code”; total unchanged                    |
| **DC-N04** | Negative | Partially similar code (`SAVE1`, `SAVE30`)      | Not accepted; error shown                                   |
| **DC-N05** | Negative | Internal spaces in code (`SA VE10`)             | Invalid; error shown                                        |
| **DC-N06** | Negative | Invalid code followed by valid code             | First shows error; valid replaces it successfully           |
| **DC-N07** | Negative | Discount with invalid quantity (0 or negative)  | No negative price; enforce Quantity ≥ 1                     |
| **DC-OB01** | Negative / Order | Discount error visible, try to order   | Order blocked until error cleared                           |
| **DC-OB02** | Negative / Order | Fix invalid code, then order           | Error cleared; success message shown                        |  
| **DC-OB03** | Negative / Order | No discount entered                    | Order proceeds normally; no blocking                        |
| **DC-PF01** | Performance | Apply valid discount (`SAVE10`)             | Update & success message < 100 ms                           |
| **DC-PF02** | Performance | Apply invalid code                          | Error shown immediately; no lag                             |

---

## 1. Positive test cases – core discount behaviour

### DC-P01 – Apply SAVE10 with default settings
- Preconditions:
  - Quantity = 1
  - Shipping = Standard ($0.00)
  - Gift Wrap = unchecked
  - No discount applied
- Steps:
  1. Enter `SAVE10` into Discount Code field.
  2. Click `Apply`.
- Expected:
  - Code is accepted.
  - DiscountValue = 10% of (BasePrice * 1).
  - Total Price recalculated using formula:
    `TotalPrice = (BasePrice * Quantity) + ShippingCost + GiftWrapCost - DiscountValue`.
  - Clear success message is shown.
  - Total updates immediately.

### DC-P02 – Apply SAVE20 with default settings
- Preconditions: 
  - same as DC-P01
- Steps:
  1. Enter `SAVE20`.
  2. Click `Apply`.
- Expected:
  - DiscountValue = 20% of (BasePrice * 1).
  - Total recalculated correctly.
  - Success message displayed.

### DC-P03 – Case-insensitive and trimmed code 
- Preconditions: 
  - Quantity = 1
- Steps:
  1. Enter `  save10  ` (lowercase, with leading/trailing spaces).
  2. Click `Apply`.
- Expected:
  - Input is trimmed and treated case-insensitively.
  - Code accepted as SAVE10.
  - 10% discount applied.

### DC-P04 – SAVE10 with quantity > 1
- Preconditions:
  - Quantity = 3
  - Standard shipping, Gift Wrap unchecked
- Steps:
  1. Enter `SAVE10`.
  2. Click `Apply`.
- Expected:
  - DiscountValue = 10% of (BasePrice * 3).
  - Total = Subtotal + Shipping + GiftWrap − DiscountValue.

### DC-P05 – SAVE20 with quantity > 1
- Preconditions:
  - Quantity = 5
  - Standard shipping, Gift Wrap unchecked
- Steps:
  1. Enter `SAVE20`.
  2. Click `Apply`.
- Expected:
  - DiscountValue = 20% of (BasePrice * 5).
  - Total lower than without discount.

## 2. Positive – interaction with price, gift wrap, shipping

### DC-P06 – Discount with Gift Wrap and Priority shipping
- Preconditions:
  - Quantity = 2
  - Gift Wrap = checked ($5.00)
  - Shipping = Priority ($7.50)
- Steps:
  1. Enter `SAVE10`.
  2. Click `Apply`.
- Expected:
  - Discount applies only to Subtotal (BasePrice * Quantity).
  - GiftWrap and Shipping are not discounted.
  - Total = Subtotal + 7.50 + 5.00 − DiscountValue.

### DC-P07 – Change quantity after discount applied
- Preconditions:
  - Quantity = 1
  - `SAVE10` already applied successfully
- Steps:
  1. Change Quantity from 1 → 4.
- Expected:
  - DiscountValue recalculated as 10% of new Subtotal.
  - Total updates instantly.
  - Discount remains valid, no error shown.

### DC-P08 – Change shipping after discount applied
- Preconditions:
  - Quantity = 2
  - `SAVE20` applied successfully
  - Shipping = Standard
- Steps:
  1. Change shipping from Standard → Express.
- Expected:
  - DiscountValue remains 20% of Subtotal.
  - Total increases exactly by Express shipping cost.
  - Calculation happens instantly.

### DC-P09 – Replace one valid discount code with another
- Preconditions:
  - `SAVE10` already applied successfully
- Steps:
  1. Enter `SAVE20`.
  2. Click `Apply`.
- Expected:
  - Only one discount is active.
  - Previous 10% discount is replaced by 20%.
  - Total recalculated using 20% discount.

### DC-P10 – Clear discount code to remove discount
- Preconditions:
  - Any valid discount is applied (`SAVE10` or `SAVE20`)
- Steps:
  1. Clear the Discount Code field (leave empty).
  2. Click `Apply`.
- Expected:
  - Discount is removed from calculation.
  - Total returns to full price (still including current Shipping and Gift Wrap).
  - If treated as invalid/empty, a proper error message is shown.

---

## 3. Negative test cases – validation & input

### DC-N01 – Empty discount code
- Steps:
  1. Leave Discount Code field empty.
  2. Click `Apply`.
- Expected:
  - Error message like “Please enter a code”.
  - No discount applied.
  - Total unchanged.

### DC-N02 – Code with only spaces
- Steps:
  1. Enter only spaces.
  2. Click `Apply`.
- Expected:
  - Treated as empty input.
  - Error message shown.
  - No discount applied.

### DC-N03 – Completely invalid code
- Steps:
  1. Enter `ABC123`.
  2. Click `Apply`.
- Expected:
  - Error “Invalid discount code”.
  - Total Price does not change.

### DC-N04 – Partially similar code (`SAVE1`, `SAVE30`)
- Steps:
  1. Enter `SAVE1` (or another unsupported variant).
  2. Click `Apply`.
- Expected:
  - Code not accepted.
  - Error message displayed.
  - No discount applied.

### DC-N05 – Internal spaces in code (`SA VE10`)
- Steps:
  1. Enter `SA VE10`.
  2. Click `Apply`.
- Expected:
  - Not recognized as a valid code.
  - Error message shown.
  - Total unchanged.

### DC-N06 – Invalid code followed by valid code
- Steps:
  1. Enter `BADCODE`, click `Apply`.
  2. Then enter `SAVE10`, click `Apply`.
- Expected:
  - Step 1: Error message, no discount applied.
  - Step 2: Error cleared, success message shown, 10% discount applied.

### DC-N07 – Discount with invalid quantity (0 or negative)
- Preconditions:
  - Quantity set to 0 or < 1 (if UI allows)
- Steps:
  1. Enter `SAVE10`.
  2. Click `Apply`.
- Expected:
  - No negative / nonsensical Total Price.
  - System enforces minimum Quantity ≥ 1 for a valid discounted order.

---

## 4. Negative – Order button & error state

### DC-OB-01 – Discount error blocks placing an order
- Preconditions:
  - Enter `BADCODE`, click `Apply`.
  - Error message “Invalid discount code” visible.
  - Terms checkbox is checked (Order button appears enabled).
- Steps:
  1. Click `Order Now`.
- Expected:
  - Order is not placed.
  - No “Order placed successfully” message.
  - User must fix or clear discount error first.

### DC-OB-02 – After fixing discount error, order is allowed
- Preconditions:
  - From DC-OB-01: discount error visible
- Steps:
  1. Replace invalid code with `SAVE20`.
  2. Click `Apply`.
  3. Ensure Terms checkbox is checked.
  4. Click `Order Now`.
- Expected:
  - Error message cleared.
  - Discount applied successfully.
  - Order completes and success message displayed.

### DC-OB-03 - No discount interaction → no discount-related blocking
- Preconditions:
  - User never typed anything in Discount Code field.
- Steps:
  1. Set Quantity ≥ 1.
  2. Check Terms checkbox.
  3. Click `Order Now`.
- Expected:
  - No discount errors.
  - Order is allowed (Discount Code feature does not block normal flow if unused).

---

## 5. Performance test cases (3.4 Performance)

### DC-PF-01 – Performance when applying a valid discount
- Steps:
  1. Enter `SAVE10`.
  2. Click `Apply`.
- Expected:
  - Success message and Total Price update appear almost instantly (<100ms).
  - No visible lag or UI freeze.

### DC-PF-02 – Performance when applying an invalid discount
- Steps:
  1. Enter `BADCODE`.
  2. Click `Apply`.
- Expected:
  - Error message appears immediately.
  - Total remains unchanged.
  - No noticeable delay.
