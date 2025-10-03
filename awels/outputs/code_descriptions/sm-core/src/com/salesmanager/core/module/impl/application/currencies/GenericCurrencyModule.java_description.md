# GenericCurrencyModule.java

## Review

## 1. Summary  

`GenericCurrencyModule` is a concrete implementation of the `CurrencyModule` interface used by the SalesManager core to handle currency‑specific formatting and parsing.  
It stores a `Currency` entity and exposes the following responsibilities:

| Responsibility | Key method |
|----------------|------------|
| **Currency configuration** | `setCurrency(Currency)` |
| **String → BigDecimal conversion** | `getAmount(String)` |
| **Formatting a BigDecimal into a locale‑aware string** | `getFormatedAmount(BigDecimal)`, `getFormatedAmountWithCurrency(BigDecimal)`, `getFormatedAmountWithCurrency(BigDecimal, String)` |
| **Formatting a measurement value** | `getMeasure(BigDecimal, String)` |
| **Decoding the currency symbol** | `getCurrencySymbol()` |

The class relies on a handful of third‑party libraries (Apache Commons Lang, Commons Validator, XWork) and uses standard Java `Locale`, `NumberFormat`, and regex facilities to perform the heavy lifting.

---

## 2. Detailed Description  

### 2.1 Core Components  

| Field | Purpose |
|-------|---------|
| `currency` | The active `Currency` entity (contains symbol, suffix, decimal places, etc.). |
| `s` | Raw symbol string (may contain unicode escape sequences). |
| `decimalCount`, `decimalPoint`, `thousandPoint` | Locale‑specific numeric formatting rules. |
| `suffix` | Optional postfix text (e.g., “USD”). |

The class is deliberately *stateful* – the `setCurrency()` method configures the instance for a particular currency, and all subsequent operations are based on that state.

### 2.2 Execution Flow  

1. **Initialization** – `setCurrency(Currency)` copies fields from the supplied `Currency` entity.  
   *Note*: the current code sets `suffix` twice, once unconditionally, which overwrites any earlier conditional assignment.*

2. **Parsing (`getAmount`)**  
   - Strips all decimal/thousand separators from the input string and validates that the remainder is an integer.  
   - If the input contains no separators or spaces, it is assumed to be a plain integer and validated with `CurrencyValidator`.  
   - Otherwise a custom regex is built that matches the expected thousands/decimal pattern for the configured locale.  
   - The string is then parsed into a `BigDecimal` using `CurrencyValidator.validate()`.  

3. **Formatting**  
   - `getFormatedAmount(BigDecimal)` obtains a `NumberFormat` for the appropriate locale (US or GERMAN) and applies the configured fraction digits.  
   - `getFormatedAmountWithCurrency` prefixes the formatted number with the decoded symbol (and suffix if present).  
   - `getFormatedAmountWithCurrency(BigDecimal, String)` additionally wraps the numeric part in a `<font>` tag with a CSS class.  
   - `getMeasure` behaves like `getFormatedAmount` but forces a single fractional digit and rounds half‑up.

4. **Utility (`getCurrencySymbol`)**  
   Decodes `\uXXXX` unicode escapes manually, building a plain string representation of the symbol.

### 2.3 Assumptions & Constraints  

| Assumption | Implication |
|------------|-------------|
| `decimalCount` is a single digit `'0'..'9'` | Used as an int by parsing the char; fails if it contains a multi‑digit value or is not numeric. |
| Only US and German locales are supported | No internationalization beyond these two locales. |
| Input strings contain no spaces unless they are part of the thousands separator | `getAmount()` rejects any string that mixes spaces with numeric characters. |
| `setCurrency()` is called once before any other operation | The class is not thread‑safe; concurrent use could lead to inconsistent state. |

### 2.4 Architectural Choices  

