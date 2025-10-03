# ProductUtil.java

## Review

## 1. Summary  

`ProductUtil` is a **static helper** class that centralises all logic dealing with product pricing, discounts, and display formatting in the Sales Manager core layer.  
Key responsibilities include:

| Feature | What it does | Where it lives |
|---------|--------------|----------------|
| Price calculation | Determines final price (regular, discounted, attribute‑adjusted) | `determinePrice…` methods |
| Discount handling | Detects, formats, and applies discount rules | `hasDiscount`, `getDiscountEndDate`, `determinePrice(ProductPrice)` |
| Display helpers | Formats prices and attributes for UI output (HTML, CSS, labels) | `formatHTMLProductPrice…` |
| Order conversion | Populates an `OrderProduct` instance from a `Product` and related services | `initOrderProduct` |
| Utility lookups | Fetch product name by locale | `getProductName` |

The class uses **third‑party** libraries:  
* `org.apache.commons.lang.StringUtils`  
* `org.apache.log4j.Logger`  
* `java.math.BigDecimal` (Java SE)  
plus internal Sales Manager services (`CatalogService`, `OrderService`, `RefCache`, etc.).

The design follows a procedural style (static methods, heavy use of `Iterator`, manual loops, raw `Set`/`Map` types). There is no use of generics, no modern Java 8+ streams or lambdas, and no unit‑testable interfaces – the code is essentially a collection of static utilities.

---

## 2. Detailed Description  

### 2.1 Flow of Execution  

| Method | Typical Call‑Path | Side Effects | Notes |
|--------|-------------------|--------------|-------|
| `initOrderProduct` | Called by the checkout controller when building an `OrderProduct` from a `Product` | Mutates the passed `OrderProduct`, sets prices, attributes, discount specials | Uses two services (`OrderService`, `CatalogService`) that are resolved via a static factory (`ServiceFactory`). Throws generic `Exception` wrapped in a log – no propagation to the caller. |
| `formatHTMLProductPrice…` | Render price blocks on product pages | Returns an HTML string; internally builds the string with manual string concatenation. | Duplicated logic between `formatHTMLProductPriceWithAttributes` and `formatHTMLProductPrice`. |
| `determinePrice…` | Called by front‑end to display the effective price | Pure (no side effects) | Several overloaded versions – confusing API. |
| `hasDiscount` / `hasDiscount(ProductPrice)` | Detects an active discount | Pure (except for `new Date()`) | Very similar logic to `determinePrice` – duplicated. |
| `getDiscountEndDate` | Computes the end date of a discount | Pure | Complexity of date handling can be simplified. |

### 2.2 Core Components & Interaction  

* **Entities** (`Product`, `ProductPrice`, `ProductPriceSpecial`, `Special`, `ProductAttribute`, `OrderProduct`, …) provide the domain data.  
* **Cache** (`RefCache.getCurrenciesListWithCodes()`) supplies currency definitions.  
* **Service layer** (`CatalogService`, `OrderService`) provide product and order look‑ups.  
* **Locale/Label util** (`LabelUtil`) fetches i18n strings for formatting.  
* **CurrencyUtil** performs formatting of `BigDecimal` values into strings with or without symbols.

The class treats these components as global singletons, fetching them at runtime. This is fine for a small utility but limits testability (no dependency injection) and hides the heavy service resolution cost behind every call.

### 2.3 Design & Style Issues  

1. **Generics are missing** – all collections are raw `Set`/`Map`.  
2. **Manual iteration** (e.g., `Iterator` loops) – makes the code verbose and error‑prone.  
3. **Duplicated logic** – price and discount calculations appear in multiple places with identical conditions.  
4. **Date handling** uses `new Date()` in many spots; this is *not* thread‑safe if the code were to be used in a background thread, and it makes unit testing difficult.  
5. **Error handling** – `initOrderProduct` swallows any `Exception`, logs it, and returns a partially‑initialised object. The caller has no clue that the initialisation failed.  
6. **String concatenation** – heavy use of `StringBuffer`/`StringBuilder` (manually via `+`) can be replaced with templating or `StringJoiner`.  
7. **No caching** of computed results – each call recomputes discounts from scratch.  
8. **Hard‑coded currency conversion logic** – the code sets a `Currency c` variable that is never used.  
9. **No separation of concerns** – UI rendering (`formatHTML…`) and business logic (`determinePrice…`) live in the same class.  
10. **Lack of unit‑tests** – no public interfaces or mockable dependencies.

---

## 3. Functions / Methods  

Below is a quick reference of the public API. The overload hierarchy is confusing; I recommend consolidating the API.

| Method | Signature | Purpose | Comments |
|--------|-----------|---------|----------|
| `initOrderProduct(OrderProduct, Product, Collection<Long>)` | Populate an `OrderProduct` | Mutates input | Service lookup inside; no `throws` – hides failures. |
| `getProductName(Product, Locale)` | Returns product name for a locale | Pure | Could use `product.getProductName()` if that method already localises. |
| `determinePriceNoDiscount(Product, Locale, String)` | Base price (ignoring discounts) | Pure | Duplicate of `determinePrice`. |
| `determinePriceNoDiscountWithAttributes(Product, Collection<Long>, Locale, String)` | Base price plus attribute additions | Pure | Raw collection of `Long`s; no generics. |
| `determinePriceWithAttributes(Product, Collection, Locale, String)` | Price after discount + attributes | Pure | Overloaded with `Collection` and `Collection<Long>` – ambiguous. |
| `determinePrice(Product, Locale, String)` | Final price (regular or discounted) | Pure | This is the “canonical” price calculation. |
| `getDiscountEndDate(ProductPrice)` | Compute discount end date | Pure | Complex date logic. |
| `hasDiscount(Product)` / `hasDiscount(ProductPrice)` | Detect active discount | Pure | Duplicate with `determinePrice`. |
| `determinePrice(ProductPrice)` | Resolve price for a `ProductPrice` object | Pure | Duplicate logic. |
| `determinePrice(OrderProductPrice)` | Resolve price for an `OrderProductPrice` | Pure | Very similar to `determinePrice(ProductPrice)`. |
| `formatHTMLProductPrice…` | Render price blocks | Returns HTML string | Two almost identical methods – refactor into a single one. |
| `formatHTMLProductPriceWithAttributes` | Render price with attribute selection | Returns HTML string | Duplicate logic with `formatHTMLProductPrice`. |

