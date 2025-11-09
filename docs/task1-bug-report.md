# Task 1: Find the Bugs (Functional & Non-Functional)

## QA Bug Report 

This document summarizes all identified issues (24 total) based on the product specification and current implementation.

---

## Bug Classification Guidelines

To improve clarity and traceability, all identified defects are grouped by type.  
Each category corresponds to a specific quality aspect of the product:

### **F — Functional Bugs**  
Issues related to business logic or core functionality that affect how the form behaves or how results are calculated.  
> Examples: incorrect price formula, validation errors, missing or broken user interactions.

### **NF — Non-Functional Bugs**  
Defects that do not directly impact functionality but affect system stability, testability, maintainability, or security.  
> Examples: dynamically generated IDs, unstable selectors, code that complicates automation or maintainability.

### **A — Accessibility & UI Bugs**  
Problems related to accessibility, user interface, or HTML semantics that affect user experience — especially for users relying on assistive technologies.  
> Examples: missing `alt` attributes, elements not reachable via keyboard, multiple `<h1>` tags, low contrast.

### **P — Performance & Responsive Bugs**  
Issues impacting performance, responsiveness, or behavior on different devices and screen sizes.  
> Examples: artificial timeouts, slow UI feedback, non-responsive layout, excessive DOM updates.

---

## Functional Bugs

### F1 – Discount logic and total price formula are incorrect
**Severity:** 🔴 Critical  **Priority:** P1  

#### **Summary**
The discount logic applies the wrong formula **and** uses an incorrect interpretation of discount value.  
Instead of applying a **percentage discount** (10% / 20%) to the **subtotal (BasePrice × Quantity)**,  
the system subtracts a **fixed dollar amount ($10 / $20) per item** and calculates the total incorrectly.

#### Requirement
From the specification:
```text
Total = (BasePrice × Quantity) + Shipping + GiftWrap − DiscountValue
DiscountValue = % × (BasePrice × Quantity)
```

**Valid codes:**
- SAVE10 → 10% off subtotal  
- SAVE20 → 20% off subtotal

#### Actual (in code)
```js
// Incorrect implementation
const flawedTotal = (base - dVal) * qVal + gwc + sc;
```

**Where:**
- `base` = price per item  
- `qVal` = quantity  
- `dVal` = 10 or 20 (used as flat dollars, not percentages)

This logic:
1. Treats `dVal` as a dollar value instead of percentage.  
2. Applies the discount before multiplying by quantity.  
3. Results in over-discounting when quantity > 1.

#### Expected
```js
// Correct implementation
subtotal = base * qVal;
discountValue = subtotal * discountPercent; // 0.1 or 0.2
total = subtotal + gwc + sc - discountValue;
```

Discount should be calculated on the subtotal and expressed as a percentage, not as a fixed value per item.

#### Steps to reproduce
1. Open the “Awesome Gadget” product page.  
2. Base Price = $99.99, Quantity = 3.  
3. Select Standard shipping ($0), no gift wrap.  
4. Apply `SAVE20` discount.  
5. Observe the total.

#### Example

| Scenario | Current (Incorrect)            | Correct (Expected)  | Difference                 |
|-----------|-------------------------------|---------------------|----------------------------|
| 99.99 × 3, SAVE10 | (99.99−10)×3 = 269.97 | 299.97−10% = 269.97 | Coincidence only           |
| 99.99 × 3, SAVE20 | (99.99−20)×3 = 239.97 | 299.97−20% = 239.97 | Same visually, wrong logic |
| 59.99 × 3, SAVE10 | (59.99−10)×3 = 149.97 | 179.97−10% = 161.97 | −$12.00 error              |   
| 120 × 2, SAVE20   | (120−20)×2 = 200.00   | 240−20% = 192.00    | −$8.00 error               |

#### Business Impact
- Inconsistent with product specification.  
- Risk of over- or undercharging users.  
- Breaks scalability for future codes (e.g. `SAVE15`).  
- Creates accounting and trust issues.

#### Recommendation
Unify discount handling logic:
- Store discounts as **percentages** (`0.1`, `0.2`).  
- Apply discount **after subtotal calculation**.  
- Update formula accordingly.

--- 

