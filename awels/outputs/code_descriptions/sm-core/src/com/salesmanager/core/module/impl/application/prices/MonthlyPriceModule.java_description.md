# MonthlyPriceModule.java

## Review

## 1. Summary

**Purpose**  
`MonthlyPriceModule` implements the `PriceModule` interface to calculate and format recurring monthly pricing for orders. It supports:
- One‑time upfront fees (for the first month)
- Recursive monthly fees (subsequent months)
- Special promotional pricing (discounts, rebates, limited‑time offers)
- HTML rendering of the price for display on the storefront

**Key Components**
| Class/Interface | Responsibility |
|-----------------|----------------|
| `MonthlyPriceModule` | Implements business logic for monthly pricing |
| `ProductPrice`, `ProductPriceSpecial` | Domain entities representing base price and special offers |
| `Order`, `OrderProduct`, `OrderProductPrice`, `OrderTotalSummary`, `OrderTotalLine` | Order‑related entities used to accumulate and present totals |
| Utility classes (`CurrencyUtil`, `DateUtil`, `LabelUtil`, `LocaleUtil`, `ProductUtil`) | Provide formatting, locale handling, and product‑specific calculations |

**Design patterns / frameworks**
- **Strategy** – `PriceModule` is an interface that can be implemented by various pricing strategies (e.g., monthly, yearly, one‑time).  
- **Template Method** – The `calculateOrderPrice` method follows a fixed sequence of steps that can be overridden for different price modules.  
- **Utility/Helper** – A collection of static helper classes (`ProductUtil`, `CurrencyUtil`, etc.) abstract common logic.

---

## 2. Detailed Description

### Execution Flow

1. **Input extraction** – The method pulls data from `Order`, `OrderProduct`, and `OrderProductPrice` objects.  
2. **Base price determination** – Uses `productPrice.getProductPriceAmount()` or the default price if the module uses a non‑default price.  
3. **Quantity handling** – Multiplies the unit price by the ordered quantity.  
4. **One‑time fees** – Adds the computed price to `orderSummary.getOneTimeSubTotal()`.  
5. **Special offers** –  
   - If a start/end date is defined, the method checks if the current order date falls within the window.  
   - If a duration (in days) is defined, the method verifies that the offer is still active relative to the current date.  
   - Computes rebates/discounts and generates credit lines for “due now” and “recursive” summaries.  
6. **Recursive pricing** – Adds the full price to `orderSummary.getRecursiveSubTotal()` and creates an `OrderTotalLine` for the recurring charge.  
7. **Return** – The updated `OrderTotalSummary` is returned.

### Assumptions & Constraints

- The method assumes `productPrice` and `orderProduct` are non‑null; no defensive checks are performed.  
- Dates are handled using `java.util.Date`/`Calendar`; no time‑zone awareness.  
- Prices are represented as `BigDecimal` for the core calculation, but at least one place converts to `float`, risking precision loss.  
- The HTML rendering method uses `StringBuffer` (thread‑safe but slower) instead of `StringBuilder`.  
- The class is tightly coupled to the legacy domain model and several utility singletons.

### Architecture & Design Choices

- **Single Responsibility** – The module violates SRP: it calculates prices **and** formats HTML. A separate presenter/renderer would be cleaner.  
- **Encapsulation** – The heavy logic is embedded inside `calculateOrderPrice`; helper methods would improve readability.  
- **Extensibility** – Adding a new price module is straightforward, but extending the existing logic (e.g., supporting “daily” recursive pricing) requires deep changes to this file.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `calculateOrderPrice(Order, OrderTotalSummary, OrderProduct, OrderProductPrice, String, Locale)` | Main business logic for monthly pricing. Handles one‑time and recursive charges, special offers, and credit lines. | `Order` – order data<br>`OrderTotalSummary` – running totals<br>`OrderProduct` – product in the order<br>`OrderProductPrice` – price record<br>`String currency` – currency code<br>`Locale locale` – language/region | Updated `OrderTotalSummary` | Modifies `orderSummary`, `orderProduct` (applicable credits) |
| `calculateOrderPrice(Order, OrderTotalSummary, OrderProduct, OrderProductPrice, String)` | Unimplemented overload (placeholder). | Same as above but without `Locale` | `null` (stub) | None |
| `getPrice(ProductPrice, String)` | Returns the effective price for a product (uses `ProductUtil.determinePrice`). | `ProductPrice`, currency | `BigDecimal` | None |
| `getHtmlPriceFormated(String, ProductPrice, Locale, String)` | Builds an HTML snippet representing the price, including discounts and savings. | `prefix`, `ProductPrice`, `Locale`, `String currency` | `String` (HTML) | None |
| `getPriceText(String, Locale)` | Returns the description text for the module. | `String currency`, `Locale locale` | `String` | None |
| `getPricePrefixText(String, Locale)` | Returns any prefix text (currently empty). | `String currency`, `Locale locale` | `String` | None |
| `isTaxApplicable()` | Declares tax applicability. | None | `true` | None |
| `getPriceSuffixText(String, Locale)` | Retrieves the suffix text for the recurring monthly price. | `String currency`, `Locale locale` | `String` | None |

