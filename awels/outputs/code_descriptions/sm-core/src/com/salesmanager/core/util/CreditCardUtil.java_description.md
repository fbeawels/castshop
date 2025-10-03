# CreditCardUtil.java

## Review

## 1. Summary  
`CreditCardUtil` is a small utility class that provides a set of static helpers for working with credit‑card data in the *SalesManager* code base.  Its responsibilities include:

| Responsibility | What it does | Key methods |
|----------------|--------------|-------------|
| **Masking** | Obscures all but the first four and last four digits of a card number | `maskCardNumber(String)` |
| **Validation** | Verifies that a card number matches the expected length & prefix for a given card type, that the expiry date is not in the past, and that the CVV is the correct length for the card type | `validate(String, int, String, String)`, `validateCvv(String, int)`, `validateNumber(String, int)`, `validateDate(int, int)` |
| **Checksum** | Implements the Luhn algorithm to catch entry errors | `luhnValidate(String)` |
| **UI Support** | Supplies the image file names for supported card icons (used by the front‑end) | `getCreditCardStripImages()` |

The class uses the **Apache Commons Lang** `StringUtils` helper, a custom `LabelUtil` for i18n messages, and a `RefCache` that holds the set of supported credit cards.

---

## 2. Detailed Description  

### Core Flow

1. **Input** – A credit‑card number (with optional spaces, dashes or dots), the card type (represented by a constant), the expiry month and year, and the CVV.
2. **Sanitisation** – `validate()` strips non‑numeric characters from the number and checks that the month & year are numeric.
3. **Date Validation** – `validateDate()` compares the supplied month/year against the current date, ensuring the card has not expired.
4. **Card‑type Validation** – `validateNumber()` checks length/prefix against the chosen card type.
5. **Checksum Validation** – `luhnValidate()` confirms the number satisfies the Luhn algorithm.
6. **CVV Validation** – `validateCvv()` ensures the CVV is numeric and of the correct length for the card type.
7. **Masking** – `maskCardNumber()` hides sensitive digits when the card number needs to be stored/displayed.

### Design Choices & Assumptions

| Choice | Reason | Comment |
|--------|--------|---------|
| **Integer constants for card types** | Avoids string comparison overhead | Replaces the now‑common `enum` pattern; hard to extend |
| **Non‑generics raw `Map`/`List`** | Legacy code style | Causes unchecked warnings and loss of type safety |
| **`LabelUtil` for messages** | Centralises localisation | Re‑calling `LabelUtil.getInstance()` repeatedly is unnecessary |
| **Fixed “XXXXXXXXXX” mask** | Simple placeholder | Does not adapt to card numbers shorter than 18 digits |
| **Luhn algorithm implementation** | Standard, well‑tested | Works for all card types but is called for every card, regardless of earlier validation failure |

The class is a mix of static and instance methods (only `validate()` is non‑static).  This inconsistency is a stylistic issue rather than a functional bug but could lead to confusion.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side Effects / Exceptions |
|--------|---------|------------|---------|---------------------------|
| `public static String maskCardNumber(String clearcardnumber)` | Returns a masked card number showing only the first 4 and last 4 digits | `clearcardnumber` – raw card number | Masked string | Throws `CreditCardUtilException` if length < 10 |
| `public void validate(String number, int type, String month, String date)` | Validates a full card entry (number, type, expiry, CVV) | `number` – card number (may contain spaces/dashes) <br> `type` – card type constant <br> `month`, `date` – expiry month/year as strings | none | Throws `CreditCardUtilException` on any validation error |
| `private void validateDate(int m, int y)` | Checks expiry date against current month/year | `m`, `y` – parsed month/year | none | Throws `CreditCardUtilException` if expired |
| `public void validateCvv(String cvvNumber, int type)` | Checks CVV numeric content & length for a card type | `cvvNumber` – CVV <br> `type` – card type | none | Throws `CreditCardUtilException` on failure |
| `private void validateNumber(String number, int type)` | Ensures card number length & prefix match card type | `number` – cleaned numeric string <br> `type` – card type | none | Throws `CreditCardUtilException` if mismatched |
| `private void luhnValidate(String numberString)` | Implements the Luhn checksum algorithm | `numberString` – numeric card number | none | Throws `CreditCardUtilException` if checksum fails |
| `public static List<String> getCreditCardStripImages()` | Builds a list of image filenames for all supported card types | none | `List<String>` of image names | None |