*The design is a simple “module” pattern:*
- **Encapsulation** – All formatting logic is hidden behind the interface.  
- **Reusability** – The same instance can be reused for a single currency; new instances are created for different currencies.  
- **Extensibility** – Additional locales can be added by extending the regex logic and locale selection.  
- **Coupling** – The class depends heavily on `CurrencyValidator` and `NumberFormat`; these are well‑tested libraries but add runtime overhead.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side Effects / Notes |
|--------|---------|------------|---------|----------------------|
| `public void setCurrency(Currency currency)` | Configures the module for a particular currency. | `currency` | void | Overwrites `suffix` twice (bug). |
| `public BigDecimal getAmount(String amount)` | Parses a formatted string into a `BigDecimal`. | `amount` | `BigDecimal` | Throws `ValidationException` on invalid format. |
| `public String getCurrencySymbol()` | Decodes the currency symbol from the raw string. | – | `String` | Manual unicode escape handling. |
| `public String getFormatedAmount(BigDecimal amount)` | Formats a number according to locale/fraction rules. | `amount` | `String` | Throws `Exception` (unchecked). |
| `public String getFormatedAmountWithCurrency(BigDecimal amount)` | Same as above but prefixes symbol and suffix. | `amount` | `String` | Adds no space between symbol and amount. |
| `public String getFormatedAmountWithCurrency(BigDecimal amount, String amountCssClassName)` | Same as above but wraps numeric part in a `<font>` tag. | `amount`, `amountCssClassName` | `String` | Uses deprecated `<font>` tag. |
| `public String getMeasure(BigDecimal measure, String currencycode)` | Formats a measurement value (single decimal). | `measure`, `currencycode` | `String` | Unused `currencycode` parameter. |
| **Utility**: regex construction inside `getAmount` | Build a pattern that matches the expected thousands/decimal separators. | – | – | Hard‑coded for US/German. |

*Reusability*: `getFormatedAmount` can be used directly if no symbol is required; all other formatting methods delegate to it.

---

## 4. Dependencies  

| Library | Type | Role |
|---------|------|------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String blank checks. |
| `org.apache.commons.validator.routines.BigDecimalValidator` | Third‑party | Validates numeric strings. |
| `org.apache.commons.validator.routines.CurrencyValidator` | Third‑party | Parses currency strings with locale awareness. |
| `com.opensymphony.xwork2.validator.ValidationException` | Third‑party | Exception type for validation failures. |
| `com.salesmanager.core.entity.reference.Currency` | Project | Holds currency metadata. |
| `com.salesmanager.core.module.impl.common.CurrencyModuleUtil` | Project | Provides `matchPositiveInteger` helper. |
| `com.salesmanager.core.module.model.application.CurrencyModule` | Project | Interface contract. |
| `java.util.Locale`, `java.util.regex`, `java.math.BigDecimal`, `java.text.NumberFormat` | Standard | Core Java functionality. |

All dependencies are stable and widely used; no platform‑specific constraints beyond standard JRE support.

---

## 5. Additional Notes  

### 5.1 Edge Cases & Limitations  

| Edge Case | Current Behaviour | Suggested Fix |
|-----------|-------------------|---------------|
| `amount` contains **spaces** (e.g., `"1 234,56"`) | Rejected with `ValidationException`. | Allow optional whitespace or normalize input. |
| `decimalCount` > 9 or not numeric | Parsing `Integer.parseInt(Character.toString(decimalCount))` fails or yields incorrect digits. | Store `decimalCount` as `int` instead of `char`. |
| `suffix` overwritten by unconditional assignment | Last value always wins, even if originally `null`. | Use `if (!StringUtils.isBlank(currency.getSuffix())) { suffix = currency.getSuffix(); }` |
| `thousandPoint` equal to `decimalPoint` | Regex may produce incorrect pattern, causing parse failures. | Validate that the two chars differ. |
| `getMeasure` ignores `currencycode` | Unused parameter might indicate future use. | Remove or document its purpose. |
| Deprecated `<font>` tag | Modern HTML/CSS best practices discourage `<font>`. | Replace with `<span>` and CSS styling. |
| Not thread‑safe | Concurrent calls with different currencies could corrupt state. | Make the class immutable or synchronize access. |

