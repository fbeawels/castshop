# PackageDetail.java

## Review

## 1. Summary  

**Purpose**  
`PackageDetail` is a lightweight, serialisable Java bean that encapsulates the physical attributes of a shipping package – weight, dimensions, quantity, and a product name. It also exposes “display” getters that format these numeric values as strings using a currency/measurement formatter (`CurrencyUtil`).  

**Key Components**  

| Component | Role |
|-----------|------|
| **Numeric fields** (`shippingWeight`, `shippingLength`, etc.) | Store the raw measurement values as `double`. |
| **String fields** (`weight`, `maxWeight`, etc.) | Intended for UI‑level display but are never used in the current logic. |
| **`currency`** | Holds the ISO‑4217 currency code; defaults to the system setting if not supplied. |
| **Formatting helpers** (`CurrencyUtil.displayMeasure`) | Convert numeric values into locale‑aware string representations. |
| **`PropertiesUtil` & `Constants`** | Retrieve the default currency from a configuration store. |

**Design Patterns / Libraries**  
- The class follows the **JavaBean** convention (private fields with public getters/setters).  
- Uses **lazy initialization** for the `currency` field.  
- Relies on a custom utility class (`CurrencyUtil`) and a configuration helper (`PropertiesUtil`) – both are internal to the `com.salesmanager.core` package.  

---

## 2. Detailed Description  

### Core Flow  
1. **Construction** – No explicit constructor is defined; Java provides a default no‑arg constructor.  
2. **Setting values** – The caller populates the numeric fields via setters (e.g., `setShippingWeight`).  
3. **Formatting** – When a *display* getter (e.g., `getWeight()`) is called, the numeric value is wrapped in a `BigDecimal`, passed to `CurrencyUtil.displayMeasure`, and returned as a formatted string.  
4. **Currency handling** – `getCurrency()` lazily resolves the currency code from the configuration if not already set.  
5. **Cleanup** – None needed; the class is stateless after construction.

### Interaction & Dependencies  
- **`CurrencyUtil`** performs the heavy lifting of measurement formatting; the class assumes it correctly handles locale, units, and rounding.  
- **`PropertiesUtil`** retrieves the default currency via a key (`core.system.defaultcurrency`).  
- **`Constants`** supplies a fallback currency (`Constants.CURRENCY_CODE_USD`).  

### Assumptions & Constraints  
- Numeric fields are `double`; no validation for negative or zero values.  
- The string fields (`weight`, `height`, etc.) are never used in formatting; they appear to be remnants of an earlier design.  
- The class is **serialisable** (`implements Serializable`) but does not define `serialVersionUID`.  
- Thread‑safety is not a concern for a typical JavaBean; however, the lazy init of `currency` is not synchronized.

### Architecture & Design Choices  
- **Separate numeric & string representations**: The pattern of storing raw values and providing formatted strings is common, but the redundant string fields make the class bloated.  
- **No validation or business logic**: The bean is purely a data holder; any constraints should be enforced elsewhere.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `getCurrency()` | Lazy‑loads the currency code from config if `null`. | None | `String` | May read from `PropertiesUtil` |
| `setCurrency(String)` | Sets the currency code explicitly. | `String` | void | assigns to field |
| `getShippingHeight()/setShippingHeight(double)` | Accessor for height. | `double` | `double` | None |
| `getShippingLength()/setShippingLength(double)` | Accessor for length. | `double` | `double` | None |
| `getShippingMaxWeight()/setShippingMaxWeight(double)` | Accessor for max weight. | `double` | `double` | None |
| `getShippingQuantity()/setShippingQuantity(int)` | Accessor for quantity. | `int` | `int` | None |
| `getShippingWeight()/setShippingWeight(double)` | Accessor for weight. | `double` | `double` | None |
| `getShippingWidth()/setShippingWidth(double)` | Accessor for width. | `double` | `double` | None |
| `getHeight()/setHeight(String)` | Returns formatted height; sets unused string field. | `String` | `String` | Sets internal string |
| `getLength()/setLength(String)` | Returns formatted length; sets unused string field. | `String` | `String` | Sets internal string |
| `getMaxWeight()/setMaxWeight(String)` | Returns formatted max weight; sets unused string field. | `String` | `String` | Sets internal string |
| `getWeight()/setWeight(String)` | Returns formatted weight; sets unused string field. | `String` | `String` | Sets internal string |
| `getWidth()/setWidth(String)` | Returns formatted width; sets unused string field. | `String` | `String` | Sets internal string |
| `getTreshold()/setTreshold(int)` | Accessor for “threshold” (misspelled). | `int` | `int` | None |
| `getProductName()/setProductName(String)` | Accessor for product name. | `String` | `String` | None |

**Reusable / Utility Methods**  
- The formatted getters (`getWeight`, `getHeight`, etc.) all delegate to `CurrencyUtil.displayMeasure`.  
- `getCurrency()` provides a convenient way to obtain the configured default.

---

## 4. Dependencies  

| Library / Class | Type | Notes |
|-----------------|------|-------|
| `com.salesmanager.core.constants.Constants` | Third‑party internal | Provides default currency code. |
| `com.salesmanager.core.util.CurrencyUtil` | Third‑party internal | Handles measurement formatting. |
| `com.salesmanager.core.util.PropertiesUtil` | Third‑party internal | Reads configuration. |
| `java.math.BigDecimal` | JDK | For precise numeric conversion. |
| `java.io.Serializable` | JDK | Enables object serialization. |

