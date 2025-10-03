# ShippingEstimate.java

## Review

## 1. Summary  

`com.salesmanager.core.entity.shipping.ShippingEstimate` is a plain‑old‑Java‑object (POJO) that represents a shipping cost estimate for a specific customer zone.  
Key responsibilities:

| Component | Role |
|-----------|------|
| `regions` | Holds a mapping from zone indices to `ShippingPriceRegion` objects that contain delivery windows (min/max days). |
| `shippingModule`, `shippingType` | Store the name of the shipping carrier and whether the shipment is national or international. |
| `locale`, `currency` | Provide localisation and currency context for labels that will be displayed to the end‑user. |
| `defaultShippingEstimateText` | A fallback string when no region is found or when delivery windows are unspecified. |
| `customerCountry`, `storeCountry` | Used to produce human‑readable descriptions (e.g. “Delivery to France”). |
| `shippingCompanyLogo` | Optional logo URL/name for the carrier. |

The class relies on a small amount of external code:

* `org.apache.commons.lang.StringUtils` – for simple string checks.  
* `com.salesmanager.core.util.LabelUtil` – retrieves i18n text from a message bundle.  
* `com.salesmanager.core.util.LocaleUtil` – obtains a default locale when none is supplied.  

The design is straightforward: expose getters/setters for all fields and provide three convenience methods that build user‑friendly strings (`getShippingEstimateDescription()`, `getShippingCompany()`, `getShippingTypeDescription()`). No business logic is performed beyond formatting, so the class is primarily a data transfer object (DTO).

---

## 2. Detailed Description  

### Data model  

The class holds all information required to display a shipping estimate:

1. **Regions** – a `Map<Integer, ShippingPriceRegion>` keyed by zone index.  
   `ShippingPriceRegion` (not shown) contains at least `minDays` and `maxDays`.  
2. **Shipping metadata** – module name, type, country codes, locale and currency.  
3. **Auxiliary data** – default text, zone index, logo.

### Flow of execution  

#### Initialization  
Typical usage:  
```java
ShippingEstimate estimate = new ShippingEstimate();
estimate.setRegions(regionsMap);
estimate.setCustomerZoneIndex(customerZoneIndex);
estimate.setLocale(Locale.US);
```
The calling code populates the fields; the object itself does not perform any validation.

#### Runtime behaviour  

* `getShippingEstimateDescription()`  
  * Looks up the `ShippingPriceRegion` for the current `customerZoneIndex`.  
  * If the region is missing or its delivery window is unspecified (`-1`), the method returns the `defaultShippingEstimateText`.  
  * Otherwise, it formats a message using `LabelUtil`.  
    * If `minDays == maxDays` → “Delivery to {country} in {days} days”.  
    * Else → “Delivery to {country} in {minDays}–{maxDays} days”.  
  * The method ensures a non‑null locale by falling back to `LocaleUtil.getDefaultLocale()`.

* `getShippingCompany()`  
  * Returns `null` if `shippingModule` is blank.  
  * Otherwise, fetches a localized name via `LabelUtil`.  
  * **Note**: it does not guarantee a non‑null locale.

* `getShippingTypeDescription()`  
  * Returns an empty string if `shippingType` is `null`.  
  * If `NATIONAL`, returns the `storeCountry`.  
  * If `INTERNATIONAL`, uses a localized “international” label.  
  * **Note**: does not guard against a null locale.

#### Cleanup  
There is no explicit cleanup – the object is purely in‑memory and immutable after construction.

### Design choices & assumptions  

* **Mutable POJO** – all fields are mutable via setters, which is convenient for frameworks that rely on JavaBeans conventions (e.g., Hibernate, Spring).  
* **Lazy localisation** – localisation is performed on demand; the object does not pre‑compute strings.  
* **Error handling** – missing or malformed data results in fallback behaviour rather than exceptions (e.g., missing region → default text).  
* **Dependency on `LabelUtil`** – the class delegates all message formatting to this external utility, keeping it free of formatting logic.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `getShippingEstimateDescription()` | Build a human‑readable shipping estimate string based on the selected zone. | none | `String` | None |
| `getShippingCompany()` | Return the carrier name (localized). | none | `String` (or `null`) | None |
| `getShippingTypeDescription()` | Return a description of the shipping type (national vs international). | none | `String` | None |

### Utility/auxiliary methods  

* All standard getters/setters – provide access to the internal state.  
* `getLocale()`/`setLocale()` – used indirectly by the three public methods.  

---

## 4. Dependencies  

| External | Type | Role |
|----------|------|------|
| `org.apache.commons.lang.StringUtils` | 3rd‑party | `isBlank()` helper |
| `com.salesmanager.core.util.LabelUtil` | 3rd‑party (internal) | Internationalised message lookup |
| `com.salesmanager.core.util.LocaleUtil` | 3rd‑party (internal) | Provides a default `Locale` |

All dependencies are either standard Java (`java.util.*`) or project‑specific utilities. No network or I/O operations are performed.

---

## 5. Additional Notes  

### Edge‑case / robustness concerns  

1. **Null locale** – `getShippingCompany()` and `getShippingTypeDescription()` call `label.getText(locale, …)` without checking if `locale` is `null`. If a caller forgets to set the locale, a `NullPointerException` will be thrown.  
2. **Untyped `List`** – the code uses raw `List` in `getShippingEstimateDescription()`; this triggers an unchecked conversion warning and can lead to `ClassCastException` if the list is mis‑used elsewhere.  
3. **Integer autoboxing** – adding `int` values to the `parameters` list relies on autoboxing; while harmless, it is less explicit.  
4. **Unclear contract on `customerZoneIndex`** – negative values default to “unknown zone” but the meaning of `-1` is not documented.  
5. **Missing validation** – no checks that `regions` contains an entry for `customerZoneIndex` or that `shippingType` is non‑null before use.  

