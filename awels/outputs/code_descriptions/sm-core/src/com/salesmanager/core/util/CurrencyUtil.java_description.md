# CurrencyUtil.java

## Review

## 1. Summary  

**Purpose**  
`CurrencyUtil` is a static helper class that centralises all currency‑ and measurement‑related logic used across the SalesManager application.  Its responsibilities include:

* Loading a mapping of currency codes to `CurrencyModule` beans (specific or generic) from Spring/Cache.
* Validating and parsing string representations of amounts and measures for a given currency.
* Converting between different weight (lb/​kg) and length (in/​cm) units.
* Converting amounts from one currency to another using exchange rates stored in the `RefCache`.
* Formatting amounts, measures, and currency symbols for display (including HTML snippets for editable price fields).
* Providing a fallback default currency if none is supplied.

**Key components**

| Component | Role |
|-----------|------|
| `currencyMap` | Cache of `CurrencyModule` instances keyed by ISO currency code. |
| `GenericCurrencyModule` | Default fallback when a specific Spring bean is not available. |
| `RefCache.getCurrenciesListWithCodes()` | Retrieves `Currency` entities (with exchange‑rate values) used for conversions. |
| `CurrencyModule` interface | Defines currency‑specific parsing, formatting, and conversion logic. |
| `PropertiesUtil` / `Configuration` | Loads application‑wide default currency setting. |

**Design patterns / frameworks**

* **Factory/Strategy** – The `CurrencyModule` beans act as strategy objects that encapsulate currency‑specific behaviour.
* **Singleton / Static Utility** – The class itself is a singleton by virtue of only static members; it uses static initialisation to populate its cache.
* **Spring / Dependency Injection** – Beans are obtained via `SpringUtil.getBean(code)`.
* **Cache** – Uses a reference cache (`RefCache`) to avoid repeated DB lookups.

---

## 2. Detailed Description  

### Initialisation  
At class load time, the static block:

1. Calls `RefCache.getCurrenciesListWithCodes()` to get a `Map<String, Currency>` of all supported currencies.  
2. Iterates over each currency code and attempts to obtain a Spring bean named after the code (`CurrencyModule`).  
3. If a bean is present, it is configured with the corresponding `Currency` object and stored in `currencyMap`.  
4. If the bean is missing or an exception occurs, a `GenericCurrencyModule` is created and used instead.  

The block is wrapped in a broad `try‑catch`; any failure is logged as an error but does not prevent the class from loading.

### Core execution flow  

| Phase | Description |
|-------|-------------|
| **Validation** | `validateMeasure()` / `validateCurrency()` retrieve the appropriate `CurrencyModule`, parse the string amount, round the result and return a `BigDecimal`. |
| **Unit conversion** | `getWeight()` and `getMeasure()` convert values between store‑configured units (lb/kg or in/cm) using hard‑coded constants and `BigDecimal` rounding. |
| **Currency conversion** | `convertToCurrency()` fetches the source and target `Currency` objects from `RefCache`, normalises the amount by dividing by the source rate and multiplying by the target rate, then returns the converted value as a `BigDecimal`. |
| **Formatting / display** | A set of `display*` methods fetch the relevant `CurrencyModule` and delegate formatting (measure, amount with/without symbol, CSS‑friendly output). `displayEditablePriceWithCurrency()` builds an HTML input element pre‑populated with the formatted amount. |
| **Default currency** | `getDefaultCurrency()` reads the `core.system.defaultcurrency` property; if missing, defaults to USD. |

### Assumptions & constraints  

* The `RefCache` and Spring context are already initialised before `CurrencyUtil` is first referenced.  
* All currency codes used as bean names in Spring follow the exact naming convention.  
* Conversion constants (2.2 for lb/kg, 0.39 for cm/in) are hard‑coded and not configurable.  
* The class is intended to be thread‑safe only by virtue of using immutable objects (`BigDecimal`) and a read‑only cache after initialisation.  
* Error handling is performed by throwing `ValidationException` for known problems and logging generic errors for unexpected ones.

### Architecture & design choices  

