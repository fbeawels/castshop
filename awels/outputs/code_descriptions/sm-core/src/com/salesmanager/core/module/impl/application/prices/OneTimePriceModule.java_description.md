# OneTimePriceModule.java

## Review

## 1. Summary  

`OneTimePriceModule` is a concrete implementation of the `PriceModule` interface that handles pricing for products that are sold as *one‑time* (up‑front) items.  
The module calculates the amount that must be paid immediately (`orderSummary.getOneTimeSubTotal()`), handles special pricing (discounts, limited‑time offers), applies credits, and builds human‑readable notes that are added to the `OrderTotalSummary`.  
The class is heavily intertwined with the domain model (`Order`, `OrderProduct`, `ProductPrice`, `ProductPriceSpecial`, …) and contains business logic for date‑based specials, credit calculations, and price formatting for the front‑end.  

Key design elements  
* **Strategy pattern** – `PriceModule` is an interface; concrete modules (e.g. one‑time, subscription, recurring) can be swapped at runtime.  
* **Utility classes** – `CurrencyUtil`, `DateUtil`, `LabelUtil`, `LocaleUtil`, `ProductUtil` provide domain‑specific helpers.  
* **Use of `BigDecimal`** – all monetary values are represented as `BigDecimal`, but the code sometimes converts to `float` and back, which is a potential source of rounding errors.

## 2. Detailed Description  

### Core flow in `calculateOrderPrice(Order, OrderTotalSummary, OrderProduct, OrderProductPrice, String, Locale)`  

1. **Initialize totals**  
   * `otprice` is taken from the current order summary (or zero if null).  
   * `finalPrice` is the product’s base price multiplied by the quantity.  
   * `otprice` is increased by `finalPrice` and stored back into the order summary.

2. **Build the line note**  
   * A `StringBuffer` (`notes`) is populated with quantity, product name, price and price name.

3. **Special pricing logic**  
   * If a `ProductPriceSpecial` is present, two scenarios are evaluated:  
     * **Time‑window special** – start and end dates encompass the purchase date.  
     * **Duration‑based special** – special applies for a fixed number of days after purchase.  
   * In either case, if the special price is cheaper than the normal price, a “credit” amount is calculated and a red‑colored note is created.  
   * The credit is added to the order summary’s *due‑now credits* and to the order product’s `applicableCreditOneTimeCharge`.

4. **Non‑default price description**  
   * If the product price is not the default price, a line item containing the note and `finalPrice` is appended to the order summary’s “other due now” list.

5. **Return** the updated `OrderTotalSummary`.

### Lifecycle & side effects  

* The method mutates several objects: `orderSummary`, `orderProduct`, and the newly created `OrderTotalLine` objects.  
* No cleanup is required – all state is stored in the order objects that are passed in.

### Assumptions & constraints  

* `order.getDatePurchased()` must not be null; otherwise `pps.getProductPriceSpecialStartDate()` comparisons will throw an NPE.  
* `orderProduct.getApplicableCreditOneTimeCharge()` may be null – the code assumes it returns a non‑null `BigDecimal`.  
* All monetary calculations are performed with `BigDecimal`, but the helper method `ProductUtil.determinePrice` is called via a `floatValue()` conversion, which can lose precision.  
* The module always reports that tax is applicable (`isTaxApplicable()` returns `true`).

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side effects |
|--------|---------|--------|---------|--------------|
| `calculateOrderPrice(Order, OrderTotalSummary, OrderProduct, OrderProductPrice, String)` | Stub – returns the passed `orderSummary` unchanged. | `order`, `orderSummary`, `orderProduct`, `productPrice`, `currency` | `OrderTotalSummary` (unchanged) | None |
| `calculateOrderPrice(Order, OrderTotalSummary, OrderProduct, OrderProductPrice, String, Locale)` | Main pricing logic for one‑time products. | `order`, `orderSummary`, `orderProduct`, `productPrice`, `currency`, `locale` | Updated `orderSummary` | Mutates `orderSummary`, `orderProduct`, creates `OrderTotalLine` objects |
| `isTaxApplicable()` | Indicates whether this price module participates in tax calculations. | – | `true` | None |
| `getPrice(ProductPrice, String)` | Returns the product price (possibly discounted). | `productPrice`, `currency` | `BigDecimal` | None |
| `getHtmlPriceFormated(String, ProductPrice, Locale, String)` | Builds an HTML snippet that shows the product price (and discount if any). | `prefix`, `productPrice`, `locale`, `currency` | `String` (HTML) | None |
| `getPricePrefixText(String, Locale)` | Stub – intended to return a text prefix. | `currency`, `locale` | `null` | None |
| `getPriceSuffixText(String, Locale)` | Stub – intended to return a suffix. | `currency`, `locale` | `""` | None |
| `getPriceText(String, Locale)` | Returns a localized description for this price module. | `currency`, `locale` | `String` | None |

