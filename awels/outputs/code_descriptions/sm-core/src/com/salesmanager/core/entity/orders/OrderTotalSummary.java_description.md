# OrderTotalSummary.java

## Review

## 1. Summary  

**Purpose**  
`OrderTotalSummary` is a plain‑old Java object (POJO) that aggregates all the monetary values that appear on a shopping‑cart or invoice screen. It holds the line‑item breakdown (`OrderProduct[]`), shipping details, tax amounts, and credit adjustments for both one‑time and recurring purchases.

**Key Components**  

| Field | Role |
|-------|------|
| `orderProducts` | Array of products in the cart. |
| `shippingLine` | Shipping and handling information. |
| `recursiveAmounts`, `otherDueNowAmounts`, `taxAmounts`, `dueNowCredits`, `recursiveCredits` | Collections of `OrderTotalLine` objects that represent individual line‑items (e.g., product price, tax, credit). |
| `oneTimeSubTotal`, `recursiveSubTotal`, `taxTotal`, `shippingTotal`, `total` | `BigDecimal` aggregates of the line items. |
| `currency` | String code of the currency (e.g., “USD”). |
| `validationError`, `shipping`, `errorMessage` | UI‑level flags / messages. |

**Design Patterns & Libraries**  
* Uses **Builder‑like** additive methods (`addRecursivePrice`, `addTaxPrice`, etc.) to construct the summary.  
* Relies on **Apache Commons Lang** (`StringUtils`) for null/blank checks.  
* Implements `java.io.Serializable` but omits a `serialVersionUID`.  
* The class mixes arrays (`OrderProduct[]`) with collections (`Collection<OrderTotalLine>`), which is a minor inconsistency.

---

## 2. Detailed Description  

### Initialization  
The constructor requires a `currency` string and initializes every `BigDecimal` aggregate to zero (via `"0"`). All `Collection` fields remain `null` until items are added. No defensive copying is performed for the passed currency value.

### Runtime Behavior  
1. **Adding Line Items** –  
   - `addRecursivePrice`, `addOtherDueNowPrice`, `addTaxPrice`, `addDueNowCredits`, `addRecursiveCredits` lazily instantiate the underlying `ArrayList` if it is `null` and then add the supplied `OrderTotalLine`.  
   - These methods expose the internal collection to the caller through the public `get*` methods, which return the raw `Collection` or convert to an array.  

2. **Aggregating Totals** –  
   - The class does not perform any automatic aggregation. The caller is responsible for summing the line items into the BigDecimal fields (`oneTimeSubTotal`, `recursiveSubTotal`, etc.).  

3. **Display Strings** –  
   - `getTotalText()` appends the currency code to the stored `totalText` value if present.  
   - The other `*Text` fields are simply stored and retrieved; no formatting logic is embedded.

### Cleanup / Serialization  
`OrderTotalSummary` implements `Serializable`, but:
- No `serialVersionUID` is defined → defaults to the hash of the class, making deserialization fragile across changes.  
- There is no custom `writeObject/readObject`; the class relies on default serialization of its fields.  