### F2 – Discount not recalculated properly when quantity changes (Sub-task / Related to F1)
**Severity:** 🟠 Major  **Priority:** P2  
This issue is a concrete scenario of F1 (wrong discount formula).
Once F1 is fixed and retested, this ticket is expected to be resolved as well.

#### Requirement
Discount should be a percentage of subtotal (`BasePrice * Quantity`) at all times.  

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page in the browser.  
2. Set **Quantity = 1**.  
3. Enter discount code `SAVE20` and click **Apply**.  
4. Observe the **Total Price** – it correctly reflects a 20% discount.  
5. Now **change the quantity** to `16` (or any higher number).  
6. Observe the **Total Price** again.  

#### Observed behaviour
- The total price increases linearly with quantity, but the discount amount remains constant (static `dVal`).  
- The formula used is `(base - dVal) * qVal`, meaning the system treats `dVal` as a flat discount **per item**, not as a percentage of the total.  

#### Actual
Discount is stored as a static `dVal` and used inside `(base - dVal) * qVal`.  
It acts as a fixed discount per item, not as a percentage of subtotal.  

#### Expected
When quantity changes, the discount should be recalculated dynamically as:  
```
DiscountValue = (BasePrice × Quantity) × DiscountPercent
```
The **Total Price** should always reflect the correct percentage discount, regardless of quantity.  

#### Recommendation
Refactor the discount calculation to derive `DiscountValue` from the updated subtotal every time quantity changes.  
Call `updatePrice()` after quantity modification to reapply discount logic.

---

### F3 – Total not updating when shipping option changes  
**Severity:** 🔴 Critical  **Priority:** P1  

#### Requirement
Total must update immediately when the user changes the shipping option (Standard, Priority, or Express).  

#### Steps to Reproduce
1. Open the form.  
2. Observe total = $99.99 (base price).  
3. Select **Priority (2–3 Days)**.  
4. Total does not change.  
5. Change quantity or gift wrap — total updates only then.

#### Actual
The total price **does not update** when switching between shipping options.  
It only updates after another field changes (e.g., quantity, discount, or gift wrap).  

#### Root cause (code reference)
```js
shippingRadios.forEach(radio => {
    radio.addEventListener('change', (e) => {
        sc = parseFloat(e.target.value);
        // updatePrice();  // <— should be active
    });
});
```

#### Expected
`updatePrice()` must be called immediately after updating the shipping cost (`sc`) so that the total reflects the current selection.  

#### Recommendation
Uncomment `updatePrice();` or trigger total recalculation inside the shipping change handler.

---

### F4 – Gift wrap resets after applying discount  
**Severity:** 🟠 Major  **Priority:** P2  

#### Requirement
GiftWrapCost should be independent from discount; it should depend only on the Gift Wrap checkbox.  

#### Steps to Reproduce
1. Open the product form page.  
2. Check the **“Add Gift Wrap (+$5.00)”** checkbox.  
3. Observe that the total price increases by **$5.00**.  
4. In the **Discount Code** field, enter `SAVE10` (or `SAVE20`).  
5. Click **Apply**.  
6. Wait for the loading (“Applying...”) to finish.  
7. Observe that the **Gift Wrap checkbox becomes unchecked**, and the **$5.00 cost is removed** from the total.  

#### Actual
When applying any discount code, the code sets `gwc = 0` and unchecks the gift wrap checkbox.  

#### Expected
Applying a discount must not change the state or cost of gift wrap.

#### Root Cause (code reference)
In the `applyBtn.addEventListener('click', ...)` block, the discount logic explicitly resets the gift wrap state:  
```js
gwc = 0;
giftWrapCheck.checked = false;
```

#### Recommendation
Remove or comment these two lines from the discount logic to prevent the gift wrap option from being altered when a discount code is applied.

---

### F5 – Quantity field allows zero or negative values (UI + Logic)
**Severity:** 🔴 Critical  **Priority:** P1  

#### Requirement
- The user must be able to select a quantity for the order.  
- The default quantity is **1**.  
- The **minimum allowable quantity** for an order is **1**.  
- Users **must not be able to order** with a quantity of **0 or less**.  

#### Actual
1. **HTML-level issue:**  
   The `<input>` element is defined as  
   ```html
   <input type="number" id="quantity-input" value="1" min="-1">
   ```  
   ➤ This allows the user to manually enter negative numbers.  