* **Static Utility** – The design favours simplicity over full object orientation; all behaviour is static.  
* **Strategy pattern** – `CurrencyModule` implementations allow per‑currency customisation without changing the util logic.  
* **Fallback strategy** – `GenericCurrencyModule` guarantees that missing beans do not crash the application.  
* **Lazy conversion** – Currency conversion is performed on demand; exchange rates are pulled from cache only once per call, which may be sub‑optimal for high‑traffic scenarios.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects / Exceptions |
|--------|---------|------------|---------|---------------------------|
| `validateMeasure(String measure, String currencycode)` | Parses a string amount and returns a rounded `BigDecimal` using the currency module. | `measure` – raw amount; `currencycode` – ISO code | `BigDecimal` | Throws `ValidationException` if module missing or parsing fails. |
| `validateCurrency(String amount, String currencycode)` | Same as above but for a normal amount string. | `amount`, `currencycode` | `BigDecimal` | Throws `ValidationException`. |
| `getWeight(double weight, MerchantStore store, String base)` | Converts weight between pounds and kilograms based on store settings. | `weight`, `store`, `base` | `double` | Rounds to 2 decimal places. |
| `convertToCurrency(BigDecimal amount, String originCurrency, String toCurrency)` | Converts `amount` from `originCurrency` to `toCurrency` using rates from `RefCache`. | `amount`, `originCurrency`, `toCurrency` | `BigDecimal` | Logs errors; returns original amount on failure. |
| `getMeasure(double measure, MerchantStore store, String base)` | Converts length between inches and centimeters. | `measure`, `store`, `base` | `double` | Rounds to 2 decimal places. |
| `displayMeasure(BigDecimal measure, String currencycode)` | Returns a human‑readable string representation of a measure using the currency module. | `measure`, `currencycode` | `String` | Logs error on missing module. |
| `displayFormatedAmount(BigDecimal amount, String currencycode)` (private) | Formats amount without currency symbol. | `amount`, `currencycode` | `String` | Logs error on missing module. |
| `displayFormatedAmountWithCurrency(BigDecimal amount, String currencycode)` | Formats amount with currency symbol. | `amount`, `currencycode` | `String` | Logs error on missing module. |
| `displayFormatedCssAmountWithCurrency(BigDecimal amount, String currencycode)` | Formats amount with CSS‑friendly wrapper (`product‑value`). | `amount`, `currencycode` | `String` | Logs error on missing module. |
| `displayFormatedAmountNoCurrency(BigDecimal amount, String currencycode)` | Convenience wrapper that forwards to `displayFormatedAmount`. | `amount`, `currencycode` | `String` | — |
| `getAmount(BigDecimal amount, String currencycode)` | Alias for `displayFormatedAmount`. | `amount`, `currencycode` | `String` | — |
| `getAmount(String amount, String currencyCode)` | Parses a string amount to `BigDecimal` via currency module. | `amount`, `currencyCode` | `BigDecimal` | Throws `ValidationException` if module missing. |
| `displayEditablePriceWithCurrency(String textname, int textsize, boolean displaycurrency, BigDecimal amount, String currencycode, String appender)` | Builds an HTML input element pre‑filled with the formatted amount. | `textname`, `textsize`, `displaycurrency`, `amount`, `currencycode`, `appender` | `String` | Logs error on missing module. |
| `getDefaultCurrency()` | Retrieves the default ISO currency from configuration (or USD). | None | `String` | — |

**Reusable / utility methods** – The various `display*` methods encapsulate formatting logic; any future changes to how amounts/measures are shown should be confined to `CurrencyModule` implementations or these helpers.

---

## 4. Dependencies  

| External Library / Framework | Usage | Type |
|------------------------------|-------|------|
| **Apache Commons Configuration** (`org.apache.commons.configuration.Configuration`) | Reads system properties for the default currency. | 3rd‑party |
| **Log4j** (`org.apache.log4j.Logger`) | Logging throughout the class. | 3rd‑party |
| **XWork2 Validator** (`com.opensymphony.xwork2.validator.ValidationException`) | Exception type for validation failures. | 3rd‑party |
| **SalesManager Core** (`com.salesmanager.core.*`) | Core domain entities (`MerchantStore`, `Currency`), modules, constants, utilities (`SpringUtil`, `PropertiesUtil`, `RefCache`). | Project‑specific |
| **Spring** (via `SpringUtil.getBean`) | Retrieves currency modules by bean name. | 3rd‑party |

