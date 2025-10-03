# USDCurrencyModule.java

## Review

## 1. Summary  

`USDCurrencyModule` implements the `CurrencyModule` interface to provide USD‑specific
parsing, formatting, and symbol handling.  
Key responsibilities:

| Feature | Description |
|---------|-------------|
| **Formatting** | `getFormatedAmount`, `getFormatedAmountWithCurrency`, and the CSS‑variant format return a locale‑aware string with two decimal places and a leading “$”. |
| **Parsing** | `getAmount` converts a String into a `BigDecimal` using Apache Commons `CurrencyValidator`. |
| **Measure formatting** | `getMeasure` is a helper that returns a one‑decimal representation of a measure (e.g., quantity). |
| **Currency meta** | Holds a `Currency` object and optionally a suffix (e.g., “USD”) that is appended to formatted amounts. |
| **Symbol** | `getCurrencySymbol` simply returns “$”. |

The class relies on `java.text.NumberFormat` for locale‑aware number formatting and on
`org.apache.commons.validator` for validation.  The design is straightforward: it
acts as a stateless service object that can be injected wherever a USD currency
module is required.

---

## 2. Detailed Description  

### Initialization  
No constructor is defined; the class is instantiated with the default no‑arg
constructor.  The only mutable state is the `Currency` and optional `suffix`
fields, which are set via `setCurrency`.  No thread‑safety guarantees are
documented, but all fields are simple strings, so the class is effectively
immutable once configured.

### Runtime Behaviour  

| Method | What it does | Dependencies | Notes |
|--------|--------------|--------------|-------|
| `getMeasure` | Formats a `BigDecimal` as a string with one decimal place. | `NumberFormat` | Uses `BigDecimal.ROUND_HALF_UP` (deprecated in recent JDKs). |
| `getAmount` | Parses a string to a `BigDecimal`.  It first checks for a plain integer; if the string contains commas, periods, or spaces, it uses a regex pattern to validate the format and then delegates to `CurrencyValidator`. | `Pattern`, `Matcher`, `CurrencyValidator` | Does **not** trim input, does not support negative values, and ignores locale beyond `Locale.US`. |
| `getFormatedAmount` | Formats a `BigDecimal` to a string with two decimals. | `NumberFormat` | Straightforward; could cache the formatter. |
| `getFormatedAmountWithCurrency` (2 overloads) | Adds the “$” symbol (and optionally a CSS class or suffix) around the formatted amount. | `StringBuffer` | Uses raw HTML; could use `StringBuilder`. |
| `getCurrencySymbol` | Returns “$”. | None | Simple accessor. |

### Cleanup  
No external resources are opened or held; cleanup is unnecessary.

### Assumptions & Constraints  

* All numbers are processed with `Locale.US`; no support for other locales is provided.  
* Input strings are expected to be well‑formatted USD values; malformed strings raise a `ValidationException`.  
* The class is not thread‑safe with respect to `setCurrency`; calling it concurrently may lead to inconsistent state.  
* Only positive numbers are supported; negative amounts are rejected silently by the validator.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return Type | Side Effects | Notes |
|--------|---------|------------|-------------|--------------|-------|
| `getSuffix()` | Accessor for optional currency suffix. | – | `String` | – | – |
| `setCurrency(Currency currency)` | Stores a `Currency` instance and extracts its suffix if present. | `Currency` | `void` | Sets internal fields. | No null‑check on `currency`; can throw `NullPointerException` if passed null. |
| `getMeasure(BigDecimal measure, String currencycode)` | Formats `measure` with one decimal place. | `measure`, `currencycode` (unused) | `String` | None | `currencycode` is ignored; could be removed. Uses deprecated rounding mode. |
| `getAmount(String amount)` | Parses a numeric string into a `BigDecimal`. | `amount` | `BigDecimal` | Throws `ValidationException` on parse failure. | No trimming; negative values fail. |
| `getFormatedAmount(BigDecimal amount)` | Formats amount with two decimals. | `amount` | `String` | None | Uses `Locale.US`. |
| `getFormatedAmountWithCurrency(BigDecimal amount)` | Prepends “$” and optionally suffix. | `amount` | `String` | None | Uses `StringBuffer`. |
| `getFormatedAmountWithCurrency(BigDecimal amount, String amountCssClassName)` | Same as above but wraps amount in an HTML `<font>` tag. | `amount`, `amountCssClassName` | `String` | None | Potentially insecure if `amountCssClassName` is not sanitized. |
| `getCurrencySymbol()` | Returns the symbol used for this module. | – | `String` | None | Always “$”. |

Reusable utility methods:  
* `getFormatedAmount` can be reused for any currency that uses two‑decimal formatting.

---

## 4. Dependencies  

