# EURCurrencyModule.java

## Review

## 1. Summary  

The `EURCurrencyModule` is a concrete implementation of the `CurrencyModule` interface that handles formatting and parsing of Euro‑denominated amounts.  
* **Core responsibilities**  
  * Render amounts using German locale conventions (comma as decimal separator, dot as thousands separator).  
  * Append the Euro symbol (and an optional suffix, e.g., “EUR”).  
  * Parse user‑supplied strings back into `BigDecimal`.  
* **Key components**  
  * `NumberFormat` with `Locale.GERMAN` for formatting.  
  * Regular expression (`pattern`) to detect German‑style numbers.  
  * Apache Commons Validator (`CurrencyValidator`) for safe conversion to `BigDecimal`.  
  * Simple helper methods that assemble the final display string.  
* **Design**  
  * Very small, single‑responsibility class that can be swapped out for other currency modules.  
  * No complex patterns (Factory, Strategy) are visible beyond the interface implementation.  
  * Relies on standard Java SE (java.util, java.math) and two third‑party libraries (Apache Commons Lang & Validator).

---

## 2. Detailed Description  

### Initialization  
* A static `Pattern` is compiled once when the class is loaded.  
* The `EURCurrencyModule` holds two mutable fields:  
  * `currency` – a `Currency` entity (not used beyond fetching the suffix).  
  * `suffix` – optional string that can be appended after the formatted amount.

### Runtime Behaviour  
1. **Setting currency**  
   ```java
   setCurrency(Currency c)
   ```
   Stores the passed `Currency` and extracts its suffix (if any).  
2. **Formatting**  
   * `getMeasure(BigDecimal, String)` – formats a measurement value to **one** decimal place.  
   * `getFormatedAmount(BigDecimal)` – formats an amount to **two** decimal places.  
   * `getFormatedAmountWithCurrency(...)` – prepends the Euro symbol and optionally the suffix.  
3. **Parsing**  
   * `getAmount(String)` takes a string that may be in either plain or German‑style numeric form.  
   * If the string has no delimiters, it is treated as a positive integer; otherwise, the pattern is applied.  
   * Upon a successful match, the string is converted to a “standard” decimal form (comma → dot) and parsed with `CurrencyValidator`.  
4. **Utility**  
   * `getCurrencySymbol()` simply returns the Euro sign.

### Cleanup  
The class has no explicit cleanup – it is stateless after construction, apart from the stored `suffix`.

### Dependencies & Constraints  
* **Locale** – Hard‑coded `Locale.GERMAN`; changes would require code modification.  
* **Regular expression** – Assumes that any German‑style number matches the pattern; it does not account for optional sign or alternative grouping styles.  
* **Apache Commons Validator** – Used for safe conversion; any locale change would need adjusting the validator call.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `setCurrency(Currency)` | Stores a `Currency` entity and extracts its suffix. | `Currency` | `void` | mutates `currency` & `suffix` |
| `getSuffix()` | Exposes the optional suffix. | `-` | `String` | none |
| `getMeasure(BigDecimal, String)` | Formats a measurement value to one decimal place. | `measure`, `currencycode` (unused) | `String` (formatted value) | attempts to modify `measure` with `setScale` (but ineffective due to immutability) |
| `getAmount(String)` | Parses a numeric string into `BigDecimal`. | `amount` | `BigDecimal` | none; throws `ValidationException` on failure |
| `getFormatedAmount(BigDecimal)` | Formats amount to two decimal places. | `amount` | `String` | none |
| `getFormatedAmountWithCurrency(BigDecimal)` | Adds Euro symbol + optional suffix. | `amount` | `String` | none |
| `getFormatedAmountWithCurrency(BigDecimal, String)` | Same as above but wrapped in a `<font>` tag with a CSS class. | `amount`, `amountCssClassName` | `String` | none |
| `getCurrencySymbol()` | Returns the Euro sign. | `-` | `String` | none |