2. **Validation logic issue:**  
   The order validation checks:  
   ```js
   if (qVal < -1)
   ```  
   ➤ This means quantities of `0` and `-1` are **not blocked** and are treated as successful orders.  

#### Expected
- The quantity input should have `min="1"` to prevent invalid values in the UI.  
- Validation should block order submission when `quantity < 1`.  
- Both UI and backend (JS logic) must consistently enforce the same business rule.  

#### Impact
- Users can place orders for **zero or negative quantities**, breaking core business logic.  
- Causes **data integrity issues**, incorrect totals, and potential **financial loss**.  

#### Recommendation
- Fix the HTML input to:  
  ```html
  <input type="number" id="quantity-input" value="1" min="1">
  ```  
- Update validation logic to:  
  ```js
  if (qVal < 1) {
      orderMsg.textContent = 'Error: Quantity must be at least 1.';
      return;
  }
  ```  
- Add a unit/UI test to ensure orders cannot be placed with 0 or negative quantities.

---

### F6 – Empty quantity defaults to 0 (Dependent / Cascade bug)
**Severity:** 🟠 Major  **Priority:** P2  

This issue will be resolved once F5 is fixed, as it shares the same validation logic.

#### Requirement
Minimum order quantity = 1; empty or invalid quantity should not be treated as 0.  

#### Actual
If the quantity field is cleared, `qVal` becomes 0 (`if (isNaN(qVal)) qVal = 0;`). Order can then be placed with 0 items.  

#### Expected
Show a validation error or reset to 1 when the field is empty; do not treat empty input as 0.

---

### F7 – Discount code is case-sensitive and not trimmed  
**Severity:** 🟠 Major  **Priority:** P2  

#### Requirement
Codes must be case-insensitive and ignore leading/trailing whitespace.  

#### Actual
Validation compares the raw input directly (`code === 'SAVE10'` / `'SAVE20'`), so `" save10 "`, `"save10"`, `"SAVE10 "` are treated as invalid.  

#### Expected
Normalize input by trimming and converting to uppercase before comparison (e.g. `code.trim().toUpperCase()`).

---

### F8 – Gift wrap increases price when unchecked  
**Severity:** 🔴 Critical  **Priority:** P1  

#### Requirement
Unchecking gift wrap should remove the **$5.00** fee from the total.

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page in the browser.  
2. Ensure:  
   - **Quantity** = `1`  
   - **Shipping** = **Standard (5–7 Days)**  
   - **Discount Code** field is **empty**.  
3. Note the **Total Price** (should be `$99.99`).  
4. Check the **"Add Gift Wrap (+$5.00)"** checkbox.  
   - Observe: Total Price increases to `$104.99`.  
5. Now **uncheck** the **"Add Gift Wrap"** checkbox.  
   - Observe: Total Price increases again to `$109.99` instead of returning to `$99.99`.  
6. Repeat checking/unchecking several times.  
   - The total keeps increasing by `$5.00` every time the checkbox is unchecked.

#### Actual
Logic uses:  
```js
gwc += 5.00
```  
when the checkbox is **unchecked**, causing the total to **increase** instead of **decrease** each time it’s toggled off.

#### Expected
When the gift wrap checkbox is **unchecked**, set:
```js
gwc = 0;
```
and recalculate the total so the **$5.00** fee is properly **removed** from the total price.

#### Impact
- Users are charged extra each time they toggle the gift wrap option off.  
- Creates **critical billing inconsistencies** and **trust issues**.  

#### Recommendation
Update the event handler for the gift wrap checkbox to use **explicit assignment** instead of incremental logic:
```js
giftWrapCheckbox.addEventListener('change', (e) => {
    gwc = e.target.checked ? 5.00 : 0;
    updatePrice();
});
```

---

### F9 – Order allowed while discount error is visible  

**Severity:** 🟠 Major  **Priority:** P2  

#### Requirement
System must not allow an order to be placed if an error is present (e.g., "Invalid discount code").  

#### Actual
Order submission does not check `discount-message`. Orders can be placed while "Invalid discount code" is shown.  

#### Expected
Block order submission while any discount error message is visible.

---

### F10 – Discount success message disappears after quantity change  
**Severity:** 🟡 Minor  **Priority:** P3  

#### Requirement
After a valid code is applied, feedback should remain visible as long as the discount is active.  

#### Actual
When quantity changes, if `dApplied` is true, the code calls `setDiscountMessage('', '')`, clearing the success message even though discount remains applied.  