**Reusable / Utility methods**  
- `ProductUtil.determinePrice()` – obtains the correct price for a product.  
- `CurrencyUtil.displayFormatedAmountWithCurrency()` – formats monetary values.  
- `LabelUtil.getInstance().getText()` – localization helper.  
- `DateUtil.formatDate()` – date formatting.

---

## 4. Dependencies

| External / Third‑Party | Type | Notes |
|------------------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For checking blank strings. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `java.math.BigDecimal`, `java.util.*` | Standard | Core Java. |
| `com.salesmanager.core.*` | Domain / Utility | Internal project packages. |
| No database or web framework dependencies in this file (the entities are plain POJOs). |

---

## 5. Additional Notes & Recommendations

### 5.1 Precision & Type Safety

- **Mixing `float` and `BigDecimal`**  
  ```java
  BigDecimal dPrice = new BigDecimal(ProductUtil.determinePrice(productPrice).floatValue());
  ```
  This introduces a floating‑point conversion that may truncate or round the value. All calculations should stay within `BigDecimal` to preserve monetary precision.

- **Scale & rounding** – There is no explicit scale or rounding mode for division or multiplication. Consider using `BigDecimal.setScale(2, RoundingMode.HALF_UP)` where appropriate.

### 5.2 Date Handling

- The code uses legacy `Date`/`Calendar`. Java 8’s `java.time` package (e.g., `LocalDate`, `ZonedDateTime`) would simplify date arithmetic and eliminate time‑zone pitfalls.
- `Date dt = new Date(new Date().getTime());` is redundant; just `Date dt = new Date();`.
- Comparison logic (`pps.getProductPriceSpecialStartDate().before(order.getDatePurchased())`) could be replaced with `!order.getDatePurchased().before(pps.getProductPriceSpecialStartDate())`.

### 5.3 Code Duplication & Complexity

- The special‑price handling block is duplicated twice (for start/end dates and for duration days). Extract a helper method to encapsulate:
  - Validation of the offer window.
  - Calculation of rebate/discount.
  - Creation of `OrderTotalLine` objects.
- The string concatenation for HTML could be done with a `StringBuilder` or a template engine for maintainability.

### 5.4 Null & Error Handling

- No null‑checks on `productPrice`, `orderProduct`, or `order`. A `NullPointerException` could occur in many places.
- The method swallows exceptions when retrieving the suffix text; better to log the cause and fallback to a default string.

### 5.5 Unused Method

- `calculateOrderPrice(Order, OrderTotalSummary, OrderProduct, OrderProductPrice, String)` returns `null`. If this overload is not used, it should be removed or fully implemented to avoid confusion.

### 5.6 Performance

- The method performs many BigDecimal multiplications per product; this is expected but could be optimized by caching the quantity as a `BigDecimal` once.
- String concatenation in loops (`notes`, `spacialNote`) can be costly; `StringBuilder` would be preferable.

### 5.7 Testability

- The class is heavily coupled to static utilities and entity state. Extracting the business logic into a pure service with dependency injection would allow unit testing with mocks.

### 5.8 Future Enhancements

| Feature | Description | Benefit |
|---------|-------------|---------|
| **Java 8 Time API** | Replace `Date`/`Calendar` with `LocalDate` | Cleaner code, time‑zone safe |
| **DTO / Value Objects** | Create a `PriceCalculationResult` DTO | Decouples calculation from persistence |
| **Strategy for Offer Types** | Separate classes for “percentage”, “fixed”, “duration” offers | Extensible offer engine |
| **Template Rendering** | Use a templating engine (FreeMarker, Thymeleaf) for HTML | Easier UI changes |
| **Logging improvements** | Use SLF4J + Logback instead of Log4J | Modern logging ecosystem |
| **Unit Tests** | Add JUnit tests for each logical branch | Regression safety |