### Utility helpers used inside

* `CurrencyUtil.displayFormatedAmountWithCurrency(BigDecimal, String)` – formats amounts with currency symbols.  
* `ProductUtil.determinePrice(ProductPrice)` – calculates the effective price (base or special).  
* `ProductUtil.getDiscountEndDate(ProductPrice)` – fetches the date when a discount ends.  
* `DateUtil.formatDate(Date)` – formats dates.  
* `LabelUtil.getInstance().getText(Locale, String)` – fetches internationalized labels.

## 4. Dependencies  

| Dependency | Type | Purpose |
|------------|------|---------|
| `java.math.BigDecimal` | Standard | Monetary arithmetic |
| `java.util.Date` / `Calendar` | Standard | Date handling (legacy) |
| `java.util.Locale` | Standard | Internationalisation |
| `org.apache.commons.lang.StringUtils` | Third‑party | String helper (`isBlank`) |
| `com.salesmanager.core.entity.*` | Domain | Order, product, price, etc. |
| `com.salesmanager.core.util.*` | Utility | Currency, date, label, locale, product utilities |
| `com.salesmanager.core.module.model.application.PriceModule` | Interface | Strategy for pricing modules |

No external network or platform‑specific APIs are used. All dependencies are standard or part of the SalesManager core.

## 5. Additional Notes & Recommendations  

### 5.1 Precision & BigDecimal usage  
* The code repeatedly casts `BigDecimal` to `float` and back (`new BigDecimal(ProductUtil.determinePrice(productPrice).floatValue())`).  
* **Impact** – Loss of precision (especially for large amounts or currencies with many decimal places).  
* **Fix** – Use `ProductUtil.determinePrice(productPrice)` directly as a `BigDecimal`, or if necessary convert via `BigDecimal.valueOf(double)` which still retains the double’s precision, but prefer the original `BigDecimal`.

### 5.2 Null‑safety  
* `orderProduct.getApplicableCreditOneTimeCharge()` is used without a null check; if the underlying field is `null`, a `NullPointerException` will be thrown.  
* `pps` (special price) and its dates are checked for null, but the dates themselves may be null leading to `NullPointerException` when calling `before`/`after`.  
* **Recommendation** – Guard against `null` before accessing fields, or initialise defaults in the domain objects.

### 5.3 Date handling  
* Uses legacy `Date` and `Calendar`.  
* **Modern alternative** – `java.time` API (`LocalDate`, `ZonedDateTime`) provides clearer semantics, immutability, and better handling of time zones.  
* **Benefits** – Easier date arithmetic, no `Calendar` pitfalls, and more testable code.

### 5.4 Code duplication  
* The logic for computing the credit and building the red‑colored note is duplicated in both the time‑window and duration‑based special branches.  
* **Solution** – Extract a private helper method (`applySpecialCredit(...)`) to avoid duplication and improve readability.

### 5.5 Internationalisation & UI code  
* `getHtmlPriceFormated` mixes business logic (calculating discount percentages) with presentation logic (HTML string).  
* **Separation of concerns** – Consider moving presentation code to a view/template layer, passing plain data structures (price, discount, endDate) to the view.

