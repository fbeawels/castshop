# CADCurrencyModule.java

## Review

## 1. Summary  
**Purpose** – The `CADCurrencyModule` is a very lightweight implementation of a currency formatter that is tailored to the Canadian Dollar (CAD). It extends the existing `USDCurrencyModule` and simply overrides the currency symbol while delegating all numeric parsing/formatting logic to the parent class.  

**Key Components**  
- **`CADCurrencyModule`** – Concrete class that represents the CAD currency.  
- **`USDCurrencyModule`** – The parent class that already contains all the heavy lifting (parsing, formatting, locale‑specific logic).  
- **`Currency` entity** – An instance field that can be injected (via `setCurrency`) if needed, although it is not used by the current implementation.  

**Design Pattern / Library**  
- The code follows a *Template‑Method* style: the subclass provides only the minimal overrides required for its specific context (currency symbol).  
- No external frameworks are involved beyond the standard JDK (`java.math.BigDecimal`) and the domain `Currency` entity.

---

## 2. Detailed Description  
1. **Initialization**  
   - The class is instantiated with a default constructor (implicit).  
   - The `Currency` field can be set via `setCurrency`, but nothing else in the class relies on it, so it is effectively a no‑op for current functionality.

2. **Runtime Behavior**  
   - **`getAmount(String amount)`** – Calls `USDCurrencyModule.getAmount`, which parses the string (e.g., `"1,234.56"`) into a `BigDecimal`.  
   - **`getFormatedAmount(BigDecimal amount)`** – Delegates to the parent’s formatting logic (adds thousand separators, decimal places, etc.).  
   - **`getFormatedAmountWithCurrency(BigDecimal amount)`** – Same as above but prepends the currency symbol.  
   - **`getCurrencySymbol()`** – Overrides the parent to return `"$"` (the CAD symbol).

3. **Assumptions & Constraints**  
   - The CAD uses the same numeric formatting as USD (same decimal and grouping separators).  
   - The `Currency` field is optional; the class does not enforce any locale or currency code checks.  
   - Error handling is delegated to the parent – any `Exception` thrown by parsing/formatting is propagated up.

4. **Architecture**  
   - A simple inheritance chain: `CADCurrencyModule` → `USDCurrencyModule` → (likely a base `CurrencyModule`).  
   - No composition or dependency injection is employed beyond the simple setter.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side Effects |
|--------|-----------|---------|--------|---------|--------------|
| `setCurrency` | `void setCurrency(Currency currency)` | Stores a `Currency` instance for potential future use. | `Currency` object | none | Sets the private `currency` field. |
| `getAmount` | `BigDecimal getAmount(String amount) throws Exception` | Parses a string representation of money into a `BigDecimal`. | String amount (e.g., `"1,234.56"`) | `BigDecimal` | Delegates to `USDCurrencyModule`; may throw parsing exceptions. |
| `getFormatedAmount` | `String getFormatedAmount(BigDecimal amount) throws Exception` | Formats a `BigDecimal` into a locale‑appropriate string (e.g., `"1,234.56"`). | `BigDecimal` | Formatted string | Delegates to parent. |
| `getFormatedAmountWithCurrency` | `String getFormatedAmountWithCurrency(BigDecimal amount) throws Exception` | Formats amount and prefixes the currency symbol. | `BigDecimal` | Formatted string with symbol (e.g., `"$1,234.56"`) | Delegates to parent. |
| `getCurrencySymbol` | `String getCurrencySymbol()` | Provides the CAD currency symbol. | none | `"$"` | No side effects. |

*Utility Methods* – All heavy work is handled by the inherited methods; this class has no reusable utility functions of its own.

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `java.math.BigDecimal` | JDK | Standard library for arbitrary‑precision decimal arithmetic. |
| `com.salesmanager.core.entity.reference.Currency` | Domain | Entity representing a currency; used only as a holder, not processed. |
| `com.salesmanager.core.module.impl.application.currencies.USDCurrencyModule` | Local | Superclass providing parsing/formatting logic. |
| No other third‑party libraries or platform‑specific APIs. |

---

## 5. Additional Notes  

### Strengths  
- **Simplicity** – Minimal code, easy to read and maintain.  
- **Reusability** – By inheriting from `USDCurrencyModule`, it avoids duplication of parsing/formatting logic.  
- **Extensibility** – Future enhancements (e.g., different symbols, rounding rules) can be added in this subclass without touching the base logic.

### Weaknesses / Edge Cases  
1. **Unused `currency` Field** – The `Currency` object is never referenced. If it is meant to carry metadata (code, symbol, locale), its presence may mislead developers or cause confusion.  
2. **Hard‑coded Symbol** – CAD shares the dollar symbol with USD, but there could be ambiguity (e.g., for contexts where a different symbol is preferred).  
3. **Locale Assumptions** – The class assumes CAD formatting is identical to USD. While true in many locales, it might differ (e.g., Canadian English vs. French formatting).  
4. **Exception Propagation** – The methods simply rethrow `Exception`; a more specific exception hierarchy could improve error handling.  
5. **Method Naming** – `getFormatedAmount` is misspelled (should be `getFormattedAmount`). This may affect consistency if the base class uses the correct spelling.

### Suggested Enhancements  
- **Remove or document the `currency` field** if it is truly unused.  
- **Introduce a `Currency` reference in the superclass** to centralize symbol/locale logic, then override only where necessary.  
- **Add unit tests** to verify that the symbol and formatting behave as expected across different locales.  
- **Handle locale differences** by allowing a locale parameter or by reading locale data from the `Currency` entity.  
- **Rename methods for consistency** (e.g., `getFormattedAmount`).  
- **Add logging** for parsing failures to aid diagnostics.

Overall, the module is functional but intentionally thin; its main responsibility is to provide the CAD symbol while reusing the USD logic. The design is clean, but a few minor refactors could make the code more robust and future‑proof.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 24, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.module.impl.application.currencies;

import java.math.BigDecimal;

import com.salesmanager.core.entity.reference.Currency;

public class CADCurrencyModule extends USDCurrencyModule {

	protected final static char DOLLAR = '\u0024';

	private Currency currency;

	public void setCurrency(Currency currency) {
		this.currency = currency;
	}

	public BigDecimal getAmount(String amount) throws Exception {
		// same as US currency
		return super.getAmount(amount);
	}

	public String getFormatedAmount(BigDecimal amount) throws Exception {
		// same as USD
		return super.getFormatedAmount(amount);
	}

	public String getFormatedAmountWithCurrency(BigDecimal amount)
			throws Exception {
		// same as USD
		return super.getFormatedAmountWithCurrency(amount);
	}

	public String getCurrencySymbol() {
		return Character.toString(DOLLAR);
	}

}



```