---

### Conclusion

`MonthlyPriceModule` provides essential pricing logic for recurring monthly purchases, but the implementation is dense and mixes concerns (calculation, credit handling, and HTML rendering). Refactoring into smaller, focused components and adopting modern Java features would improve maintainability, precision, and testability. The class is a solid foundation but would benefit from a systematic clean‑up and modernization.

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
package com.salesmanager.core.module.impl.application.prices;

import java.math.BigDecimal;
import java.util.Calendar;
import java.util.Date;
import java.util.Locale;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.entity.catalog.ProductPrice;
import com.salesmanager.core.entity.catalog.ProductPriceSpecial;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductPrice;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.orders.OrderTotalLine;
import com.salesmanager.core.module.model.application.PriceModule;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.ProductUtil;

/**
 * Designed for handling monthly recursive prices
 * 
 * @author Administrator
 * 
 */
public class MonthlyPriceModule implements PriceModule {

	private Logger log = Logger.getLogger(MonthlyPriceModule.class);

	public OrderTotalSummary calculateOrderPrice(Order order,
			OrderTotalSummary orderSummary, OrderProduct orderProduct,
			OrderProductPrice productPrice, String currency, Locale locale) {

		/**
		 * Monthly price goes in the oneTime fees as well as in the upcoming
		 * recursive fees
		 */

		BigDecimal finalPrice = null;
		BigDecimal discountPrice = null;

		BigDecimal originalPrice = orderProduct.getOriginalProductPrice();
		if (!productPrice.isDefaultPrice()) {
			originalPrice = productPrice.getProductPriceAmount();
		}

		int quantity = orderProduct.getProductQuantity();

		// the real price is the price submited
		finalPrice = orderProduct.getProductPrice();
		finalPrice = finalPrice.multiply(new BigDecimal(quantity));

		// the final price is the product price * quantity

		if (finalPrice == null) {// pick it from the productPrice
			finalPrice = productPrice.getProductPriceAmount();
			finalPrice = finalPrice.multiply(new BigDecimal(quantity));
		}

		// this type of price needs an upfront payment
		BigDecimal otprice = orderSummary.getOneTimeSubTotal();
		if (otprice == null) {
			otprice = new BigDecimal(0);
		}

		otprice = otprice.add(finalPrice);
		orderSummary.setOneTimeSubTotal(otprice);

		ProductPriceSpecial pps = productPrice.getSpecial();

		// Build text
		StringBuffer notes = new StringBuffer();
		notes.append(quantity).append(" x ");
		notes.append(orderProduct.getProductName());
		notes.append(" ");
		if (!productPrice.isDefaultPrice()) {
			notes.append(CurrencyUtil.displayFormatedAmountWithCurrency(
					productPrice.getProductPriceAmount(), currency));
		} else {
			notes.append(CurrencyUtil.displayFormatedAmountWithCurrency(
					orderProduct.getProductPrice(), currency));
		}
		notes.append(" ");
		notes.append(this.getPriceSuffixText(currency, locale));

		if (pps != null) {
			if (pps.getProductPriceSpecialStartDate() != null
					&& pps.getProductPriceSpecialEndDate() != null) {
				if (pps.getProductPriceSpecialStartDate().before(
						order.getDatePurchased())
						&& pps.getProductPriceSpecialEndDate().after(
								order.getDatePurchased())) {

					BigDecimal dPrice = new BigDecimal(ProductUtil
							.determinePrice(productPrice).floatValue());

					BigDecimal subTotal = originalPrice
							.multiply(new BigDecimal(orderProduct
									.getProductQuantity()));
					BigDecimal creditSubTotal = pps
							.getProductPriceSpecialAmount().multiply(
									new BigDecimal(orderProduct
											.getProductQuantity()));
					BigDecimal credit = subTotal.subtract(creditSubTotal);

					if (dPrice.floatValue() < productPrice
							.getProductPriceAmount().floatValue()) {

						discountPrice = productPrice.getProductPriceAmount()
								.subtract(dPrice);

						BigDecimal newPrice = orderProduct.getProductPrice();

						if (!productPrice.isDefaultPrice()) {
							newPrice = productPrice.getProductPriceAmount();
						} else {
							newPrice = newPrice.add(discountPrice);
						}

						StringBuffer spacialNote = new StringBuffer();
						spacialNote.append("<font color=\"red\">[");
						spacialNote.append(orderProduct.getProductName());
						spacialNote.append(" ");
						spacialNote.append(CurrencyUtil
								.displayFormatedAmountWithCurrency(credit,
										currency));
						spacialNote.append(" ");
						spacialNote.append(LabelUtil.getInstance().getText(
								locale, "label.generic.rebate"));
						spacialNote.append(" ");
						spacialNote.append(LabelUtil.getInstance().getText(
								locale, "label.generic.until"));

						spacialNote.append(" ");
						spacialNote.append(DateUtil.formatDate(pps
								.getProductPriceSpecialEndDate()));
						spacialNote.append("]</font>");

						OrderTotalLine line = new OrderTotalLine();
						// BigDecimal credit = discountPrice;
						line.setText(spacialNote.toString());
						line.setCost(credit);
						line.setCostFormated(CurrencyUtil
								.displayFormatedAmountWithCurrency(credit,
										currency));
						orderSummary.addDueNowCredits(line);
						orderSummary.addRecursiveCredits(line);

						BigDecimal oneTimeCredit = orderProduct
								.getApplicableCreditOneTimeCharge();
						oneTimeCredit = oneTimeCredit.add(credit);
						orderProduct
								.setApplicableCreditOneTimeCharge(oneTimeCredit);

					}

				} else if (pps.getProductPriceSpecialDurationDays() > -1) {

					Date dt = new Date(new Date().getTime());

					int numDays = pps.getProductPriceSpecialDurationDays();
					Date purchased = order.getDatePurchased();
					Calendar c = Calendar.getInstance();
					c.setTime(dt);
					c.add(Calendar.DATE, numDays);

					BigDecimal dPrice = new BigDecimal(ProductUtil
							.determinePrice(productPrice).floatValue());

					if (dt.before(c.getTime())
							&& dPrice.floatValue() < productPrice
									.getProductPriceAmount().floatValue()) {

						discountPrice = productPrice.getProductPriceAmount()
								.subtract(dPrice);

						BigDecimal newPrice = orderProduct.getProductPrice();


						BigDecimal subTotal = originalPrice
								.multiply(new BigDecimal(orderProduct
										.getProductQuantity()));
						BigDecimal creditSubTotal = pps
								.getProductPriceSpecialAmount().multiply(
										new BigDecimal(orderProduct
												.getProductQuantity()));
						BigDecimal credit = subTotal.subtract(creditSubTotal);

						if (!productPrice.isDefaultPrice()) {
							newPrice = productPrice.getProductPriceAmount();
						} else {
							newPrice = newPrice.add(discountPrice);
						}

						StringBuffer spacialNote = new StringBuffer();

						spacialNote.append("<font color=\"red\">[");
						spacialNote.append(orderProduct.getProductName());
						spacialNote.append(" ");
						spacialNote.append(CurrencyUtil
								.displayFormatedAmountWithCurrency(credit,
										currency));
						spacialNote.append(" ");
						spacialNote.append(LabelUtil.getInstance().getText(
								locale, "label.generic.rebate"));
						spacialNote.append(" ");
						spacialNote.append(LabelUtil.getInstance().getText(
								locale, "label.generic.until"));

						spacialNote.append(" ");
						spacialNote.append(DateUtil.formatDate(c.getTime()));
						spacialNote.append("]</font>");


						OrderTotalLine line = new OrderTotalLine();


						line.setText(spacialNote.toString());
						line.setCost(credit);
						line.setCostFormated(CurrencyUtil
								.displayFormatedAmountWithCurrency(credit,
										currency));
						orderSummary.addDueNowCredits(line);
						if (numDays > 30) {
							orderSummary.addRecursiveCredits(line);
						}

						BigDecimal oneTimeCredit = orderProduct
								.getApplicableCreditOneTimeCharge();
						oneTimeCredit = oneTimeCredit.add(credit);
						orderProduct
								.setApplicableCreditOneTimeCharge(oneTimeCredit);

						// }

					}

				}

			}

		}


		BigDecimal newPrice = orderProduct.getProductPrice();
		if (!productPrice.isDefaultPrice()) {
			newPrice = productPrice.getProductPriceAmount();
		}

		newPrice = newPrice.multiply(new BigDecimal(quantity));

		// Recursive sub total
		BigDecimal rprice = orderSummary.getRecursiveSubTotal();
		if (rprice == null) {
			rprice = new BigDecimal(0);
		}

		// recursive always contain full price
		rprice = rprice.add(newPrice);
		orderSummary.setRecursiveSubTotal(rprice);

		// recursive price
		OrderTotalLine scl = new OrderTotalLine();
		scl.setText(notes.toString());
		scl.setCost(newPrice);
		scl.setCostFormated(CurrencyUtil.displayFormatedAmountWithCurrency(
				newPrice, currency));
		orderSummary.addRecursivePrice(scl);

		return orderSummary;

	}