**Recommendation**: expose a small, non‑duplicated API.  
* Keep only **one** `determinePrice(Product, Locale, String)` that handles attributes via a `Collection<ProductAttribute>` argument.  
* Move UI formatting into a separate class (`ProductViewFormatter`) so that `ProductUtil` stays pure business logic.  
* Provide an interface for the discount logic (`DiscountStrategy`) that can be mocked in tests.

---

## 3.1 Duplication & Re‑usability  

| Duplicate area | Methods involved | Why it matters | Suggested fix |
|-----------------|------------------|----------------|---------------|
| Price/discount computation | `determinePrice(Product)`, `determinePrice(ProductPrice)`, `determinePrice(OrderProductPrice)` | Same logic repeated, making maintenance hard. | Extract a **private** helper (`applySpecial(ProductPriceSpecial, BigDecimal, Date)`) and reuse it. |
| Attribute price addition | `determinePriceNoDiscountWithAttributes`, `determinePriceWithAttributes`, `determinePrice(Product)` | Manual iteration over attributes appears in 3 places. | Create a private `attributesPrice(Collection)` helper returning a `BigDecimal`. |
| HTML rendering | `formatHTMLProductPriceWithAttributes`, `formatHTMLProductPrice` | 90 % identical code blocks. | Merge into one method that accepts an optional attribute collection. |
| Discount detection | `hasDiscount(Product)`, `hasDiscount(ProductPrice)` | Same date‑range logic. | Reuse `hasDiscount(ProductPrice)` inside `hasDiscount(Product)`. |
| Currency lookup | `determinePrice…` and `format…` repeatedly call `RefCache.getCurrenciesListWithCodes()` | Expensive cache hit on every call. | Cache the `Currency` in a local variable or inject it. |

---

## 3.2 Performance & Thread‑Safety  

* **Thread‑safety** – all methods are static and operate on immutable data (`BigDecimal`, `Product` fields) except for `initOrderProduct`. Because `OrderProduct` is mutated, concurrent checkout requests that share the *same* `OrderProduct` instance could clash. In practice, each request creates its own `OrderProduct`, so this is unlikely, but documenting this assumption would be prudent.  
* **Performance** – most operations are O(n) over the small sets of prices/attributes. The biggest cost is the **service look‑ups** inside `initOrderProduct`. These are inevitable but could be cached per product in the `OrderProduct` after first retrieval.  
* **BigDecimal rounding** – the code repeatedly calls `setScale(decimalPlace, BigDecimal.ROUND_HALF_UP)` in many places. This can be encapsulated in a helper method to avoid duplication.

---

## 3.3 Error Handling  

| Method | Current behaviour | Suggested behaviour |
|--------|-------------------|---------------------|
| `initOrderProduct` | Catches `Exception`, logs, *silently* returns a partially‑initialized product. | Throw a checked exception (`ProductInitializationException`) so callers can react (e.g., show error to user). |
| All methods using `new Date()` | Creates a new timestamp each call – makes unit tests fragile. | Inject a `Clock` (Java 8+) or provide a protected `now()` method that can be overridden/mocked. |
| `formatHTML…` | Builds a string by concatenation, no null‑checks on intermediate results. | Use `StringJoiner` or a templating library; guard against null attributes or label look‑ups. |

---

## 3.4 Code‑Quality Issues  

| Category | Issue | Impact | Suggested Fix |
|----------|-------|--------|---------------|
| **Generics / type safety** | `Set` and `Map` raw types (`Set prices = product.getPrices();`) | Compile‑time unchecked warnings, risk of `ClassCastException`. | Use generics (`Set<ProductPrice> prices`) and `Map<String, Currency>`. |
| **Iterator misuse** | Manual `Iterator` loops where enhanced for‑loop would suffice. | Verbose, more room for mistakes. | Replace with `for (ProductPrice pp : product.getPrices())`. |
| **String concatenation** | Manual `+` in loops, resulting in many temporary strings. | Performance overhead, difficult to read. | Use `StringBuilder`, `StringJoiner`, or a template engine (e.g., Thymeleaf, JSP). |
| **Hard‑coded dates** | `new Date()` inside many methods – no timezone consideration. | Time‑zone drift, tests fail on day change. | Pass a `Clock` or `Date` to the methods, or use `Instant.now()` with `ZoneId`. |
| **Magic numbers** | `decimalPlace = 2`, `-1` for “no duration” etc. | Magic values are scattered. | Create constants (`DEFAULT_DECIMAL_PLACES`, `DURATION_NOT_SET`). |
| **Logging** | Uses `logger.info` for normal operation, `logger.error` for exceptions only. | Logging noise can hide real issues. | Keep a single logger instance, use proper levels (`debug` for internal computation). |
| **No unit‑testing hooks** | Methods rely on static service factory. | Hard to mock in unit tests. | Provide dependency injection or static `Supplier` fields that can be overridden in tests. |
| **Duplicated discount logic** | Several methods implement the same discount range check. | Inconsistent behaviour if logic changes in one place but not the other. | Centralise the discount calculation in a single method, e.g., `applySpecial(ProductPriceSpecial, BigDecimal, Date now)`. |
| **Method overloads with different parameter types** | `determinePriceNoDiscountWithAttributes(Product, Collection<Long>…)` vs. `determinePriceWithAttributes(Product, Collection…)` | Confusing API, potential accidental overload resolution. | Rename to `determinePriceWithoutDiscountWithAttributes` / `determinePriceWithAttributes` and keep a single public signature. |