### Reusable / Utility Methods
* `luhnValidate` can be reused wherever a generic Luhn check is needed.
* `validateCvv` is card‑type specific but the logic for numeric‑only validation could be extracted.

---

## 4. Dependencies  

| Library / Package | Type | Notes |
|-------------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Legacy Apache Commons Lang (`lang` 2.x).  Modern code prefers `org.apache.commons.lang3.StringUtils`. |
| `com.salesmanager.core.entity.reference.CentralCreditCard` | Project | Domain entity holding card metadata. |
| `com.salesmanager.core.service.cache.RefCache` | Project | Provides cached map of supported credit cards. |
| `com.salesmanager.core.util.LabelUtil` | Project | i18n helper that supplies error messages. |
| Java SE (`java.util`, `java.util.regex`) | Standard | Used for collections and regex. |

No platform‑specific code is present; the class is pure Java and should run on any JVM.

---

## 5. Additional Notes & Recommendations  

### Strengths
* Provides a single, reusable point for all card‑related validation.
* Luhn algorithm implementation is straightforward and efficient.
* Uses a cache for card metadata, avoiding repeated database look‑ups.

### Weaknesses & Edge‑Cases
1. **Hard‑coded constants**  
   * Card‑type identifiers are raw integers; an `enum` would be safer and self‑documenting.  
   * `maskCardNumber()` always inserts 10 “X” characters regardless of card length; for a 13‑digit Visa this yields a 17‑character result, which may be confusing.

2. **Limited prefix support**  
   * Discover cards can start with 6011, 622126‑622925, 644‑649, or 65; the current logic only accepts 6011.  
   * Diners’ logic has a precedence bug: `((A != 36 && B != 38) && C < 300 || C > 305)` actually evaluates as `((A != 36 && B != 38) && C < 300) || C > 305`.  It fails to correctly reject numbers like 36300000000000.  
   * American Express corporate codes (378734493671000) are never explicitly validated; the `AMEX` branch only checks prefix 34/37.

3. **Date validation is incomplete**  
   * Only checks that the year is not in the past and that the month is not before the current month when the year is equal.  
   * Does not validate that the month value is in 1‑12 range or that the day is provided.

4. **Non‑generics and unchecked casts**  
   * `RefCache.getSupportedCreditCards()` returns a raw `Map`, leading to unchecked casts and warnings.  
   * `getCreditCardStripImages()` returns a raw `List`; callers must cast to `List<String>`.

5. **Mixed static / instance usage**  
   * `validate()` is the only non‑static method.  All others are static, so an instance must be created solely to call `validate()`, which is confusing.

6. **Error handling**  
   * All validation errors throw `CreditCardUtilException` with a generic message that is fetched at runtime.  The message lookup could be cached or injected to avoid repeated lookups.

7. **Performance**  
   * Repeated construction of regex `Pattern` objects on each call to `validate()` and `validateCvv()`.  Compiling the patterns once (static final) would be more efficient.

8. **Internationalization**  
   * The class relies on `LabelUtil.getInstance()` for every error message, which may incur overhead in high‑traffic scenarios.

### Suggested Improvements

| Issue | Fix / Enhancement |
|-------|-------------------|
| Card‑type representation | Replace integer constants with an `enum` (`enum CardType { VISA, MASTERCARD, AMEX, DISCOVER, DINERS }`). |
| Masking | Use `String.format("%sXXXXXXXXXX%s", prefix, suffix)` only if `number.length() > 10`, otherwise mask the entire number. |
| Prefix validation | Expand the switch logic to fully support all known prefixes (especially Discover).  Consider using regex patterns per card type. |
| Luhn check | Keep as is, but expose it as a public static method so it can be reused elsewhere. |
| Date validation | Validate month in 1‑12 and optionally include day.  Throw a distinct exception for invalid date formats. |
| Generics | Update `getCreditCardStripImages()` to return `List<String>` and cast the cached map with generics. |
| Static patterns | Declare `private static final Pattern NON_NUMERIC = Pattern.compile("[^\\d\\s.-]");` and similar for other regexes. |
| Method staticness | Make all methods static or expose a singleton instance; remove unnecessary object creation. |
| Logging | Add optional logging of validation failures for audit purposes (but avoid leaking card data). |
| Unit tests | Provide comprehensive JUnit tests covering all card types, edge cases, and failure scenarios. |
| Documentation | Add Javadoc comments explaining each public method, parameters, and possible exceptions. |

