# CreditCard.java

## Review

## 1. Summary  

The **`CreditCard`** class is a lightweight data holder that represents a customer’s payment card.  
Its responsibilities are:

| Responsibility | What it does |
|-----------------|--------------|
| **Storage** | Keeps the raw card number, expiration month/year, CVV, card owner, a numeric card‑type code, and an optional `Locale`. |
| **Utility** | Exposes a masked (encrypted) card number and a human‑readable card name that is localized via `LabelUtil`. |
| **Serialization** | Implements `Serializable` so that instances can be stored or transmitted. |

Design choices:

* The class is deliberately simple, following a **POJO (Plain Old Java Object)** pattern – getters/setters only.  
* Card‑type constants are supplied by `CreditCardUtil` and translated into internationalised strings using `LabelUtil`.  
* No validation or persistence logic is included; the class is purely a data container.

---

## 2. Detailed Description  

### Core Components  
1. **Fields** – All fields are `private`.  
   * `String cardNumber` – the raw card number.  
   * `int creditCardCode` – numeric type (VISA, AMEX, …). Defaults to `-1`.  
   * `String expirationYear/Month` – stored as strings (e.g., “2025”, “07”).  
   * `String cvv` – card‑verification value.  
   * `String cardOwner` – holder’s name.  
   * `Locale locale` – optional localisation context.

2. **Getter/Setter Methods** – Standard JavaBean accessors.  
3. **Derived Properties**  
   * `getEncryptedCreditCardNumber()` – returns a masked version of `cardNumber` using `CreditCardUtil.maskCardNumber`.  
   * `getCreditCardName()` – maps `creditCardCode` to a localized string via `LabelUtil`.

### Flow of Execution  

| Stage | Description |
|-------|-------------|
| **Instantiation** | A new `CreditCard` object is created; fields are set via setters. |
| **Runtime** | When a client calls `getEncryptedCreditCardNumber()` or `getCreditCardName()`, the object uses the stored state to produce the derived value. |
| **Cleanup** | No special cleanup; the class is purely data‑driven. |

### Assumptions & Constraints  

* `locale` may be `null`; callers must provide it before invoking localisation.  
* `creditCardCode` is an `int` that must match one of the constants defined in `CreditCardUtil`.  
* The card number is treated as a plain string; no validation or formatting is enforced.  
* The class relies on two external utilities:
  * `com.salesmanager.core.util.CreditCardUtil` – for masking and constants.
  * `com.salesmanager.core.util.LabelUtil` – for i18n lookup.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects / Notes |
|--------|---------|------------|---------|---------------------|
| `setLocale(Locale)` | Store the locale used for localisation. | `Locale locale` | void | |
| `getCardNumber()` | Return the raw card number. | – | `String` | |
| `setCardNumber(String)` | Set the raw card number. | `String cardNumber` | void | |
| `getCreditCardCode()` | Return the numeric card type code. | – | `int` | |
| `setCreditCardCode(int)` | Set the card type code. | `int creditCardCode` | void | |
| `getCvv()` | Return CVV. | – | `String` | |
| `setCvv(String)` | Set CVV. | `String cvv` | void | |
| `getExpirationMonth()` | Return month part of expiration. | – | `String` | |
| `setExpirationMonth(String)` | Set month. | `String expirationMonth` | void | |
| `getExpirationYear()` | Return year part of expiration. | – | `String` | |
| `setExpirationYear(String)` | Set year. | `String expirationYear` | void | |
| `getEncryptedCreditCardNumber()` | Mask the card number. | – | `String` | Instantiates `CreditCardUtil`; swallows any exception and returns an empty string. |
| `getCreditCardName()` | Translate the card code into a locale‑specific label. | – | `String` | Uses a `switch` over `creditCardCode`. Default case contains a typo: `"label.patment.creditcard.default"`. |
| `getCardOwner()` | Return the card holder name. | – | `String` | |
| `setCardOwner(String)` | Set the card holder name. | `String cardOwner` | void | |

**Reusable/Utility Methods** – None beyond the getters/setters. All localisation logic is bundled in `getCreditCardName()`.

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `java.io.Serializable` | Standard | Enables object serialization. |
| `java.util.Locale` | Standard | Provides localisation context. |
| `com.salesmanager.core.util.CreditCardUtil` | Third‑party (internal) | Holds card type constants & `maskCardNumber()` helper. |
| `com.salesmanager.core.util.LabelUtil` | Third‑party (internal) | Fetches i18n strings for card names. |

No external frameworks or platform‑specific APIs are required.

---

## 5. Additional Notes & Recommendations  

### 5.1  Error Handling  
`getEncryptedCreditCardNumber()` catches a generic `Exception` and discards it.  
* **Problem** – silent failures make debugging hard.  
* **Fix** – log the exception (or re‑throw a runtime exception).  