**Reusable / Utility methods** – None are extracted; all formatting logic is embedded in the public methods.

---

## 4. Dependencies  

| Library | Purpose | Std / 3rd‑party |
|---------|---------|-----------------|
| `java.math.BigDecimal` | Numeric precision | Std |
| `java.text.NumberFormat` | Locale‑aware formatting | Std |
| `java.util.Locale` | Locale constant | Std |
| `java.util.regex.Pattern/Matcher` | Detect German‑style numbers | Std |
| `org.apache.commons.lang.StringUtils` | Check for blank strings | 3rd‑party |
| `org.apache.commons.validator.routines.BigDecimalValidator` (imported but unused) | Safe conversion | 3rd‑party |
| `org.apache.commons.validator.routines.CurrencyValidator` | Convert string → BigDecimal respecting locale | 3rd‑party |
| `com.opensymphony.xwork2.validator.ValidationException` | Signal parse failures | 3rd‑party |
| `com.salesmanager.core.entity.reference.Currency` | Domain entity for currency | Application |
| `com.salesmanager.core.module.impl.common.CurrencyModuleUtil` | Helper to validate positive integers | Application |
| `com.salesmanager.core.module.model.application.CurrencyModule` | Interface implementation | Application |

All third‑party dependencies are Apache Commons and XWork, which are widely used and stable.

---

## 5. Additional Notes  

### Strengths  
* **Clear separation**: Parsing and formatting logic are in distinct methods.  
* **Locale‑aware**: Uses German formatting conventions correctly.  
* **Robustness**: Employs validator to avoid `NumberFormatException`.  

### Potential Issues & Edge Cases  

1. **Immutability bug**  
   ```java
   measure.setScale(1, BigDecimal.ROUND_HALF_UP);
   ```  
   `setScale` returns a new `BigDecimal`; the original `measure` remains unchanged. The formatted string will thus use the original scale.  
2. **Unused parameters**  
   * `currencycode` in `getMeasure` is never used.  
   * Imported `BigDecimalValidator` is unused.  
3. **Parsing limitations**  
   * The regex does not support negative numbers, optional grouping symbols, or leading/trailing spaces.  
   * `CurrencyModuleUtil.matchPositiveInteger(amount)` is assumed to check positivity, but its implementation is not shown.  
   * After replacing delimiters, the validator still uses `Locale.US`. This could misinterpret numbers that contain both comma and dot when the locale is German.  
4. **Hard‑coded locale**  
   * Switching to a different locale would require editing the source.  
5. **HTML safety**  
   * `getFormatedAmountWithCurrency(BigDecimal, String)` returns raw HTML. If `amountCssClassName` comes from user input, it could lead to XSS.  
6. **StringBuffer vs StringBuilder**  
   * Modern Java prefers `StringBuilder` for single‑threaded string assembly.  
7. **Suffix handling**  
   * `suffix` is appended only in the currency‑display methods; there is no check for null/empty.  

### Suggested Enhancements  

| Area | Recommendation |
|------|----------------|
| **Immutability** | Store the result of `setScale` in a local variable and use that for formatting. |
| **Method cleanup** | Remove unused parameters/variables and unused imports. |
| **Parsing robustness** | Use `NumberFormat`’s `parse` method with the German locale instead of regex + manual replacement. |
| **Locale flexibility** | Allow the locale to be injected (e.g., via constructor or setter) to support multiple currencies without code changes. |
| **HTML safety** | Escape `amountCssClassName` or avoid raw HTML generation; delegate rendering to a template engine. |
| **Testing** | Add unit tests covering edge cases: negative numbers, numbers with spaces, numbers with different grouping styles, null/empty suffix, etc. |
| **Performance** | The static `Pattern` is fine; consider making it `final` for clarity. |

Overall, the class fulfills its intended purpose but would benefit from a few small refactors and safety improvements to make it more robust, testable, and maintainable.

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
import java.text.NumberFormat;
import java.util.Locale;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.commons.lang.StringUtils;
import org.apache.commons.validator.routines.BigDecimalValidator;
import org.apache.commons.validator.routines.CurrencyValidator;