No external platform‑specific APIs are used; the class relies on standard Java SE (particularly `java.math.BigDecimal`).

---

## 5. Additional Notes  

### Strengths  

* **Centralised currency logic** – Keeps all conversions, parsing, and formatting in one place.  
* **Extensibility** – New currencies can be supported by adding a bean that implements `CurrencyModule`.  
* **Graceful degradation** – Missing beans fall back to a generic implementation, preventing crashes.  
* **Thread‑safe after initialisation** – The cache is immutable once built.

### Issues / Risks  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Static block may swallow exceptions** | Silent failure to load currency modules; class will still be usable but with incomplete mapping. | Re‑throw as a runtime exception or add a flag to indicate failure. |
| **Hard‑coded conversion constants** | Inaccurate conversions (e.g., 1 lb = 0.453592 kg, not 2.2). | Replace with constants from `java.util.concurrent.TimeUnit` or compute via `Currency` rates; make configurable. |
| **Inconsistent rounding** | `convertToCurrency()` uses `ROUND_UP` whereas others use `ROUND_HALF_UP`. | Standardise rounding mode. |
| **Double → BigDecimal conversions** | Loss of precision during currency conversion and unit conversion. | Operate directly on `BigDecimal` using `divide()` and `multiply()` with appropriate `MathContext`. |
| **`log.equals(e)` typo** | No error is logged in `convertToCurrency()`. | Change to `log.error(e);`. |
| **Generic raw types** | Compiler warnings and potential `ClassCastException`. | Use generics (`Map<String, CurrencyModule>` etc.). |
| **Public mutable static `currencyMap`** | Potential concurrency issues if modified elsewhere. | Make the map unmodifiable (`Collections.unmodifiableMap`) after construction. |
| **`displayEditablePriceWithCurrency` building string** | Extra `StringBuffer` allocation; potential bug with `appender`. | Simplify with `StringBuilder` and proper escaping of HTML attributes. |
| **Method name conflict** (`getAmount` overloaded) | Confusing API – one returns `String`, the other `BigDecimal`. | Rename or consolidate into a single method with clear semantics. |
| **No unit tests** | Hard to guarantee correctness. | Add JUnit tests for conversion, formatting, and edge cases (nulls, unsupported currencies). |
| **No validation for null `amount` in `convertToCurrency`** | Potential `NullPointerException`. | Add null check. |

### Future Enhancements  

1. **Configuration‑driven conversion factors** – Allow admins to set weight/length conversion constants via properties.  
2. **Caching of currency rates** – Pull rates once and store with a TTL; avoid querying `RefCache` on every conversion.  
3. **Locale‑aware formatting** – Use `java.text.NumberFormat` with `Locale` tied to the currency.  
4. **Expose a non‑static API** – Inject a `CurrencyService` into Spring beans rather than relying on static utilities.  
5. **Unit tests & code coverage** – Ensure high confidence for future refactors.

---  

**Conclusion** – `CurrencyUtil` is a practical, albeit somewhat dated, utility class that consolidates currency and measurement handling.  Addressing the highlighted issues—particularly precision, configurability, and type safety—would considerably improve robustness and maintainability.

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

import java.math.BigDecimal;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.validator.ValidationException;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Currency;
import com.salesmanager.core.module.impl.application.currencies.GenericCurrencyModule;
import com.salesmanager.core.module.model.application.CurrencyModule;
import com.salesmanager.core.service.cache.RefCache;

public class CurrencyUtil {

	private final static char EURO = '\u20AC';
	private final static char POUND = '\u00A3';
	private final static char DOLLAR = '\u0024';
	private static Logger log = Logger.getLogger(CurrencyUtil.class);