#### Expected
Keep or update the success message while the discount is active.

---

### F11 – No validation for discount error on submit  
**Severity:** 🔴 Critical  **Priority:** P1  

#### Requirement
The system must not allow an order to be placed if an error is present.  

#### Actual
Order submission does not check whether a discount error (e.g. "Invalid discount code") is currently displayed.  

#### Expected
Block order submission while any discount-related error is shown.

## Non-functional Bugs (not a UI / not a performance)

### NF12 – Dynamic ID/class reassignment harms testability and stability  
**Severity:** 🟡 Minor  **Priority:** P3  

#### Requirement / QA principle
Selectors for UI automation and DOM structure should be stable and predictable. Critical elements should not rely on fragile, runtime-generated IDs.  

#### Actual
On `DOMContentLoaded`, the script generates a random prefix and assigns dynamic IDs and classes to many elements, then tries to reassign critical IDs using selectors like `label[for="quantity-input"]`, `p.h-5`, and `input[value="0.00"]`. Some of these selectors do not match the existing HTML (e.g. the Quantity label has no `for` attribute), which can lead to inconsistent or broken references.  

#### Expected
Use stable, explicit IDs or dedicated `data-` attributes for key elements. Avoid unnecessary dynamic reassignment of IDs/classes that complicates automation and risks breaking references.  

#### Impact
- Breaks automated UI tests (Playwright, Cypress, Selenium).  
- Increases maintenance cost and reduces DOM predictability.  
- Can cause runtime errors if reassignment fails.  

#### Recommendation
- Use static, meaningful IDs or `data-qa` / `data-testid` attributes.  
- Avoid generating random prefixes for production elements.  
- Validate that all selectors (`label[for="..."]`, `querySelector(...)`) match existing DOM nodes before assignment.  


## Accessibility & UI Bugs

### A13 – Terms helper text never hides  
**Severity:** 🟡 Minor  **Priority:** P3  

#### Requirement
Helper text should be visible only when the Terms checkbox is unchecked.  

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page in the browser.  
2. Scroll to the bottom of the form.  
3. Observe the helper text under the “Order Now” button — *“You must agree to the T&C to order.”*  
4. Check the checkbox **“I agree to the Terms and Conditions.”**  
5. Observe that the helper text remains visible.  

#### Actual
CSS forces the helper text to remain visible at all times:  
```css
#terms-helper {
    display: block !important;
}
```  
As a result, JS cannot hide it when the checkbox is checked.  

#### Expected
The helper text should be hidden when the user checks the Terms & Conditions checkbox.  

#### Root Cause
The `!important` flag in the CSS overrides any runtime display updates from JS.  

#### Recommendation
- Remove `!important` from the CSS rule:  
  ```css
  #terms-helper { display: block; }
  ```  
- Allow JS to toggle visibility dynamically, or use a class-based approach:  
  ```js
  termsCheckbox.addEventListener('change', () => {
      termsHelper.classList.toggle('hidden', termsCheckbox.checked);
  });
  ```  
  ```css
  #terms-helper.hidden { display: none; }
  ```  

---

### A14 – Product image missing alt text by default 
**Severity:** 🟠 Major  **Priority:** P2  

#### Requirement
All images must include descriptive `alt` text for accessibility and screen readers.  

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page in the browser.  
2. Inspect the main product image (above the “Awesome Gadget” title).  
3. Observe that the `<img>` tag has **no `alt` attribute** by default.  
4. Disable image loading or simulate an error — note that `alt` is only added dynamically when the `onerror` handler triggers.  

#### Actual
The product image lacks an `alt` attribute during normal load.  
`alt` is only set when an error occurs through `onerror`, e.g.:  
```html
<img src="awesome-gadget.png" onerror="this.alt='Product image failed to load'">
```

#### Expected
Provide a descriptive, static `alt` attribute by default, e.g.:  
```html
<img src="awesome-gadget.png" alt="Awesome Gadget product image">
```  

#### Impact
- Screen readers cannot describe the image to visually impaired users.  
- Violates accessibility and WCAG 2.1 compliance.  
- Negatively affects SEO and product usability.  

#### Recommendation
- Always include a descriptive `alt` text in the base HTML:  
  ```html
  <img id="product-image" src="awesome-gadget.png" alt="Awesome Gadget product image">
  ```  
