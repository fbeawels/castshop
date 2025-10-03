# CurrencyModule.java

## Review

## 1. Summary  
**Purpose**  
`CurrencyModule` is an abstraction for dealing with monetary values in a multi‑currency e‑commerce system.  
It defines the contract for formatting, parsing and validating amounts, and for retrieving the currency symbol. The interface is intended to be implemented by a concrete service that knows the current `Currency` (from `com.salesmanager.core.entity.reference.Currency`) and can perform locale‑aware formatting.

**Key Components**  
| Component | Role |
|-----------|------|
| `getMeasure(BigDecimal, String)` | Formats a raw numeric value into a locale‑specific string (e.g., “1 000,00 USD”). |
| `getAmount(String)` | Parses a string representation of a monetary value and returns a `BigDecimal`. |
| `getFormatedAmount(BigDecimal)` | Formats a value with the default currency (no symbol). |
| `getFormatedAmountWithCurrency(BigDecimal)` | Formats a value and prefixes/suffixes the currency symbol. |
| `getFormatedAmountWithCurrency(BigDecimal, String)` | Same as above but accepts a CSS class name for styling in an HTML context. |
| `getCurrencySymbol()` | Returns the symbol for the current currency (e.g., “$”). |
| `setCurrency(Currency)` | Sets the currency context for the module. |

**Design Patterns & Libraries**  
* The interface follows the **Strategy** pattern – concrete implementations can switch formatting logic (e.g., using `java.text.NumberFormat`, `Locale`, or a custom formatter).  
* No framework annotations are used; it is pure Java.  
* Relies on `BigDecimal` for precision monetary calculations.  
* Uses the domain entity `Currency` from the SalesManager core model.

---

## 2. Detailed Description  
### Core Flow
1. **Instantiation** – A concrete class implements `CurrencyModule`. During construction (or via a setter) it receives a `Currency` instance, which contains the currency code, symbol, decimal precision, etc.  
2. **Formatting** – Methods that return `String` take a `BigDecimal` (the amount) and use the `Currency` information plus locale rules to produce a human‑readable string.  
3. **Parsing** – `getAmount(String)` takes a user‑supplied string (possibly containing currency symbols, grouping separators, etc.) and converts it into a canonical `BigDecimal`.  
4. **Utility** – `getCurrencySymbol()` exposes the symbol so callers can render UI elements without knowing the implementation details.

### Assumptions & Constraints
* The interface expects callers to supply a **valid** `Currency` via `setCurrency`. No method is `@NonNull` annotated; a missing currency may result in `NullPointerException`.  
* All methods throw generic `Exception`. Implementations must provide meaningful error handling (e.g., `ParseException`, `NumberFormatException`).  
* No locale parameter is exposed; the implementation is presumed to handle locale internally based on the `Currency` or a thread‑local context.  
* The contract does not specify thread‑safety. Implementations should be careful if used concurrently (e.g., shared `NumberFormat` instances).  

### Architecture & Design Choices
* **Interface‑First**: The interface separates API from implementation, which is ideal for swapping between different formatting strategies (JDK, ICU4J, custom).  
* **Currency Context**: By providing a `setCurrency` method rather than passing the currency into every formatting call, the design keeps method signatures simple.  
* **String Formatting vs. UI Rendering**: Methods that accept an `amountCssClassName` indicate that the module is aware of HTML presentation, which blurs the line between business logic and view concerns.  

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Parameters | Returns | Throws | Side Effects |
|--------|-----------|---------|------------|---------|--------|--------------|
| `getMeasure` | `String getMeasure(BigDecimal measure, String currencycode) throws Exception` | Formats a raw numeric measure according to a given currency code (likely for unit display). | `measure`: numeric value; `currencycode`: ISO currency code. | Formatted string. | `Exception` (e.g., invalid code). | None. |
| `getAmount` | `BigDecimal getAmount(String amount) throws Exception` | Parses a string amount into a `BigDecimal`, validating against the current currency's rules. | `amount`: user input string. | `BigDecimal`. | `Exception` (parsing errors). | None. |
| `getFormatedAmount` | `String getFormatedAmount(BigDecimal amount) throws Exception` | Returns amount formatted without currency symbol. | `amount`: numeric value. | Formatted string. | `Exception`. | None. |
| `getFormatedAmountWithCurrency` | `String getFormatedAmountWithCurrency(BigDecimal amount) throws Exception` | Same as above but includes the currency symbol. | `amount`: numeric value. | Formatted string. | `Exception`. | None. |
| `getFormatedAmountWithCurrency` (overload) | `String getFormatedAmountWithCurrency(BigDecimal amount, String amountCssClassName) throws Exception` | Adds a CSS class name wrapper for HTML rendering. | `amount`: numeric value; `amountCssClassName`: CSS class. | Formatted string. | `Exception`. | None. |
| `getCurrencySymbol` | `String getCurrencySymbol()` | Returns the symbol for the current currency. | None. | Currency symbol. | None. | None. |
| `setCurrency` | `void setCurrency(Currency currency)` | Sets the currency context used by the module. | `currency`: instance of `com.salesmanager.core.entity.reference.Currency`. | None. | None. |