import com.opensymphony.xwork2.validator.ValidationException;
import com.salesmanager.core.entity.reference.Currency;
import com.salesmanager.core.module.impl.common.CurrencyModuleUtil;
import com.salesmanager.core.module.model.application.CurrencyModule;

public class EURCurrencyModule implements CurrencyModule {

	private final static char EURO = '\u20AC';

	private static Pattern pattern = Pattern
			.compile("\\d{1,3}(.?\\d{3})*(\\,\\d{1,2})");

	private Currency currency;
	private String suffix;

	public String getSuffix() {
		return suffix;
	}

	public void setCurrency(Currency currency) {
		this.currency = currency;
		if (!StringUtils.isBlank(currency.getSuffix())) {
			suffix = currency.getSuffix();
		}
	}

	public String getMeasure(BigDecimal measure, String currencycode)
			throws Exception {

		NumberFormat nf = null;

		nf = NumberFormat.getInstance(Locale.GERMAN);

		nf.setMaximumFractionDigits(1);
		nf.setMinimumFractionDigits(1);

		measure.setScale(1, BigDecimal.ROUND_HALF_UP);

		return nf.format(measure);
	}

	public BigDecimal getAmount(String amount) throws Exception {

		// Pattern pattern =
		// Pattern.compile("\\d{1,3}(?:(?:.\\d\\d\\d)*|\\d*)(?:\\,\\d\\d)?");

		if (!amount.contains(",") && !amount.contains(".")
				&& !amount.contains(",") && !amount.contains(" ")) {
			if (CurrencyModuleUtil.matchPositiveInteger(amount)) {
				BigDecimalValidator validator = CurrencyValidator.getInstance();
				BigDecimal bdamount = validator.validate(amount, Locale.US);
				if (bdamount == null) {
					throw new ValidationException("Cannot parse " + amount);
				} else {
					return bdamount;
				}
			} else {
				throw new ValidationException("Cannot parse " + amount);
			}
		} else {

			Matcher matcher = pattern.matcher(amount);

			if (matcher.matches()) {

				// switch comma and dots

				amount = amount.replaceAll(",", ":");
				amount = amount.replaceAll("\\.", ",");
				amount = amount.replaceAll(":", ".");

				BigDecimalValidator validator = CurrencyValidator.getInstance();
				// BigDecimal bdamount = validator.validate(amount,
				// Locale.GERMAN);//could do the job
				BigDecimal bdamount = validator.validate(amount, Locale.US);

				return bdamount;
			} else {
				throw new ValidationException("Cannot parse " + amount);
			}

		}
	}

	public String getFormatedAmount(BigDecimal amount) throws Exception {

		NumberFormat nf = null;

		nf = NumberFormat.getInstance(Locale.GERMAN);

		nf.setMaximumFractionDigits(2);
		nf.setMinimumFractionDigits(2);

		return nf.format(amount);

	}

	public String getFormatedAmountWithCurrency(BigDecimal amount)
			throws Exception {

		String returnamount = getFormatedAmount(amount);

		char display = EURO;

		StringBuffer ret = new StringBuffer().append(display).append("")
				.append(returnamount);
		if (this.getSuffix() != null) {
			ret.append(" ").append(this.getSuffix());
		}
		return ret.toString();
	}

	public String getFormatedAmountWithCurrency(BigDecimal amount,
			String amountCssClassName) throws Exception {

		String returnamount = getFormatedAmount(amount);

		char display = EURO;

		StringBuffer ret = new StringBuffer().append(display).append(
				"<font class='").append(amountCssClassName).append("'>")
				.append(returnamount);
		if (this.getSuffix() != null) {
			ret.append(" ").append(this.getSuffix());
		}

		ret.append("</font>");
		return ret.toString();
	}

	public String getCurrencySymbol() {
		return Character.toString(EURO);
	}

}



```