---

## 3.5 Security & Business Rules  

* **Discount validity** – The code uses only the current timestamp (`new Date()`) to decide if a discount applies. It does **not** consider *purchase time* or *customer status* (e.g., loyalty discounts). If such rules exist elsewhere they are invisible to this util.  
* **Currency conversion** – Prices are stored as `BigDecimal` in the product’s *local* currency; formatting always displays the currency *symbol* but never performs actual conversion. If the system ever needs multi‑currency support (e.g., store in USD, display in EUR), the util will not honour that.  
* **Attribute prices** – The code adds the attribute price to the base price *unconditionally* if `getOptionValuePrice().longValue() > 0`. There is no check for *stock* or *availability* – a product could show a price for an unavailable attribute.  

---

## 3.6 Recommendations  

| Area | Recommendation | Benefit |
|------|----------------|---------|
| **Modernisation** | Adopt Java 8+ streams, lambdas, and `Optional`. | Cleaner code, fewer explicit loops, better type safety. |
| **API design** | Consolidate the overloaded `determinePrice` methods into one that accepts a `PriceContext` object (contains product, attributes, locale, currency). | Reduces confusion and duplication. |
| **Separation of concerns** | Move HTML formatting to a dedicated view formatter (e.g., `ProductPriceFormatter`). Keep `ProductUtil` purely *business* logic. | Improves testability and readability. |
| **Dependency injection** | Inject `CatalogService`, `OrderService`, `CurrencyCache`, `LabelUtil` via constructor or static setter (used only in tests). | Enables mocking in unit tests, reduces hidden global state. |
| **Caching** | Cache attribute price sums per product–attribute set if this calculation is frequent. | Saves repeated `BigDecimal` additions on large product catalogs. |
| **Date handling** | Use `java.time` API (`Instant`, `ZonedDateTime`) and `Clock` to make tests deterministic. | Removes reliance on `java.util.Date` and `Calendar`, reduces timezone bugs. |
| **Exception handling** | Replace generic `try/catch (Exception e)` in `initOrderProduct` with specific checked exceptions or propagate them. | Gives callers a chance to react (e.g., display an error page). |
| **Logging** | Use `logger.debug` for internal calculations, `logger.error` for unexpected failures. | Keeps log output meaningful. |
| **Constants** | Extract magic numbers (`decimalPlace = 2`, `DURATION_NOT_SET = -1`) into `public static final` constants. | Easier to tweak and document. |
| **Null safety** | Guard against `null` in `Product.getPrices()` and `ProductAttribute.getOptionValuePrice()`. | Avoids `NullPointerException` in production. |
| **Internationalisation** | Centralise label look‑ups; pass a `LabelResolver` rather than calling `LabelUtil.getInstance()` repeatedly. | Makes the class easier to test and localise. |

---

## 3.7 Sample Refactor (price calculation)  

```java
public static BigDecimal determinePrice(Product product, Locale locale, String currency) {
    BigDecimal price = getBasePrice(product, locale, currency);
    return applyActiveSpecial(price, product.getSpecial(), new Date())
           .orElse(price);
}

private static BigDecimal getBasePrice(Product product, Locale locale, String currency) {
    BigDecimal base = product.getProductPrice();
    for (ProductPrice pp : product.getPrices()) {
        pp.setLocale(locale);
        if (pp.isDefaultPrice()) {
            base = pp.getProductPriceAmount();
            break;
        }
    }
    return base;
}

private static Optional<BigDecimal> applyActiveSpecial(BigDecimal price,
                                                      Special special,
                                                      Date now) {
    if (special == null) return Optional.empty();
    ProductPriceSpecial sp = special.getSpecial();
    if (sp == null) return Optional.empty();

    if (sp.isActive(now)) {           // you would add isActive() to Special
        return Optional.of(sp.getProductPriceSpecialAmount());
    }
    // Duration based discount – calculate end date only if needed
    return Optional.empty();
}
```

*Uses* generics, a helper `isActive(Date now)` on the special entity, and Java 8 `Optional`.  
This reduces duplicated loops and centralises discount logic.

---

## 3.8 Final Thoughts  

`ProductUtil` is a **work‑in‑progress** utility class that successfully encapsulates complex pricing rules but at the cost of:

* **Maintainability** – duplication and raw types hinder readability.  
* **Testability** – static methods and hidden service look‑ups make unit tests brittle.  
* **Scalability** – repeated service look‑ups and manual loops may become performance bottlenecks in a large catalog.

A **top‑down refactor** (introducing service interfaces, dependency injection, and a cleaner API) would pay dividends in the long run. For the immediate next sprint, I recommend:

1. Removing the duplicate `formatHTMLProductPrice…` methods and creating a single `PriceRenderer` component.  
2. Consolidating all discount‑related logic into a `DiscountService` that can be mocked.  
3. Refactoring `initOrderProduct` to throw a checked exception and allow the caller to decide how to handle a failure.  

These changes will make the codebase easier to understand, test, and extend while keeping the current behaviour intact.

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
import java.util.Calendar;
import java.util.Collection;
import java.util.Date;
import java.util.Iterator;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttribute;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.catalog.ProductPrice;
import com.salesmanager.core.entity.catalog.ProductPriceSpecial;
import com.salesmanager.core.entity.catalog.Special;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductPrice;
import com.salesmanager.core.entity.orders.OrderProductPriceSpecial;
import com.salesmanager.core.entity.reference.Currency;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.order.OrderService;