### Assumptions & Constraints  
* The caller must supply a non‑null currency code or else `getTotalText()` will append a blank string.  
* All `BigDecimal` fields are expected to be non‑null; the constructor guarantees this but other mutators (`setOneTimeSubTotal`, etc.) do not check for `null`.  
* Thread‑safety is **not** guaranteed; concurrent updates to the collections could cause race conditions.  
* The `orderProducts` array is exposed directly via getters/setters; defensive copying would be safer.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `OrderTotalSummary(String currency)` | Constructor; initializes all aggregates to zero and stores the currency code. | `currency` | New instance | Sets internal state |
| `getOrderProducts()` / `setOrderProducts(OrderProduct[])` | Accessors for the array of products. | N/A / array | array | None |
| `getErrorMessage()` / `setErrorMessage(String)` | UI error messaging. | N/A / String | String | None |
| `isValidationError()` / `setValidationError(boolean)` | Flag indicating cart validation status. | N/A / boolean | boolean | None |
| `getOneTimeSubTotal()` / `setOneTimeSubTotal(BigDecimal)` | Getter/Setter for one‑time subtotal. | N/A / BigDecimal | BigDecimal | None |
| `getRecursiveSubTotal()` / `setRecursiveSubTotal(BigDecimal)` | Getter/Setter for recurring subtotal. | N/A / BigDecimal | BigDecimal | None |
| `getRecursiveAmounts()` | Converts internal `recursiveAmounts` collection to an array. | N/A | `OrderTotalLine[]` | None |
| `addRecursivePrice(OrderTotalLine)` | Lazy‑instantiate & add a recursive line item. | `OrderTotalLine` | None | Modifies collection |
| `getOtherDueNowAmounts()` | Converts `otherDueNowAmounts` to array. | N/A | array | None |
| `addOtherDueNowPrice(OrderTotalLine)` | Adds a one‑time line item. | `OrderTotalLine` | None | Modifies collection |
| `getTaxAmounts()` | Converts `taxAmounts` to array. | N/A | array | None |
| `addTaxPrice(OrderTotalLine)` | Adds a tax line item. | `OrderTotalLine` | None | Modifies collection |
| `getTotalText()` | Returns `totalText` optionally suffixed with currency. | N/A | String | None |
| `setTotalText(String)` | Stores the total text for display. | String | None | None |
| `isShipping()` / `setShipping(boolean)` | Flag indicating shipping presence. | N/A / boolean | boolean | None |
| `getShippingLine()` / `setShippingLine(ShippingInformation)` | Accessors for shipping details. | N/A / `ShippingInformation` | `ShippingInformation` | None |
| `getShippingTotal()` / `setShippingTotal(BigDecimal)` | Getter/Setter for shipping total. | N/A / BigDecimal | BigDecimal | None |
| `getShippingTotalText()` / `setShippingTotalText(String)` | Accessors for shipping total display string. | N/A / String | String | None |
| `getDueNowCredits()` / `addDueNowCredits(OrderTotalLine)` | Accessor & adder for one‑time credits. | N/A / `OrderTotalLine` | Collection | Modifies collection |
| `getRecursiveCredits()` / `addRecursiveCredits(OrderTotalLine)` | Accessor & adder for recurring credits. | N/A / `OrderTotalLine` | Collection | Modifies collection |
| `getCurrency()` / `setCurrency(String)` | Accessors for currency code. | N/A / String | String | None |

**Reusable/Utility Methods** – None beyond the basic getters/setters. The conversion of collections to arrays (`get*Amounts()`) could be extracted into a generic helper to avoid code duplication.

---

## 4. Dependencies  

| Library | Role | Standard / Third‑Party |
|---------|------|------------------------|
| `java.math.BigDecimal` | Precise monetary representation | Standard |
| `java.util.*` (ArrayList, Collection) | Internal collections | Standard |
| `org.apache.commons.lang.StringUtils` | Blank/null checks | Third‑Party (Apache Commons Lang) |
| `com.salesmanager.core.entity.shipping.ShippingInformation` | Shipping details | Project‑specific |
| `com.salesmanager.core.entity.orders.OrderProduct`, `OrderTotalLine` | Domain entities | Project‑specific |

No external frameworks (e.g., Spring, Hibernate) are used; the class is a simple POJO.

---

## 5. Additional Notes & Recommendations  

### 5.1 Design & Code Quality  
1. **Generic Raw Types** – All collection fields are declared as raw `ArrayList`/`Collection` and instantiated without generics (`new ArrayList()`).  
   *Fix:* Use `ArrayList<OrderTotalLine>()` and declare the fields with generics:  
   ```java
   private List<OrderTotalLine> recursiveAmounts = new ArrayList<>();
   ```

2. **Array vs Collection** – The class mixes `OrderProduct[]` with `Collection<OrderTotalLine>`.  
   *Fix:* Prefer `List<OrderProduct>` for consistency and flexibility.

3. **Serialization** – Implement a `serialVersionUID` to maintain compatibility.  

4. **Immutable DTO** – The current design allows callers to modify internal state freely (e.g., by holding a reference to `dueNowCredits`). Consider exposing unmodifiable views or defensive copies.  

5. **Null Safety** – Setters (`setOneTimeSubTotal`, etc.) accept `null` and will propagate it to the aggregate fields. Validate inputs or document that `null` is illegal.  

6. **Thread‑Safety** – If the object will be shared across threads, guard the collection mutations (e.g., `Collections.synchronizedList`) or use immutable patterns.  

### 5.2 Functional Enhancements  
1. **Automatic Totals** – Provide methods that compute `oneTimeSubTotal`, `recursiveSubTotal`, `taxTotal`, `shippingTotal`, and `total` by summing the respective line collections. This removes the burden from callers and ensures consistency.  

2. **Currency Formatting** – Instead of plain strings, use `java.util.Currency` and a `NumberFormat` instance to format amounts according to locale.  

3. **Validation** – Encapsulate validation logic (e.g., required shipping when `shipping` is true) inside a dedicated `validate()` method that returns a list of problems.

4. **Builder Pattern** – A fluent `OrderTotalSummaryBuilder` can simplify construction and addition of line items, reducing the risk of leaving fields uninitialized.  

