# CustomerUtil.java

## Review

## 1. Summary  

**Purpose** – `CustomerUtil` is a stateless helper that exposes several convenience methods for retrieving a customer’s address information (state and country) and for validating phone numbers and email addresses.  

**Key Components**  
- **Locale‑aware lookup** – Uses the `Locale` supplied to pick the proper language‑specific lookup tables (`RefCache`).  
- **Address helpers** – Four public static methods (`getCustomerBillingState`, `getCustomerBillingCountry`, `getCustomerShippingState`, `getCustomerShippingCountry`) that return the human‑readable name of a state or country.  
- **Validation helpers** – Three static methods (`ValidatePhoneNumber`, `validateEmail`, `isValid`) that test the format of a phone number or email against regular expressions.  

**Design Patterns / Libraries**  
- **Utility / Static Factory** – All methods are `static`; the class is effectively a utility façade.  
- **Caching** – Relies on a `RefCache` (not shown) to avoid expensive lookups.  
- **Apache Commons Lang** – Uses `StringUtils.isBlank` for null/empty checks.  
- **Regex** – Simple string‑matching via `String.matches`.

---

## 2. Detailed Description  

1. **Initialization**  
   - The class has no instance state; all fields are `static final` constants.  
   - No constructor is defined; the default public constructor would be generated, but it is unnecessary.

2. **Runtime Behavior**  
   - When an address‑lookup method is called, the routine first checks whether the customer already stores a string value for the requested field (`getCustomerBillingState`, etc.).  
   - If that string is non‑blank, it is returned immediately.  
   - Otherwise, the method obtains a language‑specific map of either zones (states) or countries from `RefCache`.  
   - The customer’s ID (e.g., `getCustomerBillingZoneId()`) is used as a key to fetch the corresponding `Zone` or `Country`.  
   - If found, the name (`getZoneName()` / `getCountryName()`) is returned; otherwise an empty string is returned.

3. **Validation**  
   - `ValidatePhoneNumber` checks a string against a hard‑coded pattern `(\\d-)?(\\d{3}-)?\\d{3}-\\d{4}`.  
   - `validateEmail` delegates to `isValid`, which applies a regular expression that only allows lowercase letters and digits, with optional dot, underscore or hyphen separators.  

4. **Assumptions & Constraints**  
   - `RefCache.getAllZonesmap` / `getAllcountriesmap` are expected to return `Map<Integer, Zone>` / `Map<Integer, Country>` keyed by the ID used by `Customer`.  
   - The customer IDs must be valid and present in the cache; otherwise, an empty string is silently returned.  
   - The regex patterns assume North‑American phone number formatting and very restrictive email syntax (no uppercase letters, no TLDs longer than one letter, etc.).

5. **Architecture / Design Choices**  
   - The class is intentionally stateless, making it thread‑safe.  
   - However, the use of raw `Map` types and unchecked casts (`Zone zone = (Zone) zones.get(...)`) defeats generics safety.  
   - Error handling is minimal: failures to find a zone or country simply yield an empty string rather than throwing an informative exception.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Output | Side‑Effects | Notes |
|--------|---------|--------|--------|--------------|-------|
| `getCustomerBillingState(Customer, Locale)` | Return the billing state name for a customer. | `Customer`, `Locale` | `String` (state name or `""`) | None | Uses cache; falls back to blank string if missing. |
| `getCustomerBillingCountry(Customer, Locale)` | Return the billing country name. | Same | `String` | None | Same as above. |
| `getCustomerShippingState(Customer, Locale)` | Return the shipping state name. | Same | `String` | None | Same as above. |
| `getCustomerShippingCountry(Customer, Locale)` | Return the shipping country name. | Same | `String` | None | Same as above. |
| `ValidatePhoneNumber(String)` | Check if phone number matches pattern. | `String` | `boolean` | None | Naming inconsistent with Java conventions (`validatePhoneNumber`). |
| `validateEmail(String)` | Check if email matches pattern. | `String` | `boolean` | None | Delegates to `isValid`. |
| `isValid(String, String)` | Generic regex validator. | `String` (regex), `String` (value) | `boolean` | None | Handles null value but not empty string. |

**Reusable / Utility** – `isValid` can be used for any regex‑based validation. All other methods are specific to the customer address domain.

---

## 4. Dependencies  

| Library / API | Usage | Standard / Third‑party |
|---------------|-------|------------------------|
| `java.util.Locale` | Language extraction | Standard |
| `java.util.Map` | Cache lookups | Standard |
| `org.apache.commons.lang.StringUtils` | `isBlank` check | Third‑party (Commons Lang) |
| `com.salesmanager.core.entity.customer.Customer` | Customer data holder | Project‑specific |
| `com.salesmanager.core.entity.reference.Country` / `Zone` | Lookup objects | Project‑specific |
| `com.salesmanager.core.service.cache.RefCache` | Retrieve pre‑cached lookup maps | Project‑specific |
| `com.salesmanager.core.util.LanguageUtil` | Convert language code to number | Project‑specific |

No external web services or platform‑specific APIs are invoked.

---