### 5.2  Missing `serialVersionUID`  
The class implements `Serializable` but does not declare a `serialVersionUID`.  
* **Impact** – default UID generation may cause `InvalidClassException` if the class changes.  
* **Fix** – add `private static final long serialVersionUID = 1L;`.

### 5.3  Null / Empty Checks  
None of the getters or setters guard against `null` or malformed values.  
* **Potential** – callers could store an empty card number, CVV, or an invalid month/year.  
* **Enhancement** – validate input values (e.g., Luhn algorithm for card number, 3/4 digit CVV, numeric month/year).

### 5.4  `Locale` Handling  
`locale` is optional but required for `getCreditCardName()`.  
* **Issue** – if `locale` is `null`, `LabelUtil.getInstance().getText()` may throw an exception.  
* **Solution** – provide a default locale or check for `null` before use.

### 5.5  Typo in Default Label Key  
`"label.patment.creditcard.default"` – “patment” is misspelled.  
* **Effect** – the fallback text will be missing, resulting in `null` or the key itself being displayed.  
* **Correction** – change to `"label.payment.creditcard.default"`.

### 5.6  Design Alternatives  
1. **Enum for Card Type** – Replace the `int creditCardCode` with an enum (`CreditCardType`). This improves type safety and readability.  
2. **Immutable DTO** – Use a builder or constructor‑based approach to make the object immutable, enhancing thread safety.  
3. **Java Time API** – Replace separate `expirationYear`/`expirationMonth` strings with a `YearMonth` or `LocalDate` to enforce validity.  

### 5.7  Future Enhancements  
* **Encryption** – Instead of masking, store an encrypted representation using a secure key‑management system.  
* **Validation Layer** – Integrate a validator that checks card number format, expiration date (must be future), CVV length, and card type consistency.  
* **Audit Trail** – Add timestamps or user IDs for when the card was added/modified.  
* **Internationalisation** – Expand `LabelUtil` to support dynamic language switching at runtime.  

---

### Bottom Line  

The class is a straightforward POJO for holding credit‑card data and providing a couple of derived values. It works fine for simple scenarios but would benefit from better error handling, validation, and modern Java idioms (enums, immutable design). Addressing the typo and adding a `serialVersionUID` would make it production‑ready.

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
package com.salesmanager.core.entity.payment;

import java.io.Serializable;
import java.util.Locale;

import com.salesmanager.core.util.CreditCardUtil;
import com.salesmanager.core.util.LabelUtil;

public class CreditCard implements Serializable {

	private String cardNumber;
	private int creditCardCode = -1;
	private String expirationYear;
	private String expirationMonth;
	private String cvv;
	private String cardOwner;
	private Locale locale;

	public void setLocale(Locale locale) {
		this.locale = locale;
	}

	public String getCardNumber() {
		return cardNumber;
	}

	public void setCardNumber(String cardNumber) {
		this.cardNumber = cardNumber;
	}

	public int getCreditCardCode() {
		return creditCardCode;
	}

	public void setCreditCardCode(int creditCardCode) {
		this.creditCardCode = creditCardCode;
	}

	public String getCvv() {
		return cvv;
	}

	public void setCvv(String cvv) {
		this.cvv = cvv;
	}

	public String getExpirationMonth() {
		return expirationMonth;
	}

	public void setExpirationMonth(String expirationMonth) {
		this.expirationMonth = expirationMonth;
	}

	public String getExpirationYear() {
		return expirationYear;
	}

	public void setExpirationYear(String expirationYear) {
		this.expirationYear = expirationYear;
	}

	public String getEncryptedCreditCardNumber() {
		CreditCardUtil util = new CreditCardUtil();
		try {
			String enc = util.maskCardNumber(this.getCardNumber());
			return enc;
		} catch (Exception e) {
			// TODO: handle exception
		}
		return "";
	}

	public String getCreditCardName() {
		switch (this.getCreditCardCode()) {
		case CreditCardUtil.VISA:
			return LabelUtil.getInstance().getText(locale,
					"label.payment.creditcard.visa");
		case CreditCardUtil.AMEX:
			return LabelUtil.getInstance().getText(locale,
					"label.payment.creditcard.amex");
		case CreditCardUtil.MASTERCARD:
			return LabelUtil.getInstance().getText(locale,
					"label.payment.creditcard.mastercard");
		case CreditCardUtil.DINERS:
			return LabelUtil.getInstance().getText(locale,
					"label.payment.creditcard.diners");
		case CreditCardUtil.DISCOVER:
			return LabelUtil.getInstance().getText(locale,
					"label.payment.creditcard.discovery");
		default:
			return LabelUtil.getInstance().getText(locale,
					"label.patment.creditcard.default");
		}
	}

	public String getCardOwner() {
		return cardOwner;
	}

	public void setCardOwner(String cardOwner) {
		this.cardOwner = cardOwner;
	}

}



```
