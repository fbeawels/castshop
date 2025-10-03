# OrderUtil.java

## Review

## 1. Summary  

`OrderUtil` is a utility class that converts an `OrderTotalSummary` into a `Map<String, OrderTotal>` suitable for rendering on the front‑end.  
The method `getOrderTotals` iterates over the various components of an order (sub‑total, shipping, taxes, credits, recurring fees, etc.) and creates an `OrderTotal` DTO for each one. The DTO contains a human‑readable title, a formatted text string (including the currency symbol), the numeric value, and a sort order used for display ordering.  

Key components  
| Component | Role |
|-----------|------|
| `OrderTotalSummary` | Holds the raw numerical values for the order. |
| `OrderConstants` | Provides string constants that are used as keys in the result map and as identifiers for the different total types. |
| `LabelUtil` | Retrieves localized labels for the various totals. |
| `CurrencyUtil` | Formats `BigDecimal` amounts into currency strings. |
| `LinkedHashMap` | Maintains insertion order, which is used to preserve the UI ordering. |

The class relies on standard Java libraries (collections, `BigDecimal`, `Locale`) and a few domain‑specific entities (`OrderTotal`, `OrderTotalLine`, `OrderTotalSummary`).

---

## 2. Detailed Description  

1. **Method signature**  
   ```java
   public static Map<String, OrderTotal> getOrderTotals(
           long orderId, OrderTotalSummary summary,
           String currency, Locale locale) throws Exception
   ```  
   The method is static and stateless, making it thread‑safe but it throws a very generic `Exception`, which forces callers to handle all exceptions.

2. **Initialisation**  
   * A `LinkedHashMap` (raw type) is created to preserve the insertion order.  
   * A singleton `LabelUtil` is obtained and its locale is set; subsequently each call to `label.getText()` passes the locale again, which is redundant.

3. **Processing each component**  
   * **Other due‑now amounts** – iterates over `summary.getOtherDueNowAmounts()`, creates a `OrderTotal` for each line, assigns a module identifier (`OT_OTHER_DUE_NOW`), sets a sort order starting at 40, and stores it in the map.  
   * **Due‑now credits** – sums all credits from `summary.getDueNowCredits()` (a `Collection`), then creates a single `OrderTotal` for the cumulative credit.  
   * **Sub‑total** – if present, creates an `OrderTotal` for the sub‑total (sort order 200).  
   * **Shipping** – included only if a shipping total is present **and** `summary.isShipping()` is true.  
   * **Taxes** – iterates over `summary.getTaxAmounts()`, creating a separate `OrderTotal` for each tax line (sort order 400+index).  
   * **Total** – adds a `OrderTotal` for the overall total, but the guard mistakenly checks `shipping != null` instead of `total != null`.  
   * **Recurring amounts** – similar to other sections; the code sums all recurring fees and creates a single `OrderTotal` (sort order 600).  
   * **Recurring credits** – sums recurring credits and creates a single `OrderTotal` (sort order 700).  

4. **Return** – the fully populated map is returned.

### Assumptions & Constraints  
* `OrderTotalSummary` methods are expected to return non‑null arrays/collections (or null if not applicable).  
* The numeric values are always non‑negative (credits are treated as positive sums; no sign handling).  
* The caller supplies a non‑null `locale` and a non‑null `currency` string.  
* The method is intended to run on a server with no concurrency issues.

### Architecture & Design Choices  
* The util keeps all formatting logic inside a single method; it couples presentation concerns (currency formatting, localisation) with the business data.  
* `LinkedHashMap` is chosen to preserve the natural order that the UI expects.  
* The method returns a generic `Map`, which forces callers to cast or use raw types.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `public static Map<String, OrderTotal> getOrderTotals(...)` | Build a map of `OrderTotal` DTOs from an `OrderTotalSummary`. | `orderId` (order identifier), `summary` (data source), `currency` (ISO code or symbol), `locale` (for localisation). | `Map<String, OrderTotal>` – keys are constants or constant + index. | Sets locale on `LabelUtil`; populates map. |

*There are no helper methods; all logic resides inside `getOrderTotals`.*

---

## 4. Dependencies  