	private static Map currencyMap = new HashMap();

	static {

		try {

			Map currencies = RefCache.getCurrenciesListWithCodes();
			if (currencies != null) {
				Iterator i = currencies.keySet().iterator();
				while (i.hasNext()) {
					String code = (String) i.next();

					try {

						CurrencyModule module = (CurrencyModule) SpringUtil
								.getBean(code);
						Currency cur = (Currency) currencies.get(code);
						if (module != null) {
							module.setCurrency(cur);
							currencyMap.put(code, module);
						} else {
							log
									.warn("Currency "
											+ code
											+ " is not supported by a Spring module, using GenericCurrency");

							GenericCurrencyModule currency = new GenericCurrencyModule();
							currency.setCurrency(cur);
							currencyMap.put(code, currency);
						}

					} catch (Exception e) {
						log
								.warn("Currency "
										+ code
										+ " is not supported by a Spring module, using GenericCurrency");
						Currency cur = (Currency) currencies.get(code);
						GenericCurrencyModule currency = new GenericCurrencyModule();
						currency.setCurrency(cur);
						currencyMap.put(code, currency);
					}

				}
			}

		} catch (Exception e) {
			log.error(e);
		}

	}

	public static BigDecimal validateMeasure(String measure, String currencycode)
			throws ValidationException {

		try {

			if (currencycode == null) {
				currencycode = getDefaultCurrency();
			}

			log.debug("Trying to validate " + measure + " for currency "
					+ currencycode);

			CurrencyModule module = (CurrencyModule) currencyMap
					.get(currencycode);

			if (module == null) {
				throw new ValidationException(
						"There is no CurrencyModule defined for currency "
								+ currencycode
								+ " in module/impl/application/currrencies");
			}

			BigDecimal returnMeasure = module.getAmount(measure);
			returnMeasure.setScale(1, BigDecimal.ROUND_HALF_UP);
			return returnMeasure;

		} catch (Exception e) {
			if (e instanceof ValidationException)
				throw (ValidationException) e;
			throw new ValidationException(e.getMessage());
		}

	}

	public static BigDecimal validateCurrency(String amount, String currencycode)
			throws ValidationException {

		try {

			if (currencycode == null) {
				currencycode = getDefaultCurrency();
			}

			log.debug("Trying to validate " + amount + " for currency "
					+ currencycode);

			CurrencyModule module = (CurrencyModule) currencyMap
					.get(currencycode);

			if (module == null) {
				throw new ValidationException(
						"There is no CurrencyModule defined for currency "
								+ currencycode
								+ " in module/impl/application/currrencies");
			}

			return module.getAmount(amount);

		} catch (Exception e) {
			if (e instanceof ValidationException)
				throw (ValidationException) e;
			throw new ValidationException(e.getMessage());
		}
	}

	/**
	 * Get the measure according to the appropriate measure base. If the measure
	 * configured in store is LB and it needs KG then the appropriate
	 * calculation is done
	 * 
	 * @param weight
	 * @param store
	 * @param base
	 * @return
	 */
	public static double getWeight(double weight, MerchantStore store,
			String base) {

		double weightConstant = 2.2;
		if (base.equals(Constants.LB_WEIGHT_UNIT)) {
			if (store.getWeightunitcode().equals(Constants.LB_WEIGHT_UNIT)) {
				return new BigDecimal(String.valueOf(weight)).setScale(2,
						BigDecimal.ROUND_HALF_UP).doubleValue();
			} else {// pound = kilogram
				double answer = weight * weightConstant;
				BigDecimal w = new BigDecimal(answer);
				return w.setScale(2, BigDecimal.ROUND_HALF_UP).doubleValue();
			}
		} else {// need KG
			if (store.getWeightunitcode().equals(Constants.KG_WEIGHT_UNIT)) {
				return new BigDecimal(String.valueOf(weight)).setScale(2,
						BigDecimal.ROUND_HALF_UP).doubleValue();
			} else {

				double answer = weight / weightConstant;
				BigDecimal w = new BigDecimal(answer);
				return w.setScale(2, BigDecimal.ROUND_HALF_UP).doubleValue();

			}
		}
	}