public class ProductUtil {

	private static Logger log = Logger.getLogger(ProductUtil.class);

	/**
	 * Set product information to Order product
	 * @param orderProduct
	 * @param currency
	 * @return
	 */
	public static OrderProduct initOrderProduct(OrderProduct orderProduct,
			String currency) {
		try {

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);
			
			CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);
			Product product = cservice.getProduct(orderProduct.getProductId());
			
			int maxOrderQuantity = CatalogConstants.DEFAULT_MAX_ORDER_BY_PRODUCT;
			if(product!=null) {
				maxOrderQuantity = product.getProductQuantityOrderMax();
			}
			
			orderProduct.setProductQuantityOrderMax(maxOrderQuantity);
			orderProduct.setPriceFormated(CurrencyUtil
					.displayFormatedAmountWithCurrency(orderProduct
							.getProductPrice(), currency));
			double finalPrice = orderProduct.getProductPrice().doubleValue()
					* orderProduct.getProductQuantity();
			//revise precision
			BigDecimal bdFinalPrice = new BigDecimal(finalPrice);
			orderProduct.setCostText(CurrencyUtil
					.displayFormatedAmountWithCurrency(bdFinalPrice, currency));
			orderProduct.setPriceText(CurrencyUtil
					.displayFormatedAmountNoCurrency(orderProduct
							.getProductPrice(), currency));
			orderProduct.setFinalPrice(bdFinalPrice);
			orderProduct.setQuantityText(String.valueOf(orderProduct
					.getProductQuantity()));
			orderProduct.setPriceText(CurrencyUtil
					.displayFormatedAmountNoCurrency(orderProduct
							.getProductPrice(), currency));
			if (orderProduct.getOrderattributes() != null
					&& orderProduct.getOrderattributes().size() > 0) {
				orderProduct.setAttributes(true);
				orderProduct.setAttributesLine(CheckoutUtil
						.getAttributesLine(orderProduct));
			}