### 5.6 Stub methods  
* `getPricePrefixText`, `getPriceSuffixText` return `null`/`""`.  
* **Clarify intent** – If these are not needed, remove them; otherwise implement them or throw an exception to signal missing implementation.

### 5.7 Tax handling  
* `isTaxApplicable()` always returns `true`.  
* If different pricing modules have different tax rules, this method should be configurable (e.g., via a flag or property).

### 5.8 Logging & error handling  
* The method silently ignores exceptional cases (e.g., missing dates).  
* Adding logging at DEBUG level when a special cannot be applied due to missing data would aid troubleshooting.

### 5.9 Performance & scalability  
* The method builds `StringBuffer` objects for notes and HTML; while acceptable for small orders, consider using `StringBuilder` (no synchronization) for slightly better performance.  
* The calculation itself is O(1) per order product – acceptable for typical loads.

### 5.10 Unit tests  
* No tests are present for this class.  
* **Recommendation** – Write comprehensive tests covering:  
  * Normal one‑time price without specials.  
  * Time‑window special active/inactive.  
  * Duration‑based special with/without expiration.  
  * Edge cases: `null` fields, zero quantities, extreme monetary values.  

---

### Bottom line  

The `OneTimePriceModule` implements a clear piece of business logic but suffers from several anti‑patterns (precise monetary arithmetic, legacy date API, duplicated code, mixing of concerns). Addressing the issues above will make the module more robust, maintainable, and easier to test.

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

public class OneTimePriceModule implements PriceModule {

	public OrderTotalSummary calculateOrderPrice(Order order,
			OrderTotalSummary orderSummary, OrderProduct orderProduct,
			OrderProductPrice productPrice, String currency) {
		// TODO Auto-generated method stub
		return orderSummary;
	}

