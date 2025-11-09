# Task 3: Automation & Selectors

Imagine you need to automate a test for this form.  
The `id`, `class`, and `data-qa` attributes are dynamic and change on every page load (their prefix, e.g., `el-xyz-123`, is randomized).  

Write **stable**, **understandable**, **resilient**, and **clean** CSS or XPath selectors for the following elements:

---

### 1. The "Quantity" input field  
```xpath
//label[normalize-space(.)='Quantity']
       /following-sibling::input[@type='number']
```

**Rationale:**  
- Uses the visible text `Quantity` as anchor.  
- Selects the adjacent `input` of type `number`.  
- Ignores unstable attributes (`id`, `class`, `data-qa`).  

---

### 2. The "Apply Discount" button  
```xpath
//label[normalize-space(.)='Discount Code']
       /following-sibling::div
       //button[normalize-space(.)='Apply']
```

**Rationale:**  
- Anchored to the label `Discount Code`.  
- Searches within its container for the button text `Apply`.  
- Avoids dynamic IDs and class names.  

---

### 3. The "Total Price" text (the $ amount)  
```xpath
//p[normalize-space(.)='Total Price:']
       /following-sibling::p[contains(normalize-space(.), '$')]
```

**Rationale:**  
- Finds the `p` element with text `Total Price:`.  
- Selects the adjacent `p` element containing the `$` symbol (dynamic value).  
- Works even if price changes.  

---

### 4. The "Priority Shipping" radio button  
```xpath
//fieldset[.//legend[normalize-space(.)='Shipping Option']]
       //label[contains(normalize-space(.), 'Priority (2-3 Days)')]
       /preceding-sibling::input[@type='radio']
```

**Rationale:**  
- Targets the `fieldset` with legend `Shipping Option`.  
- Locates the label containing `Priority (2-3 Days)`.  
- Selects its corresponding `input` radio element.  

---

 **Summary:**  
These XPath locators are:  
- Independent of unstable `id`, `class`, or `data-qa` prefixes.  
- Readable and maintainable.  
- Robust against minor layout or attribute changes.