### Potential Future Enhancements
* **CVV length per card type**: Some issuers use 4‑digit CVV for Amex and 3‑digit for others; this is already handled but could be data‑driven via configuration.  
* **Support for new card schemes**: PCI‑DSS changes and emerging card types can be added without touching the core logic.  
* **Card number formatting**: Provide a helper that formats numbers into groups (e.g., `1234 5678 9012 3456`) for display purposes.  
* **Integration with a payment gateway**: The class could expose a single method that validates *and* tokenises a card number using a gateway API.  

---

### Final Verdict  

`CreditCardUtil` implements the essential business rules for credit‑card validation in a clear, concise way, but its design is a bit brittle and outdated.  Modernizing the class with enums, generics, and improved validation logic would make it safer, easier to maintain, and better aligned with current Java best practices.

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
package com.salesmanager.core.util;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.entity.reference.CentralCreditCard;
import com.salesmanager.core.service.cache.RefCache;

/**
 * Tests credit cards 
 * Master Card (16 Digits) 5105105105105100 
 * Master Card (16 Digits) 5555555555554444 
 * Visa (13 Digits) 4222222222222 
 * Visa (16 Digits) 4111111111111111 
 * Visa (16 Digits) 4012888888881881 
 * American Express (15 Digits) 378282246310005 
 * American Express (15 Digits) 371449635398431 
 * Amex Corporate (15 Digits) 378734493671000 
 * Dinners Club (14 Digits) 38520000023237
 * Dinners Club (14 Digits) 30569309025904 
 * Discover (16 Digits) 6011111111111117
 * Discover (16 Digits) 6011000990139424 
 * JCB (16 Digits) 3530111333300000 
 * JCB (16 Digits) 3566002020360505
 * 
 * @author Administrator
 * 
 */
public class CreditCardUtil {

	public static final int MASTERCARD = 0, VISA = 1;
	public static final int AMEX = 2, DISCOVER = 3, DINERS = 4;

	public static String maskCardNumber(String clearcardnumber)
			throws CreditCardUtilException {

		if (clearcardnumber.length() < 10) {
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invalidnumber"));
		}

		int length = clearcardnumber.length();

		String prefix = clearcardnumber.substring(0, 4);
		String suffix = clearcardnumber.substring(length - 4);

		StringBuffer mask = new StringBuffer();
		mask.append(prefix).append("XXXXXXXXXX").append(suffix);

		return mask.toString();
	}

	public void validate(String number, int type, String month, String date)
			throws CreditCardUtilException {

		try {
			Integer.parseInt(month);
			Integer.parseInt(date);
		} catch (NumberFormatException nfe) {
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invaliddate"),
					CreditCardUtilException.DATE);
		}

		if (number.equals("")) {
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invalidnumber"));
		}

		Matcher m = Pattern.compile("[^\\d\\s.-]").matcher(number);

		if (m.find()) {
			// setMessage("Credit card number can only contain numbers, spaces, \"-\", and \".\"");
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invalidnumber"));
		}

		Matcher matcher = Pattern.compile("[\\s.-]").matcher(number);

