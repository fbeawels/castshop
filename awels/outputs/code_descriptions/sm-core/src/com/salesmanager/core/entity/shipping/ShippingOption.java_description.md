# ShippingOption.java

## Review

## 1. Summary  
`ShippingOption` is a plain‑old Java object (POJO) that represents a single shipping method/choice in the SalesManager core domain.  
It stores metadata such as the shipping module, description, dates, and pricing information, and exposes convenient accessors for both raw data and a user‑friendly price string. The class relies on two small utility classes (`CurrencyUtil` and `PropertiesUtil`) to format the price according to the system’s default currency.

Key points  
* **Serializable** – allows shipping options to be persisted or transferred in a JVM‑agnostic manner.  
* **Price handling** – stores the price as a `BigDecimal` and offers a helper (`getOptionPriceText`) that formats the amount with a currency symbol.  
* **No business logic** – the class is essentially a data holder; any calculation or validation is delegated elsewhere.

## 2. Detailed Description  
### Core fields  
| Field | Type | Purpose |
|-------|------|---------|
| `currency` | `String` | ISO‑4217 code (e.g., `"USD"`). Lazy‑loaded from system properties if null. |
| `optionId` | `String` | Unique identifier for the shipping option. |
| `description` | `String` | Human‑readable description. |
| `module` | `String` | Name of the shipping module (e.g., “UPS”, “FedEx”). |
| `shippingDate` / `deliveryDate` | `String` | Dates (no type safety; stored as plain text). |
| `optionPrice` | `BigDecimal` | Monetary value in the chosen currency. |
| `optionName` | `String` | Friendly name. |
| `optionCode` | `String` | Short code (e.g., `"STD"`, `"EXP"`) that can be used in the UI. |
| `estimatedNumberOfDays` | `String` | Textual estimate (could be an integer or formatted string). |

### Execution flow  
* **Construction** – No explicit constructor; default constructor is implicit. Fields are set via setters.  
* **Price formatting** – When `getOptionPriceText()` is called, the method:
  1. Lazily resolves `currency` if it’s `null` by reading the `core.system.defaultcurrency` property.
  2. Delegates to `CurrencyUtil.displayFormatedAmountWithCurrency()` to format the `BigDecimal` price with the currency symbol.
* **Parsing price from text** – `setOptionPriceText(String)` accepts a numeric string and converts it to `BigDecimal`. No validation is performed, so malformed strings will throw `NumberFormatException`.

### Design choices  
* **Mutable POJO** – The class uses standard getters/setters and does not enforce immutability.  
* **Utility‑based formatting** – Keeps the formatting logic out of the domain object, allowing easier internationalization.  
* **Minimal error handling** – The class trusts callers to provide correct values.

## 3. Functions/Methods  
| Method | Purpose | Parameters | Return | Side‑Effects / Notes |
|--------|---------|------------|--------|---------------------|
| `getOptionPriceText()` | Formats the `optionPrice` into a currency string. | None | `String` | Lazily initializes `currency` if needed. Depends on `PropertiesUtil` & `CurrencyUtil`. |
| `getDescription()` | Getter for `description`. | None | `String` |  |
| `setDescription(String)` | Setter for `description`. | `String` | void |  |
| `getOptionPrice()` | Getter for raw `BigDecimal`. | None | `BigDecimal` |  |
| `setOptionPrice(BigDecimal)` | Setter for raw price. | `BigDecimal` | void |  |
| `getOptionId()` | Getter for identifier. | None | `String` |  |
| `setOptionId(String)` | Setter for identifier. | `String` | void |  |
| `getCurrency()` | Getter for currency code. | None | `String` |  |
| `setCurrency(String)` | Setter for currency code. | `String` | void |  |
| `getModule()` | Getter for module name. | None | `String` |  |
| `setModule(String)` | Setter for module name. | `String` | void |  |
| `getDeliveryDate()` | Getter for delivery date. | None | `String` |  |
| `setDeliveryDate(String)` | Setter for delivery date. | `String` | void |  |
| `getShippingDate()` | Getter for shipping date. | None | `String` |  |
| `setShippingDate(String)` | Setter for shipping date. | `String` | void |  |
| `setOptionPriceText(String)` | Parses a string into `BigDecimal` and assigns it to `optionPrice`. | `String` | void | Throws `NumberFormatException` if the string is invalid. |
| `getOptionName()` | Getter for friendly name. | None | `String` |  |
| `setOptionName(String)` | Setter for friendly name. | `String` | void |  |
| `getOptionCode()` | Getter for code. | None | `String` |  |
| `setOptionCode(String)` | Setter for code. | `String` | void |  |
| `getEstimatedNumberOfDays()` | Getter for estimate. | None | `String` |  |
| `setEstimatedNumberOfDays(String)` | Setter for estimate. | `String` | void |  |

*Reusable utilities*: None within this class; all business logic is delegated to external helpers.