**Reusable/Utility Methods**  
The interface itself contains no reusable code; the expectation is that concrete implementations will share helper functions (e.g., a private `NumberFormat` builder).  

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `java.math.BigDecimal` | Standard JDK | Provides arbitrary‑precision decimal arithmetic. |
| `com.salesmanager.core.entity.reference.Currency` | Third‑party domain | Represents a currency entity (code, symbol, precision). |
| Java text formatting APIs (`java.text.NumberFormat`, `java.util.Locale`) | Implicit | Likely used in implementations for locale‑aware formatting. |
| No external frameworks or libraries are declared in the interface itself. |  |  |

Platform/Environment: pure Java; no Android, Spring, or other platform assumptions.  

---

## 5. Additional Notes  

### Strengths  
* **Clear Separation of Concerns** – The interface focuses solely on monetary formatting, leaving other concerns (currency management, persistence) to separate components.  
* **Extensibility** – Implementers can choose any formatting strategy (JDK, ICU, custom).  
* **Internationalization Support** – By exposing currency codes and allowing formatting without a locale parameter, the module can adapt to different locales internally.

### Weaknesses / Improvement Areas  
1. **Exception Granularity**  
   * Throwing `Exception` is too broad. Define custom exceptions (`CurrencyFormatException`, `CurrencyParseException`) to give callers precise error handling.  

2. **Null Handling**  
   * `setCurrency` can accept `null`; the other methods rely on a non‑null currency. Add checks or document the contract clearly.  

3. **Thread‑Safety**  
   * If the implementation caches `NumberFormat` instances (which are not thread‑safe), it may lead to concurrency issues. Clarify whether instances are meant to be shared or per‑thread.  

4. **Separation of Presentation**  
   * `getFormatedAmountWithCurrency(BigDecimal, String)` couples formatting with an HTML CSS class, blurring business logic and view logic. Consider moving CSS handling to the presentation layer or providing a separate method that returns the raw formatted string.  

5. **Locale Flexibility**  
   * The interface has no method to specify a `Locale`. In many systems the same currency may need to be formatted differently for different regions (e.g., € displayed as “1 234,56 €” vs “1,234.56 €”). Either expose a locale parameter or document that the implementation must infer locale from the `Currency`.  

6. **Naming Consistency**  
   * “Formated” is a typo; it should be “Formatted”.  
   * `getMeasure` is unclear – consider renaming to `formatMeasure` or `formatQuantity`.  

7. **Documentation**  
   * Javadoc comments are minimal and sometimes inaccurate (e.g., “will format the amount based on the appropriate currency” – but no mention of locale). Expand the comments to describe expected input formats, locale behaviour, and potential exceptions.  

### Future Enhancements  
* **Locale Parameter** – Add `Locale` to formatting methods for explicit control.  
* **Formatting Options** – Provide flags for grouping, decimal places, and symbol placement.  
* **Currency Repository** – Inject a `CurrencyService` to look up currency data rather than a setter.  
* **Immutability** – Make the module immutable by passing `Currency` in the constructor, reducing state mutations.  
* **Unit Tests** – Create a comprehensive test suite that verifies formatting across multiple currencies and locales.  

---  

**Overall Assessment**  
The interface defines a solid foundation for currency handling in an e‑commerce context. With some refinements—particularly around exception handling, naming, and presentation separation—the API would be more robust, easier to implement, and clearer for future developers.

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
package com.salesmanager.core.module.model.application;

import java.math.BigDecimal;

import com.salesmanager.core.entity.reference.Currency;

public interface CurrencyModule {

	/**
	 * Format a measure unit
	 * 
	 * @param measure
	 * @param currencycode
	 * @return
	 * @throws Exception
	 */
	public String getMeasure(BigDecimal measure, String currencycode)
			throws Exception;

	/**
	 * This method validates that the String amount complies with the currency
	 * and returns a BigDecimal for the given amount
	 * 
	 * @param amount
	 * @return
	 */
	public BigDecimal getAmount(String amount) throws Exception;

	/**
	 * Will format the amount based on the appropriate currency
	 * 
	 * @param amount
	 * @return
	 * @throws Exception
	 */
	public String getFormatedAmount(BigDecimal amount) throws Exception;

	/**
	 * Will format the amount and add the currency symbol where appropriate
	 * 
	 * @param amount
	 * @return
	 * @throws Exception
	 */
	public String getFormatedAmountWithCurrency(BigDecimal amount)
			throws Exception;

	/**
	 * Can be used to receive an HTML formated amount that can format the amount
	 * 
	 * @param amount
	 * @param className
	 * @return
	 * @throws Exception
	 */
	public String getFormatedAmountWithCurrency(BigDecimal amount,
			String amountCssClassName) throws Exception;

	public String getCurrencySymbol();

	/**
	 * Set currency entity when instantiating objects
	 * 
	 * @param currency
	 */
	public void setCurrency(Currency currency);

}



```