## 5. Additional Notes  

### Strengths  
- **Thread safety** – Stateless and purely functional.  
- **Cache usage** – Avoids repeated database or service calls.  
- **Clear separation** – Address lookup vs. validation logic.

### Weaknesses / Risks  
1. **Raw types & unchecked casts** – `Map` is used without generics; this leads to unchecked warnings and potential `ClassCastException` if the cache content changes.  
2. **Hard‑coded regex** –  
   - Phone pattern only matches specific formats (`XXX-XXX-XXXX` or `X-XXX-XXX-XXXX`), ignoring common variations such as spaces, parentheses, or international numbers.  
   - Email regex is overly restrictive (only lowercase, no `@` local‑part characters like `+`, `-`, or multi‑character TLDs).  
3. **Inconsistent naming** – `ValidatePhoneNumber` violates Java camelCase conventions.  
4. **Silent failure** – Missing zone/country returns `""`; callers have no indication of why the lookup failed.  
5. **No null checks on `customer`** – Passing `null` would cause `NullPointerException`.  
6. **Magic strings** – Error messages in `ValidatePhoneNumber` are defined but never used; the method returns only a boolean.  
7. **Locale handling** – Only the language component is used; region or script information is ignored.

### Edge Cases Not Handled  
- Customer fields containing only whitespace.  
- Phone numbers with country codes or extensions.  
- Email addresses with uppercase letters or Unicode characters.  
- Missing or malformed `Locale` values.  

### Potential Enhancements  
- **Generics** – Replace raw `Map` with `Map<Integer, Zone>` / `Map<Integer, Country>`.  
- **Parameter validation** – Add checks for `null` `customer` and `locale`.  
- **Better error reporting** – Return an `Optional<String>` or throw a custom exception if lookup fails.  
- **Improved regex** – Adopt more permissive, RFC‑compliant patterns or delegate to a dedicated validation library (e.g., Apache Commons Validator).  
- **Consistent naming** – Rename `ValidatePhoneNumber` to `validatePhoneNumber`.  
- **Internationalization** – Extend phone and email validators to support international formats.  
- **Unit tests** – Provide comprehensive tests covering all lookup scenarios and validation edge cases.  

By addressing these points, the utility would become safer, more maintainable, and better aligned with modern Java best practices.

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

import java.util.Locale;
import java.util.Map;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.cache.RefCache;

/**
 * Returns the appropriate customer state and country based on the locale
 * 
 * @author Administrator
 * 
 */
public class CustomerUtil {

	private final static String numPattern = "(\\d-)?(\\d{3}-)?\\d{3}-\\d{4}";
	private static final String EMAIL_REGEXPR = "[a-z0-9]+([_\\.-][a-z0-9]+)*@([a-z0-9]+)+[_\\.-]+[a-z.]*[a-z]$";

	public static String getCustomerBillingState(Customer customer,
			Locale locale) {

		if (!StringUtils.isBlank(customer.getCustomerBillingState())) {
			return customer.getCustomerBillingState();
		}

		Map zones = RefCache.getAllZonesmap((LanguageUtil
				.getLanguageNumberCode(locale.getLanguage())));
		Zone zone = (Zone) zones.get(customer.getCustomerBillingZoneId());
		if (zone != null) {
			return zone.getZoneName();
		}
		return "";
	}

	public static String getCustomerBillingCountry(Customer customer,
			Locale locale) {

		Map countries = RefCache.getAllcountriesmap(((LanguageUtil
				.getLanguageNumberCode(locale.getLanguage()))));
		Country country = (Country) countries.get(customer
				.getCustomerBillingCountryId());
		if (country != null) {
			return country.getCountryName();
		}
		return "";
	}

	public static String getCustomerShippingState(Customer customer,
			Locale locale) {

		if (!StringUtils.isBlank(customer.getCustomerState())) {
			return customer.getCustomerState();
		}

		Map zones = RefCache.getAllZonesmap((LanguageUtil
				.getLanguageNumberCode(locale.getLanguage())));
		Zone zone = (Zone) zones.get(customer.getCustomerZoneId());
		if (zone != null) {
			return zone.getZoneName();
		}
		return "";
	}

	public static String getCustomerShippingCountry(Customer customer,
			Locale locale) {

		Map countries = RefCache.getAllcountriesmap(((LanguageUtil
				.getLanguageNumberCode(locale.getLanguage()))));
		Country country = (Country) countries.get(customer
				.getCustomerCountryId());
		if (country != null) {
			return country.getCountryName();
		}
		return "";
	}

	public static boolean ValidatePhoneNumber(String phNumber) {
		String msgResult = "";
		boolean valResult = false;

		valResult = phNumber.matches(numPattern);

		if (valResult) {
			msgResult = "The phone number validates.";
		} else {
			msgResult = "The phone number does not validate";
		}
		return valResult;
	}

	public static boolean validateEmail(String email) {
		return isValid(EMAIL_REGEXPR, email);
	}

	public static boolean isValid(String regExp, String value) {
		if (value == null) {
			return false;
		}
		return value.matches(regExp);
	}

}



```