| Library | Purpose | Standard / 3rd‑party |
|---------|---------|----------------------|
| `java.math.BigDecimal` | Arbitrary‑precision decimal arithmetic. | Standard |
| `java.text.NumberFormat` | Locale‑aware number formatting. | Standard |
| `java.util.Locale` | Locale constants. | Standard |
| `java.util.regex.Pattern`, `Matcher` | Regular expression validation. | Standard |
| `org.apache.commons.lang.StringUtils` | String utilities (isBlank). | 3rd‑party |
| `org.apache.commons.validator.routines.BigDecimalValidator`, `CurrencyValidator` | Parsing and validation of numeric strings. | 3rd‑party |
| `com.opensymphony.xwork2.validator.ValidationException` | Exception signalling validation errors. | 3rd‑party (XWork) |
| `com.salesmanager.core.entity.reference.Currency` | Domain entity for currency metadata. | Domain (custom) |
| `com.salesmanager.core.module.impl.common.CurrencyModuleUtil` | Helper method `matchPositiveInteger`. | Domain (custom) |
| `com.salesmanager.core.module.model.application.CurrencyModule` | Interface being implemented. | Domain (custom) |

All third‑party libraries are well‑established. No platform‑specific assumptions are evident; the code runs on any JVM that provides the standard API and the referenced libraries.

---

## 5. Additional Notes  

### Edge Cases & Limitations  

1. **Negative Values** – `CurrencyValidator` will reject negative numbers; the module does not support them.  
2. **Whitespace & Formatting** – Inputs are not trimmed; “  1234” will throw an exception.  
3. **Locale Flexibility** – The module is hardcoded to `Locale.US`. Other locales (e.g., “de_DE” where commas are decimals) are unsupported.  
4. **Thread‑Safety** – Mutating `setCurrency` in a concurrent environment may lead to race conditions.  
5. **Deprecated API** – `BigDecimal.ROUND_HALF_UP` is deprecated; use `RoundingMode.HALF_UP`.  
6. **String Concatenation** – `StringBuffer` is used unnecessarily; `StringBuilder` or `String.format` would be more efficient.  
7. **HTML Injection** – `amountCssClassName` is concatenated directly into an HTML string; if user‑supplied, it could lead to XSS.  
8. **Unnecessary Parameters** – `getMeasure` accepts a `currencycode` argument that is never used; it can be removed to simplify the API.

### Potential Enhancements  

* **Locale Support** – Add a constructor or setter to specify a `Locale` for formatting/parsing.  
* **Negative & Zero Handling** – Provide configuration to allow or disallow negative amounts.  
* **Thread‑Safe Configuration** – Make the class immutable or guard state changes with synchronization.  
* **Refactor Formatting** – Cache `NumberFormat` instances or use `DecimalFormat` with pattern strings to avoid repeated creation.  
* **Improve Validation** – Trim input, allow optional spaces, and provide more granular error messages.  
* **Replace Deprecated API** – Update rounding mode usage.  
* **Remove `StringBuffer`** – Switch to `StringBuilder` or `String.format` for clarity.  
* **Remove Dead Code** – Eliminate the unused `currencycode` parameter.  
* **Unit Tests** – Add comprehensive tests covering normal, boundary, and error cases.  

### Suggested Code Snippets  

```java
// Improved getAmount
public BigDecimal getAmount(String amount) throws ValidationException {
    if (StringUtils.isBlank(amount)) {
        throw new ValidationException("Amount cannot be null or empty");
    }
    String trimmed = amount.trim();

    BigDecimalValidator validator = CurrencyValidator.getInstance();
    BigDecimal bd = validator.validate(trimmed, Locale.US);
    if (bd == null) {
        throw new ValidationException("Cannot parse " + amount);
    }
    return bd;
}
```

```java
// Caching NumberFormat
private static final ThreadLocal<NumberFormat> USD_FMT =
    ThreadLocal.withInitial(() -> {
        NumberFormat nf = NumberFormat.getCurrencyInstance(Locale.US);
        nf.setMinimumFractionDigits(2);
        nf.setMaximumFractionDigits(2);
        return nf;
    });

public String getFormatedAmount(BigDecimal amount) {
    return USD_FMT.get().format(amount);
}
```

Overall, the class fulfills its role for USD currency handling but would benefit from minor refactoring, improved robustness, and better adherence to modern Java conventions.

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

public class USDCurrencyModule implements CurrencyModule {

	protected final static char DOLLAR = '\u0024';

	private static Pattern pattern = Pattern
			.compile("\\d{1,3}(,?\\d{3})*(\\.\\d{1,2})");

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

		nf = NumberFormat.getInstance(Locale.US);

		nf.setMaximumFractionDigits(1);
		nf.setMinimumFractionDigits(1);

		measure.setScale(1, BigDecimal.ROUND_HALF_UP);

		return nf.format(measure);

	}

	public BigDecimal getAmount(String amount) throws Exception {

		if (!amount.contains(",") && !amount.contains(".")
				&& !amount.contains(" ")) {

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
				BigDecimalValidator validator = CurrencyValidator.getInstance();
				BigDecimal bdamount = validator.validate(amount, Locale.US);

				return bdamount;
			} else {
				throw new ValidationException("Cannot parse " + amount);
			}
		}
	}

	public String getFormatedAmount(BigDecimal amount) throws Exception {

		NumberFormat nf = null;

		nf = NumberFormat.getInstance(Locale.US);

		nf.setMaximumFractionDigits(2);
		nf.setMinimumFractionDigits(2);

		return nf.format(amount);

	}

	public String getFormatedAmountWithCurrency(BigDecimal amount)
			throws Exception {

		String returnamount = getFormatedAmount(amount);

		char display = DOLLAR;

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

		char display = DOLLAR;

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
		return Character.toString(DOLLAR);
	}

}



```