## 4. Dependencies  
| Dependency | Type | Purpose |
|------------|------|---------|
| `java.math.BigDecimal` | JDK | Precise monetary representation. |
| `com.salesmanager.core.util.CurrencyUtil` | Third‑party / internal | Formats `BigDecimal` with locale/currency. |
| `com.salesmanager.core.util.PropertiesUtil` | Third‑party / internal | Reads system configuration properties. |
| `java.io.Serializable` | JDK | Marks the class as serializable. |

All dependencies are either standard JDK classes or internal utilities provided by the SalesManager codebase.

## 5. Additional Notes  
### Edge cases / limitations  
1. **Null handling** – `getOptionPriceText()` will throw a `NullPointerException` if `optionPrice` is `null`. No defensive copy or null‑check is performed.  
2. **Date format** – Dates are stored as plain `String`. Any parsing or validation of the format must happen elsewhere.  
3. **Estimate field** – `estimatedNumberOfDays` is a `String` though the name suggests a numeric value. This can lead to confusion or parsing overhead downstream.  
4. **Immutability** – The class is fully mutable, which may lead to unintended side‑effects when shared across threads or across layers.  
5. **Equality & Hashing** – `equals()`, `hashCode()`, and `toString()` are not overridden. Using instances in collections (e.g., `Set`) may yield surprising results.  
6. **No `serialVersionUID`** – While not strictly required, providing one would avoid unexpected `InvalidClassException`s if the class evolves.  

### Suggested improvements  
| Area | Recommendation |
|------|----------------|
| **Null safety** | Add a guard in `getOptionPriceText()` or require `optionPrice` to be non‑null in the constructor. |
| **Validation** | Validate the format of `price` in `setOptionPriceText(String)` or provide a safer factory method. |
| **Date handling** | Use `java.time.LocalDate` or similar types instead of raw `String`. |
| **Estimate type** | Replace `String` with an `int` or dedicated type if the value is numeric. |
| **Immutability** | Consider making the class immutable (final fields, no setters) or at least providing a builder pattern. |
| **`equals`/`hashCode`** | Override these methods based on a unique key (e.g., `optionId`). |
| **`serialVersionUID`** | Declare a constant to stabilize serialization. |
| **Documentation** | Add Javadoc comments to each method for clarity, especially the formatting behavior. |

### Future extensions  
* Add support for multi‑currency pricing (e.g., map of currency codes to `BigDecimal`).  
* Integrate with a validation framework (e.g., Hibernate Validator) to enforce field constraints.  
* Provide JSON/XML serialization helpers if the object is used in REST APIs.  

Overall, `ShippingOption` is a clean, minimal data holder suitable for its intended purpose, but it would benefit from tighter validation, better type safety, and richer contract definitions to improve robustness in a larger system.

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

import java.math.BigDecimal;

import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.PropertiesUtil;

public class ShippingOption implements java.io.Serializable {

	private String currency = null;
	private String optionId;
	private String description;
	private String module;
	private String shippingDate;
	private String deliveryDate;
	private BigDecimal optionPrice;
	private String optionName;
	private String optionCode;
	private String estimatedNumberOfDays = "";

	public String getOptionPriceText() {
		if (currency == null) {
			currency = PropertiesUtil.getConfiguration().getString(
					"core.system.defaultcurrency", "USD");
		}

		return CurrencyUtil.displayFormatedAmountWithCurrency(this
				.getOptionPrice(), currency);

	}

	public String getDescription() {
		return description;
	}

	public void setDescription(String description) {
		this.description = description;
	}

	public BigDecimal getOptionPrice() {
		return optionPrice;
	}

	public void setOptionPrice(BigDecimal optionPrice) {
		this.optionPrice = optionPrice;
	}

	public String getOptionId() {
		return optionId;
	}

	public void setOptionId(String optionId) {
		this.optionId = optionId;
	}

	public String getCurrency() {
		return currency;
	}

	public void setCurrency(String currency) {
		this.currency = currency;
	}

	public String getModule() {
		return module;
	}

	public void setModule(String module) {
		this.module = module;
	}

	public String getDeliveryDate() {
		return deliveryDate;
	}

	public void setDeliveryDate(String deliveryDate) {
		this.deliveryDate = deliveryDate;
	}

	public String getShippingDate() {
		return shippingDate;
	}

	public void setShippingDate(String shippingDate) {
		this.shippingDate = shippingDate;
	}

	public void setOptionPriceText(String price) {
		this.optionPrice = new BigDecimal(price);
	}

	public String getOptionName() {
		return optionName;
	}

	public void setOptionName(String optionName) {
		this.optionName = optionName;
	}

	public String getOptionCode() {
		return optionCode;
	}

	public void setOptionCode(String optionCode) {
		this.optionCode = optionCode;
	}

	public String getEstimatedNumberOfDays() {
		return estimatedNumberOfDays;
	}

	public void setEstimatedNumberOfDays(String estimatedNumberOfDays) {
		this.estimatedNumberOfDays = estimatedNumberOfDays;
	}

}



```