	/**
	 * Converts a BigDecimal amount from a given currency to another currency
	 * 
	 * @param amount
	 * @param originCurrency
	 * @param toCurrency
	 * @return
	 */
	public static BigDecimal convertToCurrency(BigDecimal amount,
			String originCurrency, String toCurrency) {

		try {

			// get originCurrency
			Map currencies = RefCache.getCurrenciesListWithCodes();
			double returnAmount = amount.doubleValue();
			com.salesmanager.core.entity.reference.Currency origin = (com.salesmanager.core.entity.reference.Currency) currencies
					.get(originCurrency);
			com.salesmanager.core.entity.reference.Currency convert = (com.salesmanager.core.entity.reference.Currency) currencies
					.get(toCurrency);
			if (origin == null) {
				log.error("Origin currency " + originCurrency + " not found");
				return amount;
			}

			if (convert == null) {
				log.error("Convert currency " + toCurrency + " not found");
				return amount;
			}

			returnAmount = returnAmount / origin.getValue().doubleValue();

			returnAmount = returnAmount * convert.getValue().doubleValue();

			return new BigDecimal(returnAmount)
					.setScale(2, BigDecimal.ROUND_UP);

		} catch (Exception e) {
			log.equals(e);
			return amount;
		}
	}

	/**
	 * Get the measure according to the appropriate measure base. If the measure
	 * configured in store is IN and it needs CM or vise versa then the
	 * appropriate calculation is done
	 * 
	 * @param weight
	 * @param store
	 * @param base
	 * @return
	 */
	public static double getMeasure(double measure, MerchantStore store,
			String base) {

		if (base.equals(Constants.INCH_SIZE_UNIT)) {
			if (store.getSeizeunitcode().equals(Constants.INCH_SIZE_UNIT)) {
				return new BigDecimal(String.valueOf(measure)).setScale(2,
						BigDecimal.ROUND_HALF_UP).doubleValue();
			} else {// centimeter (inch to centimeter)
				double measureConstant = 2.54;

				double answer = measure * measureConstant;
				BigDecimal w = new BigDecimal(answer);
				return w.setScale(2, BigDecimal.ROUND_HALF_UP).doubleValue();

			}
		} else {// need CM
			if (store.getSeizeunitcode().equals(Constants.CM_SIZE_UNIT)) {
				return new BigDecimal(String.valueOf(measure)).setScale(2)
						.doubleValue();
			} else {// in (centimeter to inch)
				double measureConstant = 0.39;

				double answer = measure * measureConstant;
				BigDecimal w = new BigDecimal(answer);
				return w.setScale(2, BigDecimal.ROUND_HALF_UP).doubleValue();

			}
		}

	}

	/**
	 * For displaying weight, length, height and width
	 * 
	 * @param measure
	 * @param currncycode
	 * @return
	 */

	public static String displayMeasure(BigDecimal measure, String currencycode) {

		try {

			if (measure == null) {
				return "";
			}

			if (currencycode == null) {
				currencycode = getDefaultCurrency();
			}

			CurrencyModule module = (CurrencyModule) currencyMap
					.get(currencycode);

			if (module == null) {
				log.error("There is no CurrencyModule defined for currency "
						+ currencycode
						+ " in module/impl/application/currrencies");
				return measure.toString();
			}

			return module.getMeasure(measure, currencycode);

		} catch (Exception e) {
			log.error("Cannot format measure " + measure.toString()
					+ " for currency " + currencycode);
			return measure.toString();
		}
	}

	private static String displayFormatedAmount(BigDecimal amount,
			String currencycode) {

		try {

			if (amount == null) {
				return "";
			}

			if (currencycode == null) {
				currencycode = getDefaultCurrency();
			}

			CurrencyModule module = (CurrencyModule) currencyMap
					.get(currencycode);

			if (module == null) {
				log.error("There is no CurrencyModule defined for currency "
						+ currencycode
						+ " in module/impl/application/currrencies");
				return amount.toString();
			}

			return module.getFormatedAmount(amount);

		} catch (Exception e) {
			log.error("Cannot format amount " + amount.toString()
					+ " for currency " + currencycode);
			return amount.toString();
		}

	}