	public OrderTotalSummary calculateOrderPrice(Order order,
			OrderTotalSummary orderSummary, OrderProduct orderProduct,
			OrderProductPrice productPrice, String currency, Locale locale) {
		// TODO Auto-generated method stub


		/**
		 * activation price goes in the oneTime fees
		 */

		BigDecimal finalPrice = null;
		// BigDecimal discountPrice=null;

		// order price this type of price needs an upfront payment
		BigDecimal otprice = orderSummary.getOneTimeSubTotal();
		if (otprice == null) {
			otprice = new BigDecimal(0);
		}

		// the final price is in the product price
		finalPrice = productPrice.getProductPriceAmount();
		int quantity = orderProduct.getProductQuantity();
		finalPrice = finalPrice.multiply(new BigDecimal(quantity));

		otprice = otprice.add(finalPrice);
		orderSummary.setOneTimeSubTotal(otprice);

		ProductPriceSpecial pps = productPrice.getSpecial();

		// Build text

		StringBuffer notes = new StringBuffer();
		notes.append(quantity).append(" x ");
		notes.append(orderProduct.getProductName());
		notes.append(" ");
		notes.append(CurrencyUtil.displayFormatedAmountWithCurrency(
				productPrice.getProductPriceAmount(), currency));
		notes.append(" ");

		notes.append(productPrice.getProductPriceName());

		BigDecimal originalPrice = orderProduct.getOriginalProductPrice();
		if (!productPrice.isDefaultPrice()) {
			originalPrice = productPrice.getProductPriceAmount();
		}

		if (pps != null) {
			if (pps.getProductPriceSpecialStartDate() != null
					&& pps.getProductPriceSpecialEndDate() != null) {
				if (pps.getProductPriceSpecialStartDate().before(
						order.getDatePurchased())
						&& pps.getProductPriceSpecialEndDate().after(
								order.getDatePurchased())) {

					BigDecimal dPrice = new BigDecimal(ProductUtil
							.determinePrice(productPrice).floatValue());

					if (dPrice.floatValue() < productPrice
							.getProductPriceAmount().floatValue()) {

						BigDecimal subTotal = originalPrice
								.multiply(new BigDecimal(orderProduct
										.getProductQuantity()));
						BigDecimal creditSubTotal = pps
								.getProductPriceSpecialAmount().multiply(
										new BigDecimal(orderProduct
												.getProductQuantity()));

						BigDecimal credit = subTotal.subtract(creditSubTotal);

						StringBuffer spacialNote = new StringBuffer();
						spacialNote.append("<font color=\"red\">[");

						spacialNote.append(productPrice.getProductPriceName());

						// spacialNote.append(getPriceText(currency,locale));

						spacialNote.append(" ");
						spacialNote.append(CurrencyUtil
								.displayFormatedAmountWithCurrency(credit,
										currency));

						spacialNote.append("]</font>");

						if (productPrice.getProductPriceAmount().doubleValue() > pps
								.getProductPriceSpecialAmount().doubleValue()) {

							OrderTotalLine line = new OrderTotalLine();
							line.setText(spacialNote.toString());
							line.setCost(credit);
							line.setCostFormated(CurrencyUtil
									.displayFormatedAmountWithCurrency(credit,
											currency));
							orderSummary.addDueNowCredits(line);

							BigDecimal oneTimeCredit = orderProduct
									.getApplicableCreditOneTimeCharge();
							oneTimeCredit = oneTimeCredit.add(credit);
							orderProduct
									.setApplicableCreditOneTimeCharge(oneTimeCredit);
						}
					}

				} else if (pps.getProductPriceSpecialDurationDays() > -1) {

					Date dt = new Date();

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


						BigDecimal subTotal = originalPrice
								.multiply(new BigDecimal(orderProduct
										.getProductQuantity()));
						BigDecimal creditSubTotal = pps
								.getProductPriceSpecialAmount().multiply(
										new BigDecimal(orderProduct
												.getProductQuantity()));

						BigDecimal credit = subTotal.subtract(creditSubTotal);

						StringBuffer spacialNote = new StringBuffer();
						spacialNote.append("<font color=\"red\">[");

						spacialNote.append(productPrice.getProductPriceName());

						// spacialNote.append(getPriceText(currency,locale));
						spacialNote.append(" ");
						spacialNote.append(CurrencyUtil
								.displayFormatedAmountWithCurrency(credit,
										currency));

						spacialNote.append("]</font>");

						if (productPrice.getProductPriceAmount().doubleValue() > pps
								.getProductPriceSpecialAmount().doubleValue()) {

							OrderTotalLine line = new OrderTotalLine();

							line.setText(spacialNote.toString());
							line.setCost(credit);
							line.setCostFormated(CurrencyUtil
									.displayFormatedAmountWithCurrency(credit,
											currency));
							orderSummary.addDueNowCredits(line);

							BigDecimal oneTimeCredit = orderProduct
									.getApplicableCreditOneTimeCharge();
							oneTimeCredit = oneTimeCredit.add(credit);
							orderProduct
									.setApplicableCreditOneTimeCharge(oneTimeCredit);

						}

					}

				}
			}

		}

		if (!productPrice.isDefaultPrice()) {

			// add a price description
			OrderTotalLine scl = new OrderTotalLine();
			scl.setText(notes.toString());
			scl.setCost(finalPrice);
			scl.setCostFormated(CurrencyUtil.displayFormatedAmountWithCurrency(
					finalPrice, currency));
			orderSummary.addOtherDueNowPrice(scl);
		}

		return orderSummary;

	}

	public boolean isTaxApplicable() {
		return true;
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
		p.append("<div class='product-price product-onetime-price'>");
		if (!ProductUtil.hasDiscount(productPrice)) {
			p.append("<div class='product-price-price' style='width:50%;float:left;'>");
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

			p.append("<div class='product-price-price' style='width:50%;float:left;'>");
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

	public String getPricePrefixText(String currency, Locale locale) {
		// TODO Auto-generated method stub
		return null;
	}

	public String getPriceSuffixText(String currency, Locale locale) {
		// TODO Auto-generated method stub
		return "";
	}

	public String getPriceText(String currency, Locale locale) {
		// TODO Auto-generated method stub
		if (locale == null)
			locale = LocaleUtil.getDefaultLocale();

		return LabelUtil.getInstance().getText(locale,
				"module.description.onetime-price");

	}

}



```