5. **Equality & Hashing** – Override `equals()`, `hashCode()`, and `toString()` for easier debugging and collection usage.  

### 5.3 Edge Cases  
* If `currency` is `null`, `getTotalText()` will append `" null"` because `StringUtils.isBlank(null)` returns `true`, but the string is still appended.  
* The `get*Amounts()` methods return `null` when the collection is empty. It is safer to return an empty array to avoid `NullPointerException` in callers.  
* The class does not handle negative amounts or rounding explicitly; callers must ensure appropriate scale and rounding mode.

### 5.4 Performance  
The current implementation is lightweight. However, converting collections to arrays on every getter incurs allocation overhead. If performance is critical, expose the collections directly or cache the array representation.

---

### Bottom Line  

`OrderTotalSummary` serves its purpose as a data holder for order totals, but it would benefit from modern Java practices:

* Use generics consistently.  
* Prefer lists over arrays.  
* Make defensive copies / return unmodifiable views.  
* Compute totals internally or provide helper methods.  
* Add proper serialization UID and override standard methods.  

Implementing these changes will make the class safer, easier to maintain, and more robust in multi‑threaded or evolving environments.

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
package com.salesmanager.core.entity.orders;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.entity.shipping.ShippingInformation;

/**
 * Object used in the shopping cart page
 * 
 * @author Administrator
 * 
 */
public class OrderTotalSummary implements java.io.Serializable {

	private boolean validationError;
	private boolean shipping;// is there any shipping applicable
	private String errorMessage;// for usage in the shopping cart / invoice
	private String currency = null;

	private OrderProduct[] orderProducts;// product lines

	private ShippingInformation shippingLine;

	private Collection<OrderTotalLine> recursiveAmounts;
	private Collection<OrderTotalLine> otherDueNowAmounts;
	//private Collection<OrderTotalLine> orderTotalAmounts;
	private Collection<OrderTotalLine> taxAmounts;
	private Collection<OrderTotalLine> dueNowCredits;
	private Collection<OrderTotalLine> recursiveCredits;

	private BigDecimal oneTimeSubTotal = null;// due now sub total
	private BigDecimal recursiveSubTotal = null;// upcoming recursive subtotals
	private BigDecimal taxTotal = null;
	private BigDecimal total = null;
	private BigDecimal shippingTotal = null;// contains shipping and handling

	/** used in invoice / shopping cart for display with currency **/
	private String oneTimeSubTotalText = null;
	private String recursiveSubTotalText = null;
	private String totalText = null;
	private String shippingTotalText = null;

	public OrderTotalSummary(String currency) {
		recursiveSubTotal = new BigDecimal("0");
		//recursiveSubTotal.setScale(2, BigDecimal.ROUND_DOWN);
		oneTimeSubTotal = new BigDecimal("0");
		//oneTimeSubTotal.setScale(2, BigDecimal.ROUND_DOWN);
		total = new BigDecimal("0");
		//total.setScale(2, BigDecimal.ROUND_DOWN);
		shippingTotal = new BigDecimal("0");
		//shippingTotal.setScale(2, BigDecimal.ROUND_DOWN);
		taxTotal = new BigDecimal("0");
		//taxTotal.setScale(2, BigDecimal.ROUND_DOWN);
		this.currency = currency;
	}

	public OrderProduct[] getOrderProducts() {
		return orderProducts;
	}

	public void setOrderProducts(OrderProduct[] orderProducts) {
		this.orderProducts = orderProducts;
	}

	public String getErrorMessage() {
		return errorMessage;
	}

	public void setErrorMessage(String errorMessage) {
		this.errorMessage = errorMessage;
	}

	public boolean isValidationError() {
		return validationError;
	}

	public void setValidationError(boolean validationError) {
		this.validationError = validationError;
	}

	public BigDecimal getOneTimeSubTotal() {
		return oneTimeSubTotal;
	}

	public void setOneTimeSubTotal(BigDecimal oneTimeSubTotal) {
		this.oneTimeSubTotal = oneTimeSubTotal;
	}

	public BigDecimal getRecursiveSubTotal() {
		return recursiveSubTotal;
	}

	public void setRecursiveSubTotal(BigDecimal recursiveSubTotal) {
		this.recursiveSubTotal = recursiveSubTotal;
	}

	public OrderTotalLine[] getRecursiveAmounts() {
		if (recursiveAmounts != null) {
			OrderTotalLine[] sArray = new OrderTotalLine[recursiveAmounts
					.size()];
			OrderTotalLine[] o = (OrderTotalLine[]) recursiveAmounts
					.toArray(sArray);
			return o;
		} else {
			return null;
		}
	}