*All external dependencies are internal to the `com.salesmanager.core` package; no standard JDK classes beyond `BigDecimal` and `Serializable` are used.*

---

## 5. Additional Notes  

### Strengths  
- **Clear separation** between raw numeric values and formatted string representations.  
- **Lazy initialization** of currency reduces upfront cost if the value is never accessed.  
- **Serializable** bean can be easily stored in HTTP sessions or persisted.

### Weaknesses & Edge Cases  
1. **Redundant String fields** (`weight`, `height`, etc.) are set but never read; they should be removed or repurposed.  
2. **Naming typo** – `treshold` should be `threshold`.  
3. **Thread‑safety** – `currency` is lazily initialised without synchronization; concurrent access could produce race conditions.  
4. **Precision** – Using `double` for measurements can introduce rounding errors; `BigDecimal` would be safer.  
5. **Validation** – No checks for negative or unrealistic dimensions/weights.  
6. **Locale** – `CurrencyUtil.displayMeasure` likely formats numbers but may not handle locale changes or unit conversions.  
7. **SerialVersionUID** – Missing explicit `serialVersionUID` may cause `InvalidClassException` if the class changes.  

### Suggested Improvements  
- **Remove unused string fields** and expose only numeric values; let UI layers handle formatting.  
- **Rename `treshold` → `threshold`** and update references.  
- **Make `currency` final** or synchronise its lazy init.  
- **Replace `double` with `BigDecimal`** for all measurement fields to avoid floating‑point inaccuracies.  
- **Add validation** in setters (e.g., non‑negative values).  
- **Implement `equals()`, `hashCode()`, and `toString()`** for better debugging and collection usage.  
- **Document assumptions** (e.g., units, currency behaviour) in Javadoc.  
- **Define `serialVersionUID`** to preserve serialization compatibility.  
- **Consider immutability**: provide a constructor that sets all fields, remove setters, and expose only getters.

By addressing these points, `PackageDetail` would become a cleaner, safer, and more maintainable component of the shipping subsystem.

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

import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.PropertiesUtil;

public class PackageDetail implements java.io.Serializable {

	private double shippingWeight;
	private double shippingMaxWeight;
	private double shippingLength;
	private double shippingHeight;
	private double shippingWidth;
	private int shippingQuantity;
	private int treshold;

	private String weight;
	private String maxWeight;
	private String length;
	private String height;
	private String width;
	private String productName;

	private String currency;

	public String getCurrency() {
		if (currency == null) {
			currency = PropertiesUtil.getConfiguration().getString(
					"core.system.defaultcurrency", Constants.CURRENCY_CODE_USD);
		}
		return currency;
	}

	public void setCurrency(String currency) {
		this.currency = currency;
	}

	public double getShippingHeight() {
		return shippingHeight;
	}

	public void setShippingHeight(double shippingHeight) {
		this.shippingHeight = shippingHeight;
	}

	public double getShippingLength() {
		return shippingLength;
	}

	public void setShippingLength(double shippingLength) {
		this.shippingLength = shippingLength;
	}

	public double getShippingMaxWeight() {
		return shippingMaxWeight;
	}

	public void setShippingMaxWeight(double shippingMaxWeight) {
		this.shippingMaxWeight = shippingMaxWeight;
	}

	public int getShippingQuantity() {
		return shippingQuantity;
	}

	public void setShippingQuantity(int shippingQuantity) {
		this.shippingQuantity = shippingQuantity;
	}

	public double getShippingWeight() {
		return shippingWeight;
	}

	public void setShippingWeight(double shippingWeight) {
		this.shippingWeight = shippingWeight;
	}

	public double getShippingWidth() {
		return shippingWidth;
	}

	public void setShippingWidth(double shippingWidth) {
		this.shippingWidth = shippingWidth;
	}

	public String getHeight() {
		return CurrencyUtil.displayMeasure(new BigDecimal(this
				.getShippingHeight()), currency);
	}

	public void setHeight(String height) {

		this.height = height;
	}

	public String getLength() {
		return CurrencyUtil.displayMeasure(new BigDecimal(this
				.getShippingLength()), currency);

	}

	public void setLength(String length) {
		this.length = length;
	}

	public String getMaxWeight() {
		return CurrencyUtil.displayMeasure(new BigDecimal(this
				.getShippingMaxWeight()), currency);

	}

	public void setMaxWeight(String maxWeight) {
		this.maxWeight = maxWeight;
	}

	public String getWeight() {

		return CurrencyUtil.displayMeasure(new BigDecimal(this
				.getShippingWeight()), currency);

	}

	public void setWeight(String weight) {
		this.weight = weight;
	}

	public String getWidth() {
		return CurrencyUtil.displayMeasure(new BigDecimal(this
				.getShippingWidth()), currency);

	}

	public void setWidth(String width) {
		this.width = width;
	}

	public int getTreshold() {
		return treshold;
	}

	public void setTreshold(int treshold) {
		this.treshold = treshold;
	}

	public String getProductName() {
		return productName;
	}

	public void setProductName(String productName) {
		this.productName = productName;
	}

}



```