| Library / Class | Type | Notes |
|-----------------|------|-------|
| `java.math.BigDecimal` | Standard | Numeric handling. |
| `java.util.*` | Standard | Collections, Locale. |
| `com.salesmanager.core.constants.OrderConstants` | Domain | String constants for map keys and module identifiers. |
| `com.salesmanager.core.entity.orders.OrderTotal` | Domain | DTO for presentation. |
| `com.salesmanager.core.entity.orders.OrderTotalLine` | Domain | Raw line items (text + cost). |
| `com.salesmanager.core.entity.orders.OrderTotalSummary` | Domain | Aggregated order data. |
| `LabelUtil` | Domain | Localization. |
| `CurrencyUtil` | Domain | Currency formatting. |

All external dependencies are part of the same project; no third‑party libraries are used.

---

## 5. Additional Notes  

### Potential Bugs / Edge Cases  

1. **Wrong guard for total** – the code checks `if (shipping != null)` when adding the total, so the total will never be added if shipping is `null`. It should be `if (total != null)`.  
2. **Redundant locale setting** – `label.setLocale(locale)` is followed by passing `locale` to every `getText()` call; the former is unnecessary.  
3. **Raw types** – `Map` and `Collection` are used without generics, leading to unchecked warnings and possible `ClassCastException`.  
4. **Generic `Exception`** – the method declares `throws Exception`, but no checked exceptions are thrown. Either remove the clause or throw a more specific exception.  
5. **Negative credits** – credits are summed as positive values. If the business model requires credits to be subtracted, the sign should be handled explicitly.  
6. **Empty or null `summary`** – a `NullPointerException` will be thrown if `summary` is `null`. A defensive check would improve robustness.  
7. **Inconsistent naming** – `recur`/`recuringCollection` are misspelled; consider `recurring` to avoid confusion.  
8. **Hard‑coded sort orders** – using magic numbers (40, 100, 200, etc.) is fragile; constants or an enum would make the intent clearer.

### Suggested Refactor / Enhancements  

* **Introduce generics** – change all raw collections to parameterised types (`Map<String, OrderTotal>`, `Collection<OrderTotalLine>`, etc.).  
* **Extract helper method** – create a private method `addOrderTotal` that encapsulates the creation of an `OrderTotal` object; this reduces duplication.  
* **Replace magic numbers** – define an `enum` or static final constants for the sort order values.  
* **Move localisation out of the util** – pass the resolved title strings to the method rather than performing localisation inside.  
* **Improve documentation** – add Javadoc comments explaining each section and the expected format of the keys.  
* **Unit tests** – write tests covering all branches (with/without shipping, with/without credits, etc.) to guarantee correct behaviour.  

Overall, the code achieves its goal of building an order totals map, but it suffers from a few maintainability and correctness issues that should be addressed before deployment.

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
import java.util.Collection;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.Locale;
import java.util.Map;

import com.salesmanager.core.constants.OrderConstants;
import com.salesmanager.core.entity.orders.OrderTotal;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.orders.OrderTotalLine;

public class OrderUtil {