### 5.2 Potential Enhancements  

1. **Use `DecimalFormat` directly**  
   - Build a `DecimalFormat` pattern based on `decimalPoint`, `thousandPoint`, and `decimalCount`.  
   - Simplifies parsing and formatting, removes the need for regex and external validators.

2. **Locale‑aware formatting**  
   - Accept a `Locale` parameter or expose a method to set the locale explicitly.  
   - Avoid hard‑coding US/German; support other common locales.

3. **Unicode symbol handling**  
   - Replace manual decoding with `StringEscapeUtils.unescapeJava` (Commons Lang 3).  
   - Handles all escape sequences and is battle‑tested.

4. **Error handling**  
   - Replace generic `Exception` throws with custom, checked exceptions (`CurrencyParseException`).  
   - Provide clearer diagnostics (e.g., which part of the format failed).

5. **Validation utilities**  
   - Move `matchPositiveInteger` into this class or use `BigDecimalValidator` consistently.  
   - Avoid double validation: once the regex matches, the validator can be trusted.

6. **Unit Tests**  
   - Add comprehensive test coverage for various locales, edge inputs, and negative numbers.  
   - Ensure regression protection for the bug with `suffix`.

7. **Documentation & Javadoc**  
   - Add method level Javadoc, especially for public API.  
   - Document the expected format of the `amount` string.

### 5.3 Code Quality Observations  

| Issue | Impact | Fix |
|-------|--------|-----|
| Unused parameter `currencycode` in `getMeasure` | Clutters method signature | Remove or document usage |
| Hard‑coded `NumberFormat.getInstance(locale)` | Locale limited to US/German | Allow custom `Locale` or build pattern |
| Manual string builder for `<font>` | Deprecated HTML, readability | Use `StringBuilder` with `<span>` or CSS |
| No `@Override` annotations | Potential accidental method overrides | Add annotations for all interface implementations |
| Inconsistent exception types (`ValidationException` vs generic `Exception`) | Confuses callers | Use a consistent exception hierarchy |

---

### 5.4 Final Recommendation  

The `GenericCurrencyModule` fulfils its basic purpose, but several design and implementation issues reduce maintainability and robustness:

* **Refactor** to use immutable state and `DecimalFormat` for all parsing/formatting.  
* **Remove deprecated HTML** and provide a CSS‑friendly API.  
* **Fix the `suffix` bug** and replace character‑based numeric settings with integer fields.  
* **Add comprehensive unit tests** covering all supported locales and edge cases.

Once these improvements are in place, the module will be more reliable, easier to extend, and aligned with modern Java best practices.

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

/**
 * Needed to implement other currencies
 * 
 * @author Carl Samson
 * 
 */
public class GenericCurrencyModule implements CurrencyModule {

	private Currency currency;
	private String s = "";
	private char decimalCount = '2';
	private char decimalPoint = '.';
	private char thousandPoint = ',';
	private String suffix;

	public String getSuffix() {
		return suffix;
	}

	public void setCurrency(Currency currency) {
		this.currency = currency;
		s = currency.getSymbol();
		if (!StringUtils.isBlank(currency.getSuffix())) {
			suffix = currency.getSuffix();
		}
		suffix = currency.getSuffix();
		decimalCount = currency.getDecimalPlaces();
		decimalPoint = currency.getDecimalPoint();
		thousandPoint = currency.getThousandsPoint();
	}