### Potential enhancements  

| Idea | Benefit |
|------|---------|
| **Make the class immutable** – pass all data via constructor and remove setters. This would simplify reasoning and thread‑safety. |
| **Use generics for parameters** – `List<Object>` or `List<String>` to avoid raw types. |
| **Add a `Locale` fallback in every public method** – e.g., `if (locale == null) locale = LocaleUtil.getDefaultLocale();`. |
| **Introduce a builder** – to construct the object in a fluent, readable way. |
| **Add Javadoc to clarify semantics** – especially for `customerZoneIndex` and the meaning of `-1` in delivery windows. |
| **Extract the string‑formatting logic into a separate helper** – would make unit‑testing easier. |
| **Use `String.format` or a `MessageFormat`** – instead of building the list manually, directly format the message with parameters. |

### Testing considerations  

* **Unit tests** should verify that:
  * Missing regions → default text.  
  * `minDays == maxDays` → precise message.  
  * `minDays < maxDays` → range message.  
  * `shippingModule` blank → `null`.  
  * `shippingType` national vs international yields correct description.  
* **Locale coverage** – test with at least two locales to ensure localisation works.

---

**Conclusion**  
`ShippingEstimate` is a concise DTO that cleanly separates data from presentation logic. With a few minor safety improvements (locale checks, typed collections) and some documentation, it would be robust and maintainable for use in the larger sales‑manager application.

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
package com.salesmanager.core.entity.shipping;

import java.util.ArrayList;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;

/**
 * This object is used in the catalogue
 * 
 * @author Carl Samson
 * 
 */
public class ShippingEstimate {

	private Map<Integer, ShippingPriceRegion> regions;// index, all regions
														// configured
	private String shippingModule;// shipping company
	private ShippingType shippingType;// national - international

	private String defaultShippingEstimateText = "";
	private String customerCountry;
	private String storeCountry;

	private Locale locale;
	private String currency;

	private String shippingCompanyLogo = null;

	public String getShippingCompanyLogo() {
		return shippingCompanyLogo;
	}

	public void setShippingCompanyLogo(String shippingCompanyLogo) {
		this.shippingCompanyLogo = shippingCompanyLogo;
	}

	public String getCustomerCountry() {
		return customerCountry;
	}

	public void setCustomerCountry(String customerCountry) {
		this.customerCountry = customerCountry;
	}

	public String getDefaultShippingEstimateText() {
		return defaultShippingEstimateText;
	}

	public void setDefaultShippingEstimateText(
			String defaultShippingEstimateText) {
		this.defaultShippingEstimateText = defaultShippingEstimateText;
	}

	private int customerZoneIndex = -1;

	public int getCustomerZoneIndex() {
		return customerZoneIndex;
	}

	public void setCustomerZoneIndex(int customerZoneIndex) {
		this.customerZoneIndex = customerZoneIndex;
	}

	public String getStoreCountry() {
		return storeCountry;
	}

	public void setStoreCountry(String storeCountry) {
		this.storeCountry = storeCountry;
	}

	public void setLocale(Locale locale) {
		this.locale = locale;
	}

	public void setCurrency(String currency) {
		this.currency = currency;
	}

	public ShippingType getShippingType() {
		return shippingType;
	}

	public void setShippingType(ShippingType shippingType) {
		this.shippingType = shippingType;
	}

	public Map<Integer, ShippingPriceRegion> getRegions() {
		return regions;
	}

	public void setRegions(Map<Integer, ShippingPriceRegion> regions) {
		this.regions = regions;
	}

	public String getShippingModule() {
		return shippingModule;
	}

	public void setShippingModule(String shippingModule) {
		this.shippingModule = shippingModule;
	}

	public String getShippingEstimateDescription() {

		if (regions != null) {

			ShippingPriceRegion spr = regions.get(customerZoneIndex);

			if (spr != null) {

				if (spr.getMinDays() == -1 && spr.getMaxDays() == -1) {
					return this.getDefaultShippingEstimateText();
				}

				LabelUtil label = LabelUtil.getInstance();
				String returnText = "";

				if (locale == null) {
					locale = LocaleUtil.getDefaultLocale();
				}
				label.setLocale(locale);

				if (spr.getMinDays() == spr.getMaxDays()) {
					List parameters = new ArrayList();
					parameters.add(this.getCustomerCountry());
					parameters.add(spr.getMaxDays());
					returnText = label.getText(locale,
							"message.delivery.estimate.precise", parameters);
				} else {
					List parameters = new ArrayList();
					parameters.add(this.getCustomerCountry());
					parameters.add(spr.getMinDays());
					parameters.add(spr.getMaxDays());
					returnText = label.getText(locale,
							"message.delivery.estimate.range", parameters);
				}

				return returnText;

			} else {

				return this.getDefaultShippingEstimateText();

			}

		}

		return this.getDefaultShippingEstimateText();

	}

	public String getShippingCompany() {

		if (StringUtils.isBlank(this.getShippingModule())) {
			return null;
		}

		LabelUtil label = LabelUtil.getInstance();
		String shippingCompany = label.getText(locale, "module."
				+ this.getShippingModule());

		return shippingCompany;

	}

	public String getShippingTypeDescription() {

		if (this.getShippingType() == null) {
			return "";
		}

		LabelUtil label = LabelUtil.getInstance();

		String shippingText = "";

		if (this.getShippingType() == ShippingType.NATIONAL) {
			shippingText = this.getStoreCountry();
		} else {
			shippingText = label
					.getText(locale, "label.shipping.international");
		}

		return shippingText;

	}

}



```