	public static Map<String, OrderTotal> getOrderTotals(long orderId,
			OrderTotalSummary summary, String currency, Locale locale)
			throws Exception {

		Map returnMap = new LinkedHashMap();

		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(locale);

		// other fees
		OrderTotalLine[] other = summary.getOtherDueNowAmounts();
		if (other != null) {

			for (int i = 0; i < other.length; i++) {

				OrderTotalLine line = other[i];
				OrderTotal o = new OrderTotal();
				o.setModule(OrderConstants.OT_OTHER_DUE_NOW);
				o.setOrderId(orderId);
				o.setTitle(line.getText());
				o.setText(CurrencyUtil.displayFormatedAmountWithCurrency(line
						.getCost(), currency));
				o.setValue(line.getCost());
				o.setSortOrder(40 + i);
				returnMap.put(OrderConstants.OT_OTHER_DUE_NOW + "_" + i, o);
			}

		}

		// credits
		Collection dueNowCreditsCollection = summary.getDueNowCredits();
		if (dueNowCreditsCollection != null
				&& dueNowCreditsCollection.size() > 0) {

			Iterator dueNowIterator = dueNowCreditsCollection.iterator();
			BigDecimal credit = new BigDecimal("0");
			while (dueNowIterator.hasNext()) {

				OrderTotalLine line = (OrderTotalLine) dueNowIterator
						.next();
				credit = credit.add(line.getCost());

			}

			OrderTotal o = new OrderTotal();

			o.setModule(OrderConstants.OT_CREDITS);
			o.setOrderId(orderId);
			o.setTitle(label.getText(locale, "label.order.ordertotal.credits"));
			o.setText(CurrencyUtil.displayFormatedAmountWithCurrency(credit,
					currency));
			o.setValue(credit);
			o.setSortOrder(100);

			returnMap.put(OrderConstants.OT_CREDITS, o);

		}

		BigDecimal subTotal = summary.getOneTimeSubTotal();
		if (subTotal != null) {
			OrderTotal o = new OrderTotal();
			o.setModule(OrderConstants.OT_SUBTOTAL_MODULE);
			o.setOrderId(orderId);
			o.setTitle(label.getText(locale, "label.cart.subtotal"));
			o.setText(CurrencyUtil.displayFormatedAmountWithCurrency(subTotal,
					currency));
			o.setValue(subTotal);
			o.setSortOrder(200);
			returnMap.put(OrderConstants.OT_SUBTOTAL_MODULE, o);
		}

		BigDecimal shipping = summary.getShippingTotal();
		if (shipping != null && summary.isShipping()) {
			OrderTotal o = new OrderTotal();
			o.setModule(OrderConstants.OT_SHIPPING_MODULE);
			o.setOrderId(orderId);
			o.setTitle(label.getText(locale, "label.cart.shipping"));
			o.setText(CurrencyUtil.displayFormatedAmountWithCurrency(shipping,
					currency));
			o.setValue(shipping);
			o.setSortOrder(300);
			returnMap.put(OrderConstants.OT_SHIPPING_MODULE, o);
		}

		OrderTotalLine[] taxLines = summary.getTaxAmounts();
		if (taxLines != null) {

			for (int i = 0; i < taxLines.length; i++) {

				OrderTotalLine line = taxLines[i];
				OrderTotal o = new OrderTotal();
				o.setModule(OrderConstants.OT_TAX_MODULE);
				o.setOrderId(orderId);
				o.setTitle(line.getText());
				o.setText(CurrencyUtil.displayFormatedAmountWithCurrency(line
						.getCost(), currency));
				o.setValue(line.getCost());
				o.setSortOrder(400 + i);
				returnMap.put(OrderConstants.OT_TAX_MODULE + "_" + i, o);
			}

		}

		BigDecimal total = summary.getTotal();
		if (shipping != null) {
			OrderTotal o = new OrderTotal();
			o.setModule(OrderConstants.OT_TOTAL_MODULE);
			o.setOrderId(orderId);
			o.setTitle(label.getText(locale, "label.cart.total"));
			o.setText(CurrencyUtil.displayFormatedAmountWithCurrency(total,
					currency));
			o.setValue(total);
			o.setSortOrder(500);
			returnMap.put(OrderConstants.OT_TOTAL_MODULE, o);
		}

		// Add recuring
		OrderTotalLine[] recuringCollection = summary.getRecursiveAmounts();
		if (recuringCollection != null && recuringCollection.length > 0) {

			BigDecimal recur = new BigDecimal("0");
			for (int i = 0; i < recuringCollection.length; i++) {

				OrderTotalLine line = (OrderTotalLine) recuringCollection[i];
				recur = recur.add(line.getCost());

			}

			OrderTotal o = new OrderTotal();

			o.setModule(OrderConstants.OT_RECURING);
			o.setOrderId(orderId);
			o
					.setTitle(label.getText(locale,
							"label.order.ordertotal.recuring"));
			o.setText(CurrencyUtil.displayFormatedAmountWithCurrency(recur,
					currency));
			o.setValue(recur);
			o.setSortOrder(600);

			returnMap.put(OrderConstants.OT_RECURING, o);

		}

		// @todo recuring credits

		Collection recuringCreditsCollection = summary.getRecursiveCredits();
		if (recuringCreditsCollection != null
				&& recuringCreditsCollection.size() > 0) {

			Iterator recuringIterator = recuringCreditsCollection.iterator();
			BigDecimal credit = new BigDecimal("0");
			while (recuringIterator.hasNext()) {

				OrderTotalLine line = (OrderTotalLine) recuringIterator
						.next();
				credit = credit.add(line.getCost());

			}

			OrderTotal o = new OrderTotal();

			o.setModule(OrderConstants.OT_RECURING_CREDITS);
			o.setOrderId(orderId);
			o.setTitle(label.getText(locale,
					"label.order.ordertotal.recuringcredits"));
			o.setText(CurrencyUtil.displayFormatedAmountWithCurrency(credit,
					currency));
			o.setValue(credit);
			o.setSortOrder(700);

			returnMap.put(OrderConstants.OT_RECURING_CREDITS, o);

		}

		return returnMap;

	}

}



```