	public OrderTotalSummary calculateOrderPrice(Order order,
			OrderTotalSummary orderSummary, OrderProduct orderProduct,
			OrderProductPrice productPrice, String currency) {

		return null;
	}

	public BigDecimal getPrice(ProductPrice productPrice, String currency) {
		// TODO Auto-generated method stub
		return ProductUtil.determinePrice(productPrice);
	}

	public String getHtmlPriceFormated(String prefix,
			ProductPrice productPrice, Locale locale, String currency) {

		if (locale == null)
			locale = LocaleUtil.getDefaultLocale();

		if (currency == null)
			currency = CurrencyUtil.getDefaultCurrency();

		StringBuffer p = new StringBuffer();
		p.append("<div class='product-price product-monthly-price'>");
		if (!ProductUtil.hasDiscount(productPrice)) {
			p.append("<div style='width:50%;float:left;'>");
			if (!StringUtils.isBlank(prefix)) {
				p.append("<div class='product-price-text'><strong>").append(prefix).append(" : </strong></div>");
			}
			p.append(CurrencyUtil.displayFormatedAmountWithCurrency(
					productPrice.getProductPriceAmount(), currency));
			p.append(getPriceSuffixText(currency, locale));
			p.append("</div>");
			p.append("<div class='product-line'>&nbsp;</div>");
		} else {

			double arith = productPrice.getSpecial()
					.getProductPriceSpecialAmount().doubleValue()
					/ productPrice.getProductPriceAmount().doubleValue();
			double fsdiscount = 100 - arith * 100;
			Float percentagediscount = new Float(fsdiscount);
			String savediscount = String.valueOf(percentagediscount.intValue());

			p.append("<div class='product-price product-monthly-price'>");
			if (!StringUtils.isBlank(prefix)) {
				p.append("<div class='product-price-text'><strong>").append(prefix).append(" : </strong></div>");
			}
			p.append("<strike>").append(
					CurrencyUtil.displayFormatedAmountWithCurrency(productPrice
							.getProductPriceAmount(), currency)).append(
					getPriceSuffixText(currency, locale)).append("</strike>")
					.append("</div>").append(
							"<div style='width:50%;float:right;'>").append(
							"<font color='red'>").append(
							CurrencyUtil.displayFormatedAmountWithCurrency(
									ProductUtil.determinePrice(productPrice),
									currency)).append(
							getPriceSuffixText(currency, locale)).append(
							"</font>").append("<br>").append(
							"<font color='red' style='font-size:75%;'>")
					.append(
							LabelUtil.getInstance().getText(locale,
									"label.generic.save")).append(" :").append(
							savediscount).append(
							LabelUtil.getInstance().getText(locale,
									"label.generic.percentsign")).append(" ")
					.append(
							LabelUtil.getInstance().getText(locale,
									"label.generic.off")).append("</font>");

			Date discountEndDate = ProductUtil.getDiscountEndDate(productPrice);

			if (discountEndDate != null) {
				p.append("<br>").append(" <font style='font-size:65%;'>")
						.append(
								LabelUtil.getInstance().getText(locale,
										"label.generic.until"))
						.append("&nbsp;").append(
								DateUtil.formatDate(discountEndDate)).append(
								"</font>");
			}

			p.append("</div>");
		}
		p.append("</div>");
		return p.toString();

	}

	public String getPriceText(String currency, Locale locale) {
		// TODO Auto-generated method stub

		if (locale == null)
			locale = LocaleUtil.getDefaultLocale();

		return LabelUtil.getInstance().getText(locale,
				"module.description.monthly-price");
	}

	public String getPricePrefixText(String currency, Locale locale) {
		return "";
	}

	public boolean isTaxApplicable() {
		return true;
	}

	public String getPriceSuffixText(String currency, Locale locale) {

		String desc = "";
		try {

			if (locale != null) {
				desc = LabelUtil.getInstance().getText(locale,
						"module.suffix.recursive-monthly");
			} else {
				desc = LabelUtil.getInstance().getText(
						"module.suffix.recursive-monthly");
			}

		} catch (Exception e) {
			log.error(e);
		}

		return desc;

	}

}



```