	public void addRecursivePrice(OrderTotalLine recursivePriceLine) {
		if (recursiveAmounts == null) {
			recursiveAmounts = new ArrayList();
		}
		recursiveAmounts.add(recursivePriceLine);
	}

	public OrderTotalLine[] getOtherDueNowAmounts() {
		if (otherDueNowAmounts != null) {
			OrderTotalLine[] sArray = new OrderTotalLine[otherDueNowAmounts
					.size()];
			OrderTotalLine[] o = (OrderTotalLine[]) otherDueNowAmounts
					.toArray(sArray);
			return o;
		} else {
			return null;
		}
	}

	public void addOtherDueNowPrice(OrderTotalLine oneTimePriceLine) {
		if (otherDueNowAmounts == null) {
			otherDueNowAmounts = new ArrayList();
		}
		otherDueNowAmounts.add(oneTimePriceLine);
	}

/*	public OrderTotalLine[] getOrderTotalAmounts() {
		if (orderTotalAmounts != null) {
			OrderTotalLine[] sArray = new OrderTotalLine[orderTotalAmounts
					.size()];
			OrderTotalLine[] o = (OrderTotalLine[]) orderTotalAmounts
					.toArray(sArray);
			return o;
		} else {
			return null;
		}
	}*/

	//public void addOrderTotalPrice(OrderTotalLine orderTotalLine) {
	//	if (orderTotalAmounts == null) {
	//		orderTotalAmounts = new ArrayList();
	//	}
	//	orderTotalAmounts.add(orderTotalLine);
	//}

	public OrderTotalLine[] getTaxAmounts() {
		if (taxAmounts != null) {
			OrderTotalLine[] sArray = new OrderTotalLine[taxAmounts.size()];
			OrderTotalLine[] o = (OrderTotalLine[]) taxAmounts
					.toArray(sArray);
			return o;
		} else {
			return null;
		}
	}

	public void addTaxPrice(OrderTotalLine taxLine) {
		if (taxAmounts == null) {
			taxAmounts = new ArrayList();
		}
		taxAmounts.add(taxLine);
	}

	public String getOneTimeSubTotalText() {
		return oneTimeSubTotalText;
	}

	public void setOneTimeSubTotalText(String oneTimeSubTotalText) {
		this.oneTimeSubTotalText = oneTimeSubTotalText;
	}

	public String getRecursiveSubTotalText() {
		return recursiveSubTotalText;
	}

	public void setRecursiveSubTotalText(String recursiveSubTotalText) {
		this.recursiveSubTotalText = recursiveSubTotalText;
	}

	public BigDecimal getTaxTotal() {
		return taxTotal;
	}

	public void setTaxTotal(BigDecimal taxTotal) {
		this.taxTotal = taxTotal;
	}

	public BigDecimal getTotal() {
		return total;
	}

	public void setTotal(BigDecimal total) {
		this.total = total;
	}

	public String getTotalText() {
		StringBuffer totalTextBuffer = new StringBuffer();
		totalTextBuffer.append(totalText);
		if(!StringUtils.isBlank(this.getCurrency())) {
			totalTextBuffer.append(" ").append(this.getCurrency());
		}
		return totalTextBuffer.toString();
	}

	public void setTotalText(String totalText) {
		this.totalText = totalText;
	}

	public boolean isShipping() {
		return shipping;
	}

	public void setShipping(boolean shipping) {
		this.shipping = shipping;
	}

	public ShippingInformation getShippingLine() {
		return shippingLine;
	}

	public void setShippingLine(ShippingInformation shippingLine) {
		this.shippingLine = shippingLine;
	}

	public BigDecimal getShippingTotal() {
		return shippingTotal;
	}

	public void setShippingTotal(BigDecimal shippingTotal) {
		this.shippingTotal = shippingTotal;
	}

	public String getShippingTotalText() {
		return shippingTotalText;
	}

	public void setShippingTotalText(String shippingTotalText) {
		this.shippingTotalText = shippingTotalText;
	}

	public Collection<OrderTotalLine> getDueNowCredits() {
		return dueNowCredits;
	}

	public void addDueNowCredits(OrderTotalLine dueNowCredit) {
		if (dueNowCredits == null) {
			dueNowCredits = new ArrayList();
		}
		dueNowCredits.add(dueNowCredit);
	}

	public Collection<OrderTotalLine> getRecursiveCredits() {
		return recursiveCredits;
	}

	public void addRecursiveCredits(OrderTotalLine recursiveCredit) {
		if (recursiveCredits == null) {
			recursiveCredits = new ArrayList();
		}
		recursiveCredits.add(recursiveCredit);
	}

	public String getCurrency() {
		return currency;
	}

	public void setCurrency(String currency) {
		this.currency = currency;
	}

}



```