	public static String displayFormatedAmountWithCurrency(BigDecimal amount,
			String currencycode) {

		try {

			if (currencycode == null) {
				currencycode = getDefaultCurrency();
			}

			CurrencyModule module = (CurrencyModule) currencyMap
					.get(currencycode);

			if (module == null) {
				log.error("There is no CurrencyModule defined for currency "
						+ currencycode
						+ " in module/impl/application/currrencies");
				return amount.toString();
			}

			return module.getFormatedAmountWithCurrency(amount);

		} catch (Exception e) {
			log.error("Cannot format amount " + amount.toString()
					+ " for currency " + currencycode);
			return amount.toString();
		}
	}

	public static String displayFormatedCssAmountWithCurrency(
			BigDecimal amount, String currencycode) {

		try {

			if (currencycode == null) {
				currencycode = getDefaultCurrency();
			}

			CurrencyModule module = (CurrencyModule) currencyMap
					.get(currencycode);

			if (module == null) {
				log.error("There is no CurrencyModule defined for currency "
						+ currencycode
						+ " in module/impl/application/currrencies");
				return amount.toString();
			}

			return module
					.getFormatedAmountWithCurrency(amount, "product-value");

		} catch (Exception e) {
			log.error("Cannot format amount " + amount.toString()
					+ " for currency " + currencycode);
			return amount.toString();
		}
	}

	public static String displayFormatedAmountNoCurrency(BigDecimal amount,
			String currencycode) {

		if (currencycode == null) {
			currencycode = getDefaultCurrency();
		}

		return displayFormatedAmount(amount, currencycode);

	}

	public static String getAmount(BigDecimal amount, String currencycode) {

		if (currencycode == null) {
			currencycode = getDefaultCurrency();
		}

		return displayFormatedAmount(amount, currencycode);

	}

	public static BigDecimal getAmount(String amount, String currencyCode)
			throws ValidationException {

		CurrencyModule module = (CurrencyModule) currencyMap.get(currencyCode);

		if (module == null) {
			throw new ValidationException(
					"There is no CurrencyModule defined for currency "
							+ currencyCode
							+ " in module/impl/application/currrencies");
		}

		try {

			return module.getAmount(amount);

		} catch (Exception e) {
			log.error("Cannot format amount " + amount + " for currency "
					+ currencyCode);
			return null;
		}

	}

	public static String displayEditablePriceWithCurrency(String textname,
			int textsize, boolean displaycurrency, BigDecimal amount,
			String currencycode, String appender) {

		if (currencycode == null) {
			currencycode = getDefaultCurrency();
		}

		StringBuffer formatedfieldbuffer = new StringBuffer();

		CurrencyModule module = (CurrencyModule) currencyMap.get(currencycode);

		if (module == null) {
			log.error("There is no CurrencyModule defined for currency "
					+ currencycode + " in module/impl/application/currrencies");
			return amount.toString();
		}

		String returnamount = "";
		try {
			returnamount = module.getFormatedAmount(amount);
		} catch (Exception e) {
			log.error("Cannot format amount " + amount.toString()
					+ " for currency " + currencycode);
			returnamount = amount.toString();
		}

		String display = module.getCurrencySymbol();

		return new StringBuffer().append(display).append(" ").append(
				"<input type=\"text\" name=\"").append(textname).append("\"")
				.append(" id=\"").append(textname).append("\"").append(
						" value=\"").append(returnamount).append("\"").append(
						" size=\"").append(textsize).append("\"").append(
						appender != null ? " " + appender : "").append(">")
				.toString();
	}

	public static String getDefaultCurrency() {

		Configuration conf = PropertiesUtil.getConfiguration();
		String def = conf.getString("core.system.defaultcurrency");
		if (def == null) {
			def = Constants.CURRENCY_CODE_USD;
		}

		return def;
	}

}



```