- Use the `onerror` handler only for fallback messaging, not for assigning `alt`.  

---

### A15 – Apply button not keyboard accessible
**Severity:** 🔴 Critical  **Priority:** P1  

#### Requirement
All interactive elements must be fully navigable and operable using only a keyboard.  

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page in a browser.  
2. Press the **Tab** key to navigate through form fields.  
3. Observe that focus moves through Quantity, Discount Code, etc.  
4. The **“Apply”** button is skipped — it never receives keyboard focus.  
5. Try to press **Enter** or **Space** — nothing happens because the button cannot be focused.  

#### Actual
`tabindex="-1"` on the **Apply** button removes it from the Tab order, so it cannot be reached or activated using the keyboard.  

Code snippet:
```html
<button id="apply-btn" tabindex="-1">Apply</button>
```

#### Expected
- The **Apply** button should be reachable and activatable using keyboard only.  
- Remove `tabindex="-1"` (or use `tabindex="0"`) so the button is included in the Tab order.  
- Users should be able to navigate to it via **Tab** and activate it using **Enter** or **Space**.

#### Impact
- Users who rely on keyboard navigation (e.g., motor disabilities, screen reader users, or power users) cannot use this form properly.  
- Violates **WCAG 2.1.1 – Keyboard Accessibility**.  
- Prevents discount application for keyboard-only users.  

#### Recommendation
Remove or adjust the tabindex attribute:
```html
<button id="apply-btn" tabindex="0">Apply</button>
```
or simply:
```html
<button id="apply-btn">Apply</button>
```
Ensure that all interactive elements remain reachable and operable using the keyboard.

---

### A16 – Quantity label not associated with input  
**Severity:** 🟠 Major  **Priority:** P2  

#### Requirement
All form elements must have correctly associated labels.

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page in the browser.  
2. Locate the **"Quantity"** field.  
3. Click directly on the **"Quantity"** label text.  
4. Observe that the cursor does **not** move into the input field.  
5. Inspect the element in DevTools → the `<label>` tag has **no `for="quantity-input"` attribute**.  

#### Actual
```html
<label id="quantity-label">Quantity</label>
<input type="number" id="quantity-input">
```
- The label is visually present but not programmatically linked to the input.  
- Clicking the label does not focus the input.  
- Screen readers do not announce the label.  
- JS that references `label[for="quantity-input"]` returns `null`.  
- The DOM script later tries to dynamically reassign IDs and `for` attributes, which breaks automation selectors.  

#### Expected
```html
<label for="quantity-input">Quantity</label>
<input type="number" id="quantity-input">
```
- Label is correctly linked to the input field.  
- Clicking “Quantity” focuses the field.  
- Screen readers announce: “Quantity, edit text”.  

#### Impact
- Accessibility issue (**WCAG 1.3.1 – Info and Relationships**).  
- Breaks usability for keyboard and screen reader users.  
- Causes test automation flakiness — since the locator `label[for="quantity-input"]` does not exist until JS runs.  
- When the HTML is fixed statically, the locator becomes **stable and resilient**, no longer dependent on runtime scripts.  

#### Root Cause
In the **DOMContentLoaded** block, the following line tries to assign the `id` dynamically:  
```js
document.querySelector('label[for="quantity-input"]')?.setAttribute('id', 'quantity-label');
```
Because the `<label>` initially has **no `for` attribute**, this query fails and the link between the label and input is never created.  

#### Recommendation
1. Fix HTML to define a static link between the label and the input:  
   ```html
   <label for="quantity-input" id="quantity-label">Quantity</label>
   <input type="number" id="quantity-input">
   ```
2. **Remove** the dynamic line in the JS block:  
   ```js
   document.querySelector('label[for="quantity-input"]')?.setAttribute('id', 'quantity-label');
   ```

After this fix:
- The label–input association works correctly.  
- The element becomes **keyboard- and screen reader-accessible**.  
- The test locator `label[for="quantity-input"]` becomes **static and reliable**, removing flakiness caused by dynamic ID reassignment.  

---

### A17 – Discount Code label not associated with input  
**Severity:** 🟡 Minor  **Priority:** P3  
(similar to F13)

#### Requirement
All form elements must have correctly associated labels.  

#### Actual
The "Discount Code" label has no `for` attribute, and the input is not linked programmatically.  