	public BigDecimal getAmount(String amount) throws Exception {

		// validations
		/**
		 * 1) remove decimal and thousand
		 * 
		 * String.replaceAll(decimalPoint, ""); String.replaceAll(thousandPoint,
		 * "");
		 * 
		 * Should be able to parse to Integer
		 */
		StringBuffer newAmount = new StringBuffer();
		for (int i = 0; i < amount.length(); i++) {
			if (amount.charAt(i) != decimalPoint
					&& amount.charAt(i) != thousandPoint) {
				newAmount.append(amount.charAt(i));
			}
		}

		try {
			Integer.parseInt(newAmount.toString());
		} catch (Exception e) {
			throw new ValidationException("Cannot parse " + amount);
		}

		if (!amount.contains(Character.toString(decimalPoint))
				&& !amount.contains(Character.toString(thousandPoint))
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
				throw new ValidationException("Not a positive integer "
						+ amount);
			}

		} else {

			StringBuffer pat = new StringBuffer();

			if (!StringUtils.isBlank(Character.toString(thousandPoint))) {
				pat.append("\\d{1,3}(" + thousandPoint + "?\\d{3})*");
			}

			pat.append("(\\" + decimalPoint + "\\d{1," + decimalCount + "})");

			Pattern pattern = Pattern.compile(pat.toString());

			Matcher matcher = pattern.matcher(amount);

			if (matcher.matches()) {

				Locale locale = Locale.US;

				if (this.decimalPoint == ',') {
					locale = Locale.GERMAN;
				}

				BigDecimalValidator validator = CurrencyValidator.getInstance();
				BigDecimal bdamount = validator.validate(amount, locale);

				return bdamount;
			} else {
				throw new ValidationException("Cannot parse " + amount);
			}
		}

	}

	public String getCurrencySymbol() {
		// TODO Auto-generated method stub
		int i = 0, len = s.length();
		char c;
		StringBuffer sb = new StringBuffer(len);

		while (i < len) {
			c = s.charAt(i++);
			if (c == '\\') {
				if (i < len) {
					c = s.charAt(i++);
					if (c == 'u') {
						c = (char) Integer.parseInt(s.substring(i, i + 4), 16);
						i += 4;
					} // add other cases here as desired...
				}
			} // fall through: \ escapes itself, quotes any character but u
			sb.append(c);
		}
		return sb.toString();
	}

	public String getFormatedAmount(BigDecimal amount) throws Exception {
		// TODO Auto-generated method stub
		NumberFormat nf = null;

		Locale locale = Locale.US;

		if (this.decimalPoint == ',') {
			locale = Locale.GERMAN;
		}

		nf = NumberFormat.getInstance(locale);

		nf.setMaximumFractionDigits(Integer.parseInt(Character
				.toString(decimalCount)));
		nf.setMinimumFractionDigits(Integer.parseInt(Character
				.toString(decimalCount)));

		return nf.format(amount);
	}

	public String getFormatedAmountWithCurrency(BigDecimal amount)
			throws Exception {
		// TODO Auto-generated method stub
		String returnamount = getFormatedAmount(amount);

		StringBuffer ret = new StringBuffer().append(this.getCurrencySymbol())
				.append("").append(returnamount);
		if (this.getSuffix() != null) {
			ret.append(" ").append(this.getSuffix());
		}
		return ret.toString();

	}

	public String getFormatedAmountWithCurrency(BigDecimal amount,
			String amountCssClassName) throws Exception {
		// TODO Auto-generated method stub
		String returnamount = getFormatedAmount(amount);

		StringBuffer ret = new StringBuffer().append(this.getCurrencySymbol())
				.append("<font class='").append(amountCssClassName)
				.append("'>").append(returnamount);
		if (this.getSuffix() != null) {
			ret.append(" ").append(this.getSuffix());
		}

		ret.append("</font>");
		return ret.toString();
	}

	public String getMeasure(BigDecimal measure, String currencycode)
			throws Exception {

		NumberFormat nf = null;

		Locale locale = Locale.US;

		if (this.decimalPoint == ',') {
			locale = Locale.GERMAN;
		}

		nf = NumberFormat.getInstance(locale);

		nf.setMaximumFractionDigits(1);
		nf.setMinimumFractionDigits(1);

		measure.setScale(1, BigDecimal.ROUND_HALF_UP);

		return nf.format(measure);
	}

}



```