		number = matcher.replaceAll("");
		validateDate(Integer.parseInt(month), Integer.parseInt(date));
		validateNumber(number, type);
	}

	private void validateDate(int m, int y) throws CreditCardUtilException {
		java.util.Calendar cal = new java.util.GregorianCalendar();
		int monthNow = cal.get(java.util.Calendar.MONTH) + 1;
		int yearNow = cal.get(java.util.Calendar.YEAR);
		if (yearNow > y) {
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invaliddate"));
		}
		// OK, change implementation
		if (yearNow == y && monthNow > m) {
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invaliddate"));
		}

	}

	public void validateCvv(String cvvNumber, int type)
			throws CreditCardUtilException {
		
		
		if (StringUtils.isBlank(cvvNumber)) {
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invalidcvv"),
					CreditCardUtilException.CVV);
		}

		String expression = "[0-9]*";

		Matcher m = Pattern.compile(expression).matcher(cvvNumber);

		boolean mt = m.matches();

		if (!mt) {
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invalidcvv"),
					CreditCardUtilException.CVV);
		}

		switch (type) {
		case AMEX:
			if (cvvNumber.length() != 4) {
				throw new CreditCardUtilException(LabelUtil.getInstance()
						.getText("errors.creditcard.invalidcvv"),
						CreditCardUtilException.CVV);
			}
			break;
		default:
			if (cvvNumber.length() != 3) {
				throw new CreditCardUtilException(LabelUtil.getInstance()
						.getText("errors.creditcard.invalidcvv"),
						CreditCardUtilException.CVV);
			}
		}
	}

	// Check that cards start with proper digits for
	// selected card type and are also the right length.

	private void validateNumber(String number, int type)
			throws CreditCardUtilException {
		switch (type) {

		case MASTERCARD:
			if (number.length() != 16
					|| Integer.parseInt(number.substring(0, 2)) < 51
					|| Integer.parseInt(number.substring(0, 2)) > 55) {
				throw new CreditCardUtilException(LabelUtil.getInstance()
						.getText("errors.creditcard.invalidnumber"));
			}
			break;

		case VISA:
			if ((number.length() != 13 && number.length() != 16)
					|| Integer.parseInt(number.substring(0, 1)) != 4) {
				throw new CreditCardUtilException(LabelUtil.getInstance()
						.getText("errors.creditcard.invalidnumber"));
			}
			break;

		case AMEX:
			if (number.length() != 15
					|| (Integer.parseInt(number.substring(0, 2)) != 34 && Integer
							.parseInt(number.substring(0, 2)) != 37)) {
				throw new CreditCardUtilException(LabelUtil.getInstance()
						.getText("errors.creditcard.invalidnumber"));
			}
			break;

		case DISCOVER:
			if (number.length() != 16
					|| Integer.parseInt(number.substring(0, 5)) != 6011) {
				throw new CreditCardUtilException(LabelUtil.getInstance()
						.getText("errors.creditcard.invalidnumber"));
			}
			break;

		case DINERS:
			if (number.length() != 14
					|| ((Integer.parseInt(number.substring(0, 2)) != 36 && Integer
							.parseInt(number.substring(0, 2)) != 38)
							&& Integer.parseInt(number.substring(0, 3)) < 300 || Integer
							.parseInt(number.substring(0, 3)) > 305)) {
				throw new CreditCardUtilException(LabelUtil.getInstance()
						.getText("errors.creditcard.invalidnumber"));
			}
			break;
		}
		luhnValidate(number);
	}

	// The Luhn algorithm is basically a CRC type
	// system for checking the validity of an entry.
	// All major credit cards use numbers that will
	// pass the Luhn check. Also, all of them are based
	// on MOD 10.

	private void luhnValidate(String numberString)
			throws CreditCardUtilException {
		char[] charArray = numberString.toCharArray();
		int[] number = new int[charArray.length];
		int total = 0;

		for (int i = 0; i < charArray.length; i++) {
			number[i] = Character.getNumericValue(charArray[i]);
		}

		for (int i = number.length - 2; i > -1; i -= 2) {
			number[i] *= 2;

			if (number[i] > 9)
				number[i] -= 9;
		}

		for (int i = 0; i < number.length; i++)
			total += number[i];

		if (total % 10 != 0)
			throw new CreditCardUtilException(LabelUtil.getInstance().getText(
					"errors.creditcard.invalidnumber"));

	}

	public static List<String> getCreditCardStripImages() {

		Map ccs = RefCache.getSupportedCreditCards();
		List returnList = new ArrayList();
		if (ccs != null) {
			Iterator i = ccs.keySet().iterator();
			while (i.hasNext()) {
				int code = (Integer) i.next();
				CentralCreditCard ccc = (CentralCreditCard) ccs.get(code);
				StringBuffer cardImg = new StringBuffer();
				cardImg.append("icon-cc-");
				cardImg.append(ccc.getCentralCreditCardCode());
				cardImg.append(".gif");
				returnList.add(cardImg.toString());
			}

		}

		return returnList;

	}

}



```