#### Expected
Add `for="discount-input"` on the label and ensure the input has `id="discount-input"`.

---

### A18 – Error message has low color contrast  
**Severity:** 🟡 Minor  **Priority:** P3  

#### Requirement
All text, especially error messages, must have sufficient color contrast (WCAG AA).  
*(Source: Product Specification 3.1 Accessibility – “All text, especially error messages, must have sufficient color contrast to be easily readable.”)*

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page.  
2. Leave the **Discount Code** field empty.  
3. Click **Apply Discount**.  
4. Observe the **error message** below the input field.  

#### Actual
- Error message appears in **gray color (`text-gray-400`)**.  
- Contrast ratio is too low against the white background.  
- The message looks like a *hint* rather than an error.  

```js
if (type === 'error') {
    discountMsg.className = discountInput.value
        ? 'text-sm mt-1 h-5 text-red-600'
        : 'text-sm mt-1 h-5 text-gray-400';
}
```

#### Expected
- Error messages must be visually distinct and meet WCAG AA contrast ratios.  
- Color should clearly indicate an error state.  

Example:  
```css
.text-error {
  color: #dc2626; /* Tailwind red-600 */
}
```

#### Impact
- Low contrast makes error messages **hard to read** for users with low vision or on bright screens.  
- Violates **WCAG 2.1 Success Criterion 1.4.3 – Contrast (Minimum)**.  
- Reduces UX clarity and can lead to missed validation feedback.  

#### Recommendation
- Replace `text-gray-400` with a high-contrast red (e.g., `text-red-600` or `text-red-700`).  
- Test contrast using WebAIM Contrast Checker.  
- Ensure all error messages follow the same color pattern for consistency and accessibility compliance.  

---

### A19 – Dynamic messages lack aria-live  
**Severity:** 🟡 Minor  **Priority:** P3  

#### Requirement
Feedback must be announced to screen readers.  

#### Actual
`#discount-message` and `#order-message` are updated dynamically but do not have `aria-live` or `role="status"`. Screen readers are unlikely to announce these changes automatically.  

#### Expected
Add `aria-live="polite"` or `role="status"` to dynamic status elements so messages are announced.

---

### A20 – Express shipping label invisible (missing text)  
**Severity:** 🟠 Major  **Priority:** P2  

#### Requirement
All shipping options must be visible and clearly labeled for the user:  
- Standard (5–7 Days) – $0.00  
- Priority (2–3 Days) – $7.50  
- Express (1 Day) – $20.00  

#### Steps to Reproduce
1. Open the form in a browser.  
2. Scroll to “Shipping Option”.  
3. Notice an unlabeled radio button under the Priority option.  

#### Actual
A third radio button is visible **without any label text**.  
In the CSS, the Express label has `opacity: 0`, making it invisible though still clickable.  

#### Expected
The Express option text should be fully visible to the user.

#### Root cause (code reference)
```css
#shipping-express-label {
    opacity: 0;
    cursor: pointer; /* Still clickable, but invisible */
}
```

#### Recommendation
Remove or override the `opacity: 0` style to display the Express label normally.

---

### A21 – "Terms and Conditions" text truncated on mobile  

**Severity:** 🟡 Minor  **Priority:** P3  

#### Requirement
Labels must be descriptive and unambiguous, even on small screens.  

#### Actual
On small screens, only "I agree" is visible; the rest (`"to the Terms and Conditions"`) is hidden with `hidden sm:inline`. Users do not see what they agree to.  

#### Expected
Show a full, unambiguous label (e.g. "I agree to the Terms and Conditions") on all screen sizes.

---

### A22 – Fixed height for discount message may clip text  
**Severity:** 🟡 Minor  **Priority:** P3  

#### Requirement
Text should be fully readable; UI must accommodate different languages and message lengths.  

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page in the browser.  
2. In the **Discount Code** field, enter any invalid value (e.g. `ABC`) and click **Apply**.  
3. Open **DevTools → Elements** and locate the `#discount-message` element.  
4. Manually replace its text with a **longer message**, for example:  
   > "Invalid discount code. Please check the format and try again, or contact support if the problem persists.Invalid discount code. Please check the format and try again, or contact support if the problem persists."  
5. Observe the rendered message in the UI.