			// get all discounts associated with prices
			Set prices = orderProduct.getPrices();
			if (prices != null) {
				Iterator i = prices.iterator();
				while (i.hasNext()) {
					OrderProductPrice opp = (OrderProductPrice) i.next();
					// get specials
					OrderProductPriceSpecial opps = oservice
							.getOrderProductPriceSpecial(opp
									.getOrderProductPrice());
					if (opps != null) {
						ProductPriceSpecial pps = new ProductPriceSpecial();
						pps.setOriginalPriceAmount(opps.getOriginalPrice());
						pps.setProductPriceSpecialAmount(opps
								.getSpecialNewProductPrice());
						pps.setProductPriceSpecialDurationDays(opps
								.getOrderProductSpecialDurationDays());
						pps.setProductPriceSpecialEndDate(opps
								.getOrderProductSpecialEndDate());
						pps.setProductPriceSpecialStartDate(opps
								.getOrderProductPriceSpecialStartDate());
						opp.setSpecial(pps);
					}

				}
			}

		} catch (Exception e) {
			log.error(e);
		}
		return orderProduct;
	}

	public static String getProductName(Product product, Locale locale) {

		try {

			Set descriptionset = product.getDescriptions();
			int lang = LanguageUtil.getLanguageNumberCode(locale.getLanguage());
			if (descriptionset != null) {
				Iterator i = descriptionset.iterator();
				while (i.hasNext()) {
					ProductDescription desc = (ProductDescription) i.next();
					if (desc.getId().getLanguageId() == lang) {
						return desc.getProductName();
					}
				}
			}

		} catch (Exception e) {
			log.error(e);
		}

		return "";

	}

	/**
	 * Format like <blue>[if discount striked]SYMBOL BASEAMOUNT</blue> [if
	 * discount <red>SYMBOL DISCOUNTAMOUNT</red>] [if discount <red>Save:
	 * PERCENTAGE AMOUNT</red>] [if qty discount <red>Buy QTY save [if price qty
	 * discount AMOUNT] [if percent qty discount PERCENT]</red>]
	 * 
	 * @param ctx
	 * @param view
	 * @return
	 */
	public static String formatHTMLProductPriceWithAttributes(Locale locale,
			String currency, Product view, Collection attributes,
			boolean showDiscountDate) {

		if (currency == null) {
			log.error("Currency is null ...");
			return "-N/A-";
		}

		int decimalPlace = 2;

		String prefix = "";
		String suffix = "";

		Map currenciesmap = RefCache.getCurrenciesListWithCodes();
		Currency c = (Currency) currenciesmap.get(currency);

		// regular price
		BigDecimal bdprodprice = view.getProductPrice();

		// determine any properties prices
		BigDecimal attributesPrice = null;

		if (attributes != null) {
			Iterator i = attributes.iterator();
			while (i.hasNext()) {
				ProductAttribute attr = (ProductAttribute) i.next();
				if (!attr.isAttributeDisplayOnly()
						&& attr.getOptionValuePrice().longValue() > 0) {
					if (attributesPrice == null) {
						attributesPrice = new BigDecimal(0);
					}
					attributesPrice = attributesPrice.add(attr
							.getOptionValuePrice());
				}
			}
		}

		if (attributesPrice != null) {
			bdprodprice = bdprodprice.add(attributesPrice);// new price with
															// properties
		}

		// discount price
		java.util.Date spdate = null;
		java.util.Date spenddate = null;
		BigDecimal bddiscountprice = null;
		Special special = view.getSpecial();
		if (special != null) {
			bddiscountprice = special.getSpecialNewProductPrice();
			if (attributesPrice != null) {
				bddiscountprice = bddiscountprice.add(attributesPrice);// new
																		// price
																		// with
																		// properties
			}
			spdate = special.getSpecialDateAvailable();
			spenddate = special.getExpiresDate();
		}

		// all other prices
		Set prices = view.getPrices();
		if (prices != null) {
			Iterator pit = prices.iterator();
			while (pit.hasNext()) {
				ProductPrice pprice = (ProductPrice) pit.next();
				if (pprice.isDefaultPrice()) {
					pprice.setLocale(locale);
					suffix = pprice.getPriceSuffix();
					bddiscountprice = null;
					spdate = null;
					spenddate = null;
					bdprodprice = pprice.getProductPriceAmount();
					if (attributesPrice != null) {
						bdprodprice = bdprodprice.add(attributesPrice);// new
																		// price
																		// with
																		// properties
					}
					ProductPriceSpecial ppspecial = pprice.getSpecial();
					if (ppspecial != null) {
						if (ppspecial.getProductPriceSpecialStartDate() != null
								&& ppspecial.getProductPriceSpecialEndDate() != null) {
							spdate = ppspecial
									.getProductPriceSpecialStartDate();
							spenddate = ppspecial
									.getProductPriceSpecialEndDate();
						}
						bddiscountprice = ppspecial
								.getProductPriceSpecialAmount();
						if (bddiscountprice != null && attributesPrice != null) {
							bddiscountprice = bddiscountprice
									.add(attributesPrice);// new price with
															// properties
						}
					}
					break;
				}
			}
		}

		double fprodprice = 0;
		;
		if (bdprodprice != null) {
			fprodprice = bdprodprice.setScale(decimalPlace,
					BigDecimal.ROUND_HALF_UP).doubleValue();
		}

		// regular price String
		String regularprice = CurrencyUtil
				.displayFormatedCssAmountWithCurrency(bdprodprice, currency);

		Date dt = new Date();

		// discount price String
		String discountprice = null;
		String savediscount = null;
		if (bddiscountprice != null
				&& (spdate != null && spdate.before(new Date(dt.getTime())) && spenddate
						.after(new Date(dt.getTime())))) {

			double fdiscountprice = bddiscountprice.setScale(decimalPlace,
					BigDecimal.ROUND_HALF_UP).doubleValue();

			discountprice = CurrencyUtil.displayFormatedAmountWithCurrency(
					bddiscountprice, currency);

			double arith = fdiscountprice / fprodprice;
			double fsdiscount = 100 - arith * 100;

			Float percentagediscount = new Float(fsdiscount);

			savediscount = String.valueOf(percentagediscount.intValue());

		}

		StringBuffer p = new StringBuffer();
		p.append("<div class='product-price'>");
		if (discountprice == null) {
			p.append("<div class='product-price-price' style='width:50%;float:left;'>");
			p.append(regularprice);
			if (!StringUtils.isBlank(suffix)) {
				p.append(suffix).append(" ");
			}
			p.append("</div>");
			p.append("<div class='product-line'>&nbsp;</div>");
		} else {
			p.append("<div style='width:50%;float:left;'>");
			p.append("<strike>").append(regularprice);
			if (!StringUtils.isBlank(suffix)) {
				p.append(suffix).append(" ");
			}
			p.append("</strike>");
			p.append("</div>");
			p.append("<div style='width:50%;float:right;'>");
			p.append("<font color='red'>").append(discountprice);
			if (!StringUtils.isBlank(suffix)) {
				p.append(suffix).append(" ");
			}
			p.append("</font>").append("<br>").append(
					"<font color='red' style='font-size:75%;'>").append(
					LabelUtil.getInstance().getText(locale,
							"label.generic.save")).append(": ").append(
					savediscount).append(
					LabelUtil.getInstance().getText(locale,
							"label.generic.percentsign")).append(" ").append(
					LabelUtil.getInstance()
							.getText(locale, "label.generic.off")).append(
					"</font>");

			if (showDiscountDate && spenddate != null) {
				p.append("<br>").append(" <font style='font-size:65%;'>")
						.append(
								LabelUtil.getInstance().getText(locale,
										"label.generic.until"))
						.append("&nbsp;")
						.append(DateUtil.formatDate(spenddate)).append(
								"</font>");
			}

			p.append("</div>").toString();
		}
		p.append("</div>");
		return p.toString();

	}

	public static String formatHTMLProductPrice(Locale locale, String currency,
			Product view, boolean showDiscountDate, boolean shortDiscountFormat) {

		if (currency == null) {
			log.error("Currency is null ...");
			return "-N/A-";
		}

		int decimalPlace = 2;

		String prefix = "";
		String suffix = "";

		Map currenciesmap = RefCache.getCurrenciesListWithCodes();
		Currency c = (Currency) currenciesmap.get(currency);

		// regular price
		BigDecimal bdprodprice = view.getProductPrice();

		Date dt = new Date();

		// discount price
		java.util.Date spdate = null;
		java.util.Date spenddate = null;
		BigDecimal bddiscountprice = null;
		Special special = view.getSpecial();
		if (special != null) {
			spdate = special.getSpecialDateAvailable();
			spenddate = special.getExpiresDate();
			if (spdate.before(new Date(dt.getTime()))
					&& spenddate.after(new Date(dt.getTime()))) {
				bddiscountprice = special.getSpecialNewProductPrice();
			}
		}

		// all other prices
		Set prices = view.getPrices();
		if (prices != null) {
			Iterator pit = prices.iterator();
			while (pit.hasNext()) {
				ProductPrice pprice = (ProductPrice) pit.next();
				if (pprice.isDefaultPrice()) {
					pprice.setLocale(locale);
					suffix = pprice.getPriceSuffix();
					bddiscountprice = null;
					spdate = null;
					spenddate = null;
					bdprodprice = pprice.getProductPriceAmount();
					ProductPriceSpecial ppspecial = pprice.getSpecial();
					if (ppspecial != null) {
						if (ppspecial.getProductPriceSpecialStartDate() != null
								&& ppspecial.getProductPriceSpecialEndDate() != null) {
							spdate = ppspecial
									.getProductPriceSpecialStartDate();
							spenddate = ppspecial
									.getProductPriceSpecialEndDate();
						}
						bddiscountprice = ppspecial
								.getProductPriceSpecialAmount();
					}
					break;
				}
			}
		}

		double fprodprice = 0;
		;
		if (bdprodprice != null) {
			fprodprice = bdprodprice.setScale(decimalPlace,
					BigDecimal.ROUND_HALF_UP).doubleValue();
		}

		// regular price String
		String regularprice = CurrencyUtil
				.displayFormatedCssAmountWithCurrency(bdprodprice, currency);

		// discount price String
		String discountprice = null;
		String savediscount = null;

		if (bddiscountprice != null
				&& (spdate != null && spdate.before(new Date(dt.getTime())) && spenddate
						.after(new Date(dt.getTime())))) {

			double fdiscountprice = bddiscountprice.setScale(decimalPlace,
					BigDecimal.ROUND_HALF_UP).doubleValue();

			discountprice = CurrencyUtil.displayFormatedAmountWithCurrency(
					bddiscountprice, currency);

			double arith = fdiscountprice / fprodprice;
			double fsdiscount = 100 - arith * 100;

			Float percentagediscount = new Float(fsdiscount);

			savediscount = String.valueOf(percentagediscount.intValue());

		}

		StringBuffer p = new StringBuffer();
		p.append("<div class='product-price'>");
		if (discountprice == null) {
			p.append("<div class='product-price-price' style='width:50%;float:left;'>");
			p.append(regularprice);
			if (!StringUtils.isBlank(suffix)) {
				p.append(suffix).append(" ");
			}
			p.append("</div>");
			p.append("<div class='product-line'>&nbsp;</div>");
		} else {
			p.append("<div style='width:50%;float:left;'>");
			p.append("<strike>").append(regularprice);
			if (!StringUtils.isBlank(suffix)) {
				p.append(suffix).append(" ");
			}
			p.append("</strike>");
			p.append("</div>");
			p.append("<div style='width:50%;float:right;'>");
			p.append("<font color='red'>").append(discountprice);
			if (!StringUtils.isBlank(suffix)) {
				p.append(suffix).append(" ");
			}
			p.append("</font>");
			if(!shortDiscountFormat) {
					p.append("<br>").append(
					"<font color='red' style='font-size:75%;'>").append(
					LabelUtil.getInstance().getText(locale,
							"label.generic.save")).append(": ").append(
					savediscount).append(
					LabelUtil.getInstance().getText(locale,
							"label.generic.percentsign")).append(" ").append(
					LabelUtil.getInstance()
							.getText(locale, "label.generic.off")).append(
					"</font>");
			}

			if (showDiscountDate && spenddate != null) {
				p.append("<br>").append(" <font style='font-size:65%;'>")
						.append(
								LabelUtil.getInstance().getText(locale,
										"label.generic.until"))
						.append("&nbsp;")
						.append(DateUtil.formatDate(spenddate)).append(
								"</font>");
			}

			p.append("</div>").toString();
		}
		p.append("</div>");
		return p.toString();

	}

	public static BigDecimal determinePriceNoDiscount(Product product,
			Locale locale, String currency) {

		BigDecimal price = product.getProductPrice();

		// all other prices
		Set prices = product.getPrices();
		if (prices != null) {
			Iterator pit = prices.iterator();
			while (pit.hasNext()) {
				ProductPrice pprice = (ProductPrice) pit.next();
				pprice.setLocale(locale);

				if (pprice.isDefaultPrice()) {// overwrites default price
					price = pprice.getProductPriceAmount();
				}
			}
		}

		return price;

	}

	public static BigDecimal determinePriceNoDiscountWithAttributes(
			Product product, Collection<Long> attributes, Locale locale,
			String currency) {

		BigDecimal price = product.getProductPrice();

		// all other prices
		Set prices = product.getPrices();
		if (prices != null) {
			Iterator pit = prices.iterator();
			while (pit.hasNext()) {
				ProductPrice pprice = (ProductPrice) pit.next();
				pprice.setLocale(locale);

				if (pprice.isDefaultPrice()) {// overwrites default price
					price = pprice.getProductPriceAmount();
				}
			}
		}

		BigDecimal attributesPrice = null;

		if (attributes != null) {
			Iterator i = attributes.iterator();

			while (i.hasNext()) {
				ProductAttribute attr = (ProductAttribute) i.next();
				if (!attr.isAttributeDisplayOnly()
						&& attr.getOptionValuePrice().longValue() > 0) {
					if (attributesPrice == null) {
						attributesPrice = new BigDecimal(0);
					}
					attributesPrice = attributesPrice.add(attr
							.getOptionValuePrice());
				}
			}
		}

		if (attributesPrice != null) {
			price = price.add(attributesPrice);
		}

		return price;

	}

	public static BigDecimal determinePriceWithAttributes(Product product,
			Collection attributes, Locale locale, String currency) {

		int decimalPlace = 2;

		Map currenciesmap = RefCache.getCurrenciesListWithCodes();
		Currency c = (Currency) currenciesmap.get(currency);

		// prices
		BigDecimal bdprodprice = product.getProductPrice();
		BigDecimal bddiscountprice = null;

		// discount price
		Special special = product.getSpecial();

		Date dt = new Date();

		java.util.Date spdate = null;
		java.util.Date spenddate = null;

		if (special != null) {
			spdate = special.getSpecialDateAvailable();
			spenddate = special.getExpiresDate();
			if (spdate.before(new Date(dt.getTime()))
					&& spenddate.after(new Date(dt.getTime()))) {
				bddiscountprice = special.getSpecialNewProductPrice();
			}
		}

		// all other prices
		Set prices = product.getPrices();
		if (prices != null) {
			Iterator pit = prices.iterator();
			while (pit.hasNext()) {
				ProductPrice pprice = (ProductPrice) pit.next();
				pprice.setLocale(locale);

				if (pprice.isDefaultPrice()) {// overwrites default price
					bddiscountprice = null;
					spdate = null;
					spenddate = null;
					bdprodprice = pprice.getProductPriceAmount();
					ProductPriceSpecial ppspecial = pprice.getSpecial();
					if (ppspecial != null) {
						if (ppspecial.getProductPriceSpecialStartDate() != null
								&& ppspecial.getProductPriceSpecialEndDate() != null

								&& ppspecial.getProductPriceSpecialStartDate()
										.before(new Date(dt.getTime()))
								&& ppspecial.getProductPriceSpecialEndDate()
										.after(new Date(dt.getTime()))) {

							bddiscountprice = ppspecial
									.getProductPriceSpecialAmount();

						} else if (ppspecial
								.getProductPriceSpecialDurationDays() > -1) {

							bddiscountprice = ppspecial
									.getProductPriceSpecialAmount();
						}
					}
					break;
				}
			}
		}

		double fprodprice = 0;
		;
		if (bdprodprice != null) {
			fprodprice = bdprodprice.setScale(decimalPlace,
					BigDecimal.ROUND_HALF_UP).doubleValue();
		}

		// determine any properties prices
		BigDecimal attributesPrice = null;

		if (attributes != null) {
			Iterator i = attributes.iterator();

			while (i.hasNext()) {
				ProductAttribute attr = (ProductAttribute) i.next();
				if (!attr.isAttributeDisplayOnly()
						&& attr.getOptionValuePrice().longValue() > 0) {
					if (attributesPrice == null) {
						attributesPrice = new BigDecimal(0);
					}
					attributesPrice = attributesPrice.add(attr
							.getOptionValuePrice());
				}
			}
		}

		if (bddiscountprice != null) {

			if (attributesPrice != null) {
				bddiscountprice = bddiscountprice.add(attributesPrice);
			}

			return bddiscountprice;

		} else {

			if (attributesPrice != null) {
				bdprodprice = bdprodprice.add(attributesPrice);
			}

			return bdprodprice;
		}

	}

	public static BigDecimal determinePrice(Product product, Locale locale,
			String currency) {

		int decimalPlace = 2;

		Map currenciesmap = RefCache.getCurrenciesListWithCodes();
		Currency c = (Currency) currenciesmap.get(currency);

		// prices
		BigDecimal bdprodprice = product.getProductPrice();
		BigDecimal bddiscountprice = null;

		// discount price
		Special special = product.getSpecial();

		java.util.Date spdate = null;
		java.util.Date spenddate = null;

		if (special != null) {
			bddiscountprice = special.getSpecialNewProductPrice();
			spdate = special.getSpecialDateAvailable();
			spenddate = special.getExpiresDate();
		}

		Date dt = new Date();

		// all other prices
		Set prices = product.getPrices();
		if (prices != null) {
			Iterator pit = prices.iterator();
			while (pit.hasNext()) {
				ProductPrice pprice = (ProductPrice) pit.next();
				pprice.setLocale(locale);

				if (pprice.isDefaultPrice()) {// overwrites default price
					bddiscountprice = null;
					spdate = null;
					spenddate = null;
					bdprodprice = pprice.getProductPriceAmount();
					ProductPriceSpecial ppspecial = pprice.getSpecial();
					if (ppspecial != null) {
						if (ppspecial.getProductPriceSpecialStartDate() != null
								&& ppspecial.getProductPriceSpecialEndDate() != null

								&& ppspecial.getProductPriceSpecialStartDate()
										.before(new Date(dt.getTime()))
								&& ppspecial.getProductPriceSpecialEndDate()
										.after(new Date(dt.getTime()))) {

							bddiscountprice = ppspecial
									.getProductPriceSpecialAmount();

						} else if (ppspecial
								.getProductPriceSpecialDurationDays() > -1) {

							bddiscountprice = ppspecial
									.getProductPriceSpecialAmount();

						}
					}
					break;
				}
			}
		}

		double fprodprice = 0;

		if (bdprodprice != null) {
			fprodprice = bdprodprice.setScale(decimalPlace,
					BigDecimal.ROUND_HALF_UP).doubleValue();
		}

		if (bddiscountprice != null) {

			return bddiscountprice;

		} else {
			return bdprodprice;
		}

	}

	public static Date getDiscountEndDate(ProductPrice productPrice) {
		Date dt = new Date();
		BigDecimal price = productPrice.getProductPriceAmount();
		ProductPriceSpecial ppspecial = productPrice.getSpecial();
		if (ppspecial != null) {
			// this type of discount supercedes
			if (ppspecial.getProductPriceSpecialStartDate() != null
					&& ppspecial.getProductPriceSpecialEndDate() != null) {
				if (ppspecial.getProductPriceSpecialStartDate().before(
						new Date(dt.getTime()))
						&& ppspecial.getProductPriceSpecialEndDate().after(
								new Date(dt.getTime()))) {

					return ppspecial.getProductPriceSpecialEndDate();

				} else if (ppspecial.getProductPriceSpecialDurationDays() > -1) {

					Date startDate = ppspecial
							.getProductPriceSpecialStartDate();

					int numDays = ppspecial
							.getProductPriceSpecialDurationDays();
					Date purchased = new Date();
					Calendar c = Calendar.getInstance();
					c.setTime(dt);
					c.add(Calendar.DATE, numDays);

					if (dt.before(c.getTime())
							&& ppspecial.getProductPriceSpecialAmount()
									.floatValue() < productPrice
									.getProductPriceAmount().floatValue()) {
						return c.getTime();
					} else {
						return null;
					}
				}

			}
		}
		return null;
	}
	
	
	/**
	 * Detects a discount on base price
	 * @param product
	 * @return
	 */
	public static boolean hasDiscount(Product product) {
		


		// discount price
		java.util.Date spdate = null;
		java.util.Date spenddate = null;
		BigDecimal bddiscountprice = null;
		Special special = product.getSpecial();
		
		
		boolean hasDiscount = false;
		
		if (special != null) {
			bddiscountprice = special.getSpecialNewProductPrice();
			spdate = special.getSpecialDateAvailable();
			spenddate = special.getExpiresDate();
			
			if (special.getSpecialDateAvailable() != null
					&& special.getExpiresDate() != null) {
				if (special.getSpecialDateAvailable().before(
						new Date())
						&& special.getExpiresDate().after(
								new Date())) {
					
					hasDiscount = true;

				} 
			}
			
			if(product.getPrices()!=null && product.getPrices().size()>0) {
				
				for(Object o: product.getPrices()) {
					
					ProductPrice pp = (ProductPrice)o;
					if(pp.isDefaultPrice()) {
						hasDiscount = false;
						hasDiscount = hasDiscount(pp);
						if(hasDiscount) {
							break;
						}
					}
				}
			} 
		}
		
		return hasDiscount;
		
		
	}

	public static boolean hasDiscount(ProductPrice productPrice) {
		Date dt = new Date();
		BigDecimal price = productPrice.getProductPriceAmount();
		ProductPriceSpecial ppspecial = productPrice.getSpecial();
		if (ppspecial != null) {
			// this type of discount supercedes
			if (ppspecial.getProductPriceSpecialStartDate() != null
					&& ppspecial.getProductPriceSpecialEndDate() != null) {
				if (ppspecial.getProductPriceSpecialStartDate().before(
						new Date(dt.getTime()))
						&& ppspecial.getProductPriceSpecialEndDate().after(
								new Date(dt.getTime()))) {

					return true;

				} else if (ppspecial.getProductPriceSpecialDurationDays() > -1) {

					Date startDate = ppspecial
							.getProductPriceSpecialStartDate();

					int numDays = ppspecial
							.getProductPriceSpecialDurationDays();
					Date purchased = new Date();
					Calendar c = Calendar.getInstance();
					c.setTime(dt);
					c.add(Calendar.DATE, numDays);
					// if(dt.before(c.getTime())) {

					if (dt.before(c.getTime())
							&& ppspecial.getProductPriceSpecialAmount()
									.floatValue() < productPrice
									.getProductPriceAmount().floatValue()) {
						return true;
					} else {
						return false;
					}
				}

			}
		}
		return false;
	}

	public static BigDecimal determinePrice(ProductPrice productPrice) {
		Date dt = new Date();
		BigDecimal price = productPrice.getProductPriceAmount();
		ProductPriceSpecial ppspecial = productPrice.getSpecial();
		if (ppspecial != null) {
			// this type of discount supercedes
			if (ppspecial.getProductPriceSpecialStartDate() != null
					&& ppspecial.getProductPriceSpecialEndDate() != null) {
				if (ppspecial.getProductPriceSpecialStartDate().before(
						new Date(dt.getTime()))
						&& ppspecial.getProductPriceSpecialEndDate().after(
								new Date(dt.getTime()))) {

					price = ppspecial.getProductPriceSpecialAmount();

				} else if (ppspecial.getProductPriceSpecialDurationDays() > -1) {

					Date startDate = ppspecial
							.getProductPriceSpecialStartDate();

					int numDays = ppspecial
							.getProductPriceSpecialDurationDays();
					Date purchased = new Date();
					Calendar c = Calendar.getInstance();
					c.setTime(dt);
					c.add(Calendar.DATE, numDays);
					// if(dt.before(c.getTime())) {

					if (dt.before(c.getTime())
							&& ppspecial.getProductPriceSpecialAmount()
									.floatValue() < productPrice
									.getProductPriceAmount().floatValue()) {
						price = ppspecial.getProductPriceSpecialAmount();
					} else {
						price = productPrice.getProductPriceAmount();
					}

				}

			}
		}

		return price;
	}

	public static BigDecimal determinePrice(OrderProductPrice productPrice) {
		Date dt = new Date();
		BigDecimal price = productPrice.getProductPriceAmount();
		ProductPriceSpecial ppspecial = productPrice.getSpecial();
		if (ppspecial != null) {
			// this type of discount supercedes
			if (ppspecial.getProductPriceSpecialStartDate() != null
					&& ppspecial.getProductPriceSpecialEndDate() != null) {
				if (ppspecial.getProductPriceSpecialStartDate().before(
						new Date(dt.getTime()))
						&& ppspecial.getProductPriceSpecialEndDate().after(
								new Date(dt.getTime()))) {

					price = ppspecial.getProductPriceSpecialAmount();

				} else if (ppspecial.getProductPriceSpecialDurationDays() > -1) {

					price = ppspecial.getProductPriceSpecialAmount();

				}

			}

		}

		return price;

	}



}



```
