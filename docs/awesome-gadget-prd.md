# Product Specification: "Awesome Gadget" Product Page

## 1. Objective
This document outlines the functional and non-functional requirements for the **"Awesome Gadget"** product detail page.  
The purpose of this page is to provide users with product information, allow them to configure their order,  
see an accurate real-time price, and proceed to checkout.

---

## 2. Functional Requirements

### 2.1. Pricing & Total Calculation
The **"Total Price"** displayed on the page must be calculated in real-time and should be the single source of truth for the user.

**Formula:**  
`TotalPrice = (BasePrice * Quantity) + ShippingCost + GiftWrapCost - DiscountValue`  

**Base Price:** The base price for one "Awesome Gadget" is **$99.99**.  

**Updates:** The "Total Price" must update instantly whenever the user changes:
- Quantity  
- Gift Wrap (checked or unchecked)  
- Shipping Option  
- A discount is successfully applied  

---

### 2.2. Quantity
- The user must be able to select a quantity for the order.  
- The default quantity is **1**.  
- The minimum allowable quantity for an order is **1**.  
- Users must not be able to order with a quantity of **0 or less**.  

---

### 2.3. Discount Code
- Users can apply one discount code per order.  

**Validation:**  
Codes must be case-insensitive and ignore leading/trailing whitespace (e.g., `" save10 "` should work).  

**Valid Codes:**  
- `SAVE10`: Applies a 10% discount to the subtotal (`BasePrice * Quantity`)  
- `SAVE20`: Applies a 20% discount to the subtotal (`BasePrice * Quantity`)  

**Feedback:**  
- Applying a valid code updates the "Total Price" and displays a clear success message.  
- Applying an invalid or empty code must display a clear error message.  

---

### 2.4. Gift Wrap
- This is an optional checkbox, unchecked by default.  
- Checked: Adds a flat **$5.00** fee to the "Total Price".  

---

### 2.5. Shipping Options
- Users must select one shipping option.  
- **Default:** "Standard" must be selected by default.  

**Options & Costs:**  
- Standard (5-7 Days): **$0.00**  
- Priority (2-3 Days): **$7.50**  
- Express (1 Day): **$20.00**  

---

### 2.6. Order Button & Terms
- The **"Order Now"** button is disabled by default.  
- The button must become enabled only when the **"I agree to the Terms and Conditions"** checkbox is checked.  
- The helper text ("You must agree...") must be visible when the box is unchecked and hidden when the box is checked.  
- The system must not allow an order to be placed if an error is present (e.g., "Invalid discount code" is visible)  
  or if the quantity is less than 1.  

---

## 3. Non-Functional Requirements

### 3.1. Accessibility (A11y)
- All images must have descriptive alt text for screen readers.  
- All interactive elements (inputs, buttons, radio buttons) must be fully navigable and operable using only a keyboard  
  (i.e., via **Tab** and **Enter/Space** keys).  
- All text, especially error messages, must have sufficient color contrast to be easily readable (**WCAG AA** standard).  
- All form elements must have correctly associated labels.  

---

### 3.2. Usability & Visual
- All labels and options (including all shipping options) must be clearly visible to the user.  
- The user must receive clear, immediate feedback for any action (e.g., applying a code, checking a box).  
- Labels must be descriptive and unambiguous, even on small screens.  

---

### 3.3. Responsive Design
- The layout must adapt cleanly to all screen sizes (**mobile**, **tablet**, and **desktop**).  
- No elements should overlap, become hidden, or break the layout at any viewport size.  

---

### 3.4. Performance
- All price calculations must be instantaneous (feel like **< 100ms**).  