#### Observed
- The container `#discount-message` has the Tailwind class `h-5` (fixed height ~20px).  
- The text wraps to **two lines**, but only the **first line is visible** — the rest is visually clipped.

#### Actual
`#discount-message` has a fixed height via Tailwind class `h-5`. Longer messages (e.g. in other languages or more detailed warnings) may be visually clipped.  

#### Expected
Allow the message container height to grow with content (e.g. remove fixed height or use a larger, flexible height).

---

### A23 – Multiple `<h1>` headings on the page  
**Severity:** 🟡 Minor  **Priority:** P3  

#### Requirement / QA principle
For good semantics, accessibility, and SEO, a page should typically have a single main `<h1>` heading, with additional structure represented as `<h2>`, `<h3>`, etc.  

#### Actual
The page contains two `<h1>` elements: "QA Evaluation Portal" and "Buggy Product Form".  

#### Expected
Use one main `<h1>` for the page (e.g. "QA Evaluation Portal") and downgrade the form title to `<h2>` or another appropriate heading level.  

---

## Performance & Responsive Bugs

### P19 – Artificial delay on discount application  
**Severity:** 🟠 Major  **Priority:** P2  

#### Summary
Applying a discount code feels slow and unresponsive.  
Instead of recalculating the total price immediately, the system shows “Applying…” for ~2.5 seconds before updating the total and showing any feedback.  
This violates the performance requirement for “instantaneous” calculations and creates a poor user experience.

#### Requirement
From the specification (3.4 Performance, 2.3 Discount Code):

- “All price calculations must be instantaneous (feel like < 100ms).”  
- Applying a valid discount code must:
  - Update the **Total Price** in real time.
  - Show clear success/error feedback.

#### Steps to Reproduce
1. Open the **"Buggy Product Form"** page in the browser.  
2. Ensure:
   - Quantity = `1`
   - Shipping = **Standard (5–7 Days) – $0.00**
   - Gift Wrap is **unchecked**  
3. In the **“Discount Code”** field, enter a valid code, e.g. `SAVE10`.  
4. Click the **“Apply”** button.  
5. Observe:
   - The button text changes to **“Applying…”**.
   - The **Total Price** and discount message do **not** update immediately.
   - Only after ~2.5 seconds the Total Price and discount message change.

#### Actual
- Discount logic is executed only **after** a hardcoded delay of 2500 ms.
- During this time:
  - No success/error message is shown.
  - Total Price remains unchanged.
  - The UI feels “frozen” or slow.

Code fragment:
```js
applyBtn.addEventListener('click', () => {
    applyBtn.disabled = true;
    applyBtn.textContent = 'Applying...';
    setDiscountMessage('', '');

    setTimeout(() => {
        const code = discountInput.value;

        // discount logic...

        updatePrice();

        applyBtn.disabled = false;
        applyBtn.textContent = 'Apply';
    }, 2500); // artificial delay
});
```

#### Expected
- Discount should be validated and applied **immediately** after clicking “Apply” (within < 100 ms).
- Total Price and feedback message (success/error) should update without any artificial waiting time.
- If any async call is required (e.g. future API validation), the UI should still feel responsive and not be blocked by a fixed 2.5s timeout.

#### Impact
- **User experience:** feels slow and “buggy” even when everything is working correctly.  
- **Perception:** looks like the system is lagging or hanging.  
- **Compliance:** violates the **performance requirement (<100 ms)** for price calculations.

#### Recommendation
- Remove the artificial delay:
  - Execute discount logic and `updatePrice()` **immediately** inside the click handler, without `setTimeout`.
- If simulation of async behaviour is needed for demo:
  - Use a much smaller delay (e.g. 50–100 ms) **or**
  - Display progress only when there is a real async operation (e.g. API call), not a fixed timeout.

---

### P21 – Heading overlaps layout on tablet widths  
**Severity:** 🟠 Major  **Priority:** P2  

#### Requirement
The layout must adapt cleanly; no elements should overlap or break at any viewport size.  

#### Actual
For viewport widths 640–767px, `h1#form-title` has a large `font-size` and negative `margin-bottom`, which can cause the heading to overlap adjacent elements.  

#### Expected
Use responsive font sizes and non-negative margins so headings do not overlap content on any device.

---

## QA Summary

- Total Issues: 24  
- Critical: 6  
- Major: 9  
- Minor: 9  
