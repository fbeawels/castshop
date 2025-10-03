# CheckoutUtil.java

## Review

## 1. Summary  

**Purpose**  
`CheckoutUtil` is a helper class that transforms catalog `Product` objects into order‑ready `OrderProduct` objects, calculates prices (including attribute overrides), and builds human‑readable attribute strings. It is heavily used during the shopping‑cart/checkout workflow of the SalesManager core module.

**Key Components**  

| Component | Role |
|-----------|------|
| `createOrderProduct(long,Locale,String)` | Pulls a product from the catalog, populates an `OrderProduct` with all static data (price, tax, descriptions, images, etc.) and prepares it for the checkout phase. |
| `addAttributesToProduct(...)` | Applies a list of `OrderProductAttribute` instances to an already‑created `OrderProduct`, adjusting the price and generating an attributes line. |
| `addAttributesFromRawObjects(...)` (2 overloads) | Similar to the above, but starts from raw `OrderProductAttribute` objects that were created by the front‑end JavaScript layer. It looks up the corresponding catalog attributes by `productId`. |
| `getAttributesLine(OrderProduct)` | Returns a formatted string describing the selected attributes of a product. |
| `buildAttributesLine(...)` | Placeholder – currently returns `null`. Intended for building the attributes string from a collection of attributes. |

**Design Patterns & Libraries**  
* **Factory** – `ServiceFactory` is used to obtain a `CatalogService`.  
* **DAO‑style Service Layer** – the class is a pure utility and does not manage transactions or persistence.  
* **Apache Commons Lang** – `StringUtils` for blank checks.  
* **BigDecimal** – used for currency calculations (with some inconsistent handling).  

The class is a classic *static‑utility* with no state, but it contains large amounts of duplicated logic across the three attribute‑handling methods.

---

## 2. Detailed Description  

### 2.1 Execution Flow  

1. **Product Retrieval** – `createOrderProduct` fetches a `Product` via `CatalogService.getProduct(productId)`.  
2. **Static Data Copy** – All read‑only product data (descriptions, prices, images, tax class, subscription flags, etc.) are copied into a fresh `OrderProduct`.  
3. **Price Determination** – `ProductUtil.determinePriceNoDiscount()` calculates the base price; then the method enriches the `OrderProduct` with price metadata (`priceText`, `priceFormated`, etc.).  
4. **Attribute Handling** –  
   * `addAttributesToProduct` (or the *raw* variants) iterates over a supplied list of `OrderProductAttribute` objects.  
   * For each attribute it looks up the catalog `ProductAttribute`, resolves the final price, and builds an attributes line (`[ Option → Value, … ]`).  
   * The sum of all attribute prices is added to the product’s base price, and the `finalPrice` (base + attributes) is recomputed.  
5. **Result** – The returned `OrderProduct` is ready to be stored in the session/cart or persisted as part of an `Order`.

### 2.2 Architecture & Design Choices  

* **No State** – All methods are static and stateless. This simplifies usage but forces the class to repeat a lot of lookup logic.  
* **Eager Loading** – The utility eagerly pulls all related data (descriptions, prices, attributes) in one call, which is convenient but can be expensive if the product has many attributes or prices.  
* **Hard‑coded Language Mapping** – The class uses `LanguageUtil.getLanguageNumberCode()` to match language IDs. This coupling makes the method brittle if the language mapping changes.  
* **Session Coupling** – The *raw* attribute methods use `HttpServletRequest` and `SessionUtil` to retrieve the cart, tying the utility to the web tier.  
* **Duplicate Code** – Almost identical blocks of code exist in all three attribute‑applying methods. Extracting a common helper would reduce maintenance overhead.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `createOrderProduct(long productId, Locale locale, String currency)` | Builds a new `OrderProduct` from a catalog `Product`. | `productId` – product PK; `locale` – current language; `currency` – currency code. | `OrderProduct` fully populated. | Throws `Exception` if product lookup fails. |
| `getAttributesLine(OrderProduct product)` | Formats the product’s selected attributes into a readable string. | `OrderProduct`. | `String` like `[ Color → Red, Size → L ]` or `null`. | None. |
| `addAttributesToProduct(List<OrderProductAttribute> attributes, OrderProduct product, String currency, Locale locale)` | Adds attributes to an existing `OrderProduct`, adjusts price, and generates an attributes line. | `attributes` – list from UI; `product` – pre‑created `OrderProduct`; `currency`, `locale`. | Updated `OrderProduct`. | None. |
| `buildAttributesLine(Collection<OrderProductAttribute> attributes, OrderProduct orderProduct)` | **Placeholder** – intended to create an attributes string but currently returns `null`. | `attributes`, `orderProduct`. | `String` (currently `null`). | None. |
| `addAttributesFromRawObjects(List<OrderProductAttribute> attributes, long productId, String lineId, String currency, HttpServletRequest request)` | Applies attributes that came from raw JS objects, using `lineId` to locate the `OrderProduct` in the session cart. | `attributes`, `productId`, `lineId`, `currency`, `request`. | Updated `OrderProduct` (if found). | Updates session cart entry. |
| `addAttributesFromRawObjects(List<OrderProductAttribute> attributes, OrderProduct scp, String currency, HttpServletRequest request)` | Similar to the above but starts from an `OrderProduct` supplied directly. | `attributes`, `scp`, `currency`, `request`. | Updated `OrderProduct`. | None. |

### Common Patterns  

* **Attribute Lookup** – All three “add” methods create a `Map<Long,ProductAttribute>` keyed by `optionValueId`.  
* **Price Aggregation** – They iterate over `attributes`, resolve the final attribute price (`attrPrice`), and accumulate `sumPrice`.  
* **Attributes Line Construction** – The `[ Option → Value ]` string is built inside a `StringBuffer` (`StringBuilder` would be preferable).  
* **Price Adjustment** – `productPrice = base + sumPrice`; `finalPrice = productPrice * quantity`.  

### 3.1 Notable Implementation Details  

* **BigDecimal usage** –  
  * `addAttributesToProduct` sets `sumPrice` with `new BigDecimal(attrPrice.doubleValue()).setScale(2)` (unnecessary conversion).  
  * The other two overloads use `setScale(2)` explicitly.  
  * The class should consistently use `BigDecimal.valueOf()` or a constant like `BigDecimal.ZERO` to avoid floating‑point loss.  
* **StringUtils.isBlank()** – Used to normalise empty attribute values.  
* **Language mapping** – The methods assume that the `ProductDescription`’s language ID is comparable to `locale.getLanguage()` via `LanguageUtil`.  

---

## 4. Dependencies  

| External Class | Role |
|-----------------|------|
| `ServiceFactory` | Obtains a `CatalogService`. |
| `CatalogService` | DAO‑like service providing catalog data (products, prices, attributes). |
| `ProductUtil` | Calculates the base price of a product. |
| `ProductSpecial` | Represents product special dates (availability / expiration). |
| `CurrencyUtil` | (Not imported, but referenced via `CurrencyUtil.validateCurrency` in the raw attribute methods). |
| `StringUtils` (Apache Commons Lang) | Blank checks. |
| `HttpServletRequest`, `HttpServletResponse` | Access to the web request/session. |
| `SessionUtil` | Retrieves the shopping‑cart map from the HTTP session. |
| `LanguageUtil` | Maps language strings to numeric IDs. |

The utility is tightly coupled to the SalesManager service layer and the web tier. All imports use raw types (`Collection`, `Iterator`, `Map` etc.), which makes the code hard to read and maintain.

---

## 4. Additional Notes & Recommendations  

### 4.1 Code Quality / Maintainability  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw collections & iterators** | Prone to `ClassCastException`, harder to understand, no compile‑time type safety. | Replace all raw types with generics (`Collection<ProductAttribute>`, `Iterator<ProductAttribute>`). |
| **Large code duplication** | Bug‑prone; any change must be replicated across three methods. | Extract a private helper (`applyAttributes(...)`) that accepts a `List<OrderProductAttribute>` and the target `OrderProduct`. |
| **Inconsistent BigDecimal handling** | Possible precision loss when converting to `double`. | Use `BigDecimal.valueOf(double)` or keep the original `BigDecimal` unchanged. Never convert to `double`. |
| **Missing implementation** (`buildAttributesLine`) | Unused method may confuse callers. | Implement or remove it. |
| **Hard‑coded language mapping** | Coupling to `LanguageUtil`; brittle if language tables change. | Pass `languageId` directly, or use a locale‑aware service to retrieve the correct description. |
| **Session/Web tier coupling** | Utility is no longer purely business‑logic; unit‑testing is harder. | Move session‑access logic into a separate `CartService` or make the utility accept the cart map as a parameter. |
| **Logging & Exception handling** | No diagnostic information when something goes wrong. | Add `logger` statements, wrap low‑level exceptions into a custom `CheckoutException`. |
| **Potential NPEs** | *createOrderProduct* sets download fields twice; *addAttributesFromRawObjects* may return `null` Map, leading to `NullPointerException`. | Add defensive null checks; throw a clear exception if the cart entry is missing. |
| **Price recalculation logic** | `addAttributesToProduct` multiplies `productPrice` by `quantity` after adding attributes, but ignores attribute quantity (attributes are normally per item). | Clarify intent; if attributes can be per quantity, multiply `sumPrice` by quantity before adding. |
| **Duplicate attribute values** | `Set` may contain duplicate attributes if the list contains duplicates. | Validate uniqueness or rely on the UI to prevent duplicates. |
| **Use of `new BigDecimal(double)`** | Rounding errors. | Prefer `new BigDecimal(String)` or `BigDecimal.valueOf(long)` constants. |
| **Hard‑coded property names** – e.g., `setDownloadMaxdays` twice. | Minor bug but can mislead. | Use a dedicated setter for the second call (e.g., `setDownloadCount`). |

### 4.2 Performance & Scalability  

* The utility performs many round‑trips to the `CatalogService` in a single method (e.g., fetching all attributes for a product). If a product has dozens of attributes, the overhead can become noticeable, especially in a high‑traffic store.  
* Session‑based cart storage (`SessionUtil.getOrderProducts`) is fine for single‑user scenarios but can cause contention in a clustered environment.  
* BigDecimal operations are inexpensive for typical product prices, but the repeated conversion to `double` and back introduces unnecessary cost and precision loss.  

### 4.3 Security  

No user input is validated beyond `StringUtils.isBlank()`. The methods rely on the correctness of the `CatalogService`. If the front‑end can send arbitrary `productId` or `lineId` values, ensure that the session cart cannot be tampered with (e.g., by checking that the `productId` actually belongs to the `OrderProduct` retrieved from the session).

### 4.4 Testing  

* **Unit tests** are currently difficult to write because the methods are static and perform live lookups.  
* **Integration tests** should verify that `createOrderProduct` copies all fields correctly and that attribute price overrides are applied.  
* Mocking the `CatalogService` and `ProductUtil` would allow isolated tests for the price‑adjustment logic.  

---

## 4. Summary of Suggested Improvements  

| Area | Recommendation |
|------|----------------|
| **Generics** | Replace raw collections/iterators with typed generics (`List<OrderProductAttribute>`, `Map<String,OrderProduct>`). |
| **Code Reuse** | Extract a private helper that accepts `List<OrderProductAttribute>`, `OrderProduct`, `currency`, and `locale` to perform the attribute lookup, price aggregation, and attributes line construction. |
| **BigDecimal handling** | Use constants (`BigDecimal.ZERO`, `BigDecimal.ONE`) and `setScale(2, RoundingMode.HALF_EVEN)` consistently. Avoid converting to `double`. |
| **Language handling** | Pass language ID directly, or use a service that returns the proper description map. |
| **Exception Strategy** | Create a custom `CheckoutException` instead of throwing generic `Exception`. Log the root cause. |
| **Session independence** | Move session‑related logic into a dedicated cart/service layer; let `CheckoutUtil` operate purely on POJOs. |
| **`buildAttributesLine`** | Either implement it or remove it to avoid confusion. |
| **Unit‑testability** | Introduce instance methods that accept services as constructor parameters, or use dependency injection, so tests can mock the catalog service without hitting the database. |
| **Documentation** | Add JavaDoc to every public method, especially describing the expected format of the attributes line. |
| **Coding style** | Adopt `StringBuilder`, `String.join`, or streams for building the attributes string. |
| **Logging** | Log failures of attribute lookups or missing data. |

By addressing these items the class will become safer, easier to maintain, and more testable while still delivering the same checkout‑ready objects that the SalesManager core relies on.

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
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttribute;
import com.salesmanager.core.entity.catalog.ProductAttributeDownload;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.catalog.ProductOption;
import com.salesmanager.core.entity.catalog.ProductOptionDescription;
import com.salesmanager.core.entity.catalog.ProductOptionValue;
import com.salesmanager.core.entity.catalog.ProductOptionValueDescription;
import com.salesmanager.core.entity.catalog.ProductPrice;
import com.salesmanager.core.entity.catalog.ProductPriceDescription;
import com.salesmanager.core.entity.catalog.ProductPriceSpecial;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductAttribute;
import com.salesmanager.core.entity.orders.OrderProductDownload;
import com.salesmanager.core.entity.orders.OrderProductPrice;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.www.SessionUtil;

public class CheckoutUtil {

	/**
	 * Creates an order product (OrderProduct) from a Product entity. An
	 * OrderProduct is the entity used during checkout and persisted with the
	 * order
	 * 
	 * @param productId
	 * @param locale
	 * @param currency
	 * @return
	 * @throws Exception
	 */
	public static OrderProduct createOrderProduct(long productId,
			Locale locale, String currency) throws Exception {

		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);
		Product p = cservice.getProduct(productId);

		OrderProduct scp = new OrderProduct();

		if (p != null) {

			scp.setMerchantId(p.getMerchantId());

			// check if attributes exist
			Collection attrs = cservice.getProductAttributes(p.getProductId(),
					locale.getLanguage());

			boolean hasPricedAttributes = false;
			if (attrs != null && attrs.size() > 0) {
				Iterator i = attrs.iterator();
				while (i.hasNext()) {
					ProductAttribute pa = (ProductAttribute) i.next();
					if (!pa.isAttributeDisplayOnly()) {
						hasPricedAttributes = true;
					}
				}
			}

			if (hasPricedAttributes) {
				scp.setAttributes(true);
			}

			scp.setProductName("");
			// name
			Set descriptions = p.getDescriptions();
			if (descriptions != null) {
				Iterator i = descriptions.iterator();
				while (i.hasNext()) {
					ProductDescription pd = (ProductDescription) i.next();
					if (pd.getId().getLanguageId() == LanguageUtil
							.getLanguageNumberCode(locale.getLanguage())) {
						scp.setProductName(pd.getProductName());
						scp.setProductDescription(pd.getProductDescription());
						break;
					}
				}
			}

			scp.setProductId(p.getProductId());
			scp.setTaxClassId(p.getProductTaxClassId());

			// get download
			ProductAttributeDownload download = cservice.getProductDownload(p
					.getProductId());

			if (download != null) {
				OrderProductDownload opd = new OrderProductDownload();
				opd.setDownloadCount(0);
				opd.setDownloadMaxdays(download.getProductAttributeMaxdays());
				opd.setDownloadMaxdays(download.getProductAttributeMaxdays());
				opd.setOrderProductFilename(download
						.getProductAttributeFilename());
				opd.setFileId(download.getProductAttributeId());
				Set downloads = new HashSet();
				downloads.add(opd);
				scp.setDownloads(downloads);
			}

			// now determine price
			BigDecimal price = ProductUtil.determinePriceNoDiscount(p, locale,
					currency);

			scp.setFinalPrice(price);
			scp.setProductPrice(price);
			scp.setCurrency(currency);
			scp.setProductQuantityOrderMax(p.getProductQuantityOrderMax());
			if (p.getSpecial() != null) {
				scp.setProductSpecialNewPrice(p.getSpecial()
						.getSpecialNewProductPrice());
				scp.setProductSpecialDateAvailable(p.getSpecial()
						.getSpecialDateAvailable());
				scp
						.setProductSpecialDateExpire(p.getSpecial()
								.getExpiresDate());
			}
			scp.setPriceText(CurrencyUtil.displayFormatedAmountNoCurrency(
					price, currency));
			scp.setPriceFormated(CurrencyUtil
					.displayFormatedAmountWithCurrency(price, currency));

			// original price
			scp.setOriginalProductPrice(price);

			scp.setProductImage(p.getProductImage());
			scp.setProductType(p.getProductType());
			scp.setProductVirtual(p.isProductVirtual());
			scp.setProductWidth(p.getProductWidth());
			scp.setProductWeight(p.getProductWeight());
			scp.setProductHeight(p.getProductHeight());
			scp.setProductLength(p.getProductLength());
			scp.setProductId(p.getProductId());

			Set pricesSet = p.getPrices();
			if (pricesSet != null) {
				Set productSet = new HashSet();
				Iterator i = pricesSet.iterator();
				while (i.hasNext()) {
					ProductPrice pp = (ProductPrice) i.next();

					ProductPriceSpecial pps = pp.getSpecial();
					if (pps != null) {
						pps.setOriginalPriceAmount(pp.getProductPriceAmount());
					}
					OrderProductPrice opp = new OrderProductPrice();
					opp.setDefaultPrice(pp.isDefaultPrice());
					opp.setProductPriceAmount(pp.getProductPriceAmount());
					opp.setProductPriceModuleName(pp
							.getProductPriceModuleName());
					opp.setProductPriceTypeId(pp.getProductPriceTypeId());
					opp.setSpecial(pps);

					Set priceDescriptions = pp.getPriceDescriptions();
					if (priceDescriptions != null
							&& priceDescriptions.size() > 0) {
						String priceDescription = "";
						for (Object o : priceDescriptions) {
							ProductPriceDescription ppd = (ProductPriceDescription) o;
							if (ppd.getId().getLanguageId() == LanguageUtil
									.getLanguageNumberCode(locale.getLanguage())) {
								priceDescription = ppd.getProductPriceName();
								break;
							}
						}
						opp.setProductPriceName(priceDescription);
					}

					productSet.add(opp);
				}
				scp.setPrices(productSet);
			}

			if (cservice.isProductSubscribtion(p)) {
				scp.setProductSubscribtion(true);
			}

			if (!p.isProductVirtual()) {
				scp.setShipping(true);
			}

		}

		return scp;

	}

	public static String getAttributesLine(OrderProduct product) {

		Set attributes = product.getOrderattributes();

		if (attributes != null) {

			StringBuffer attributesLine = null;

			attributesLine = new StringBuffer();
			int count = 0;
			Iterator i = attributes.iterator();
			while (i.hasNext()) {
				OrderProductAttribute opa = (OrderProductAttribute) i.next();

				if (count == 0) {
					attributesLine.append("[ ");
				}
				attributesLine.append(opa.getProductOption()).append(" -> ")
						.append(opa.getProductOptionValue());
				if (count + 1 == attributes.size()) {
					attributesLine.append("]");
				} else {
					attributesLine.append(", ");
				}
				count++;
			}

			return attributesLine.toString();

		} else {
			return null;
		}

	}

	/**
	 * Add attributes to an OrderProduct and calculates the OrderProductPrice
	 * accordingly
	 * 
	 * @param attributes
	 * @param product
	 * @param currency
	 * @param locale
	 * @return
	 * @throws Exception
	 */
	public static OrderProduct addAttributesToProduct(
			List<OrderProductAttribute> attributes, OrderProduct product,
			String currency, Locale locale) throws Exception {

		Locale loc = locale;
		String lang = loc.getLanguage();

		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);

		BigDecimal sumPrice = null;

		// get attributes for this product
		Collection productAttributes = cservice.getProductAttributes(product
				.getProductId(), locale.getLanguage());
		Map mapAttributes = new HashMap();
		if (productAttributes != null) {

			Iterator i = productAttributes.iterator();
			while (i.hasNext()) {
				ProductAttribute p = (ProductAttribute) i.next();
				mapAttributes.put(p.getOptionValueId(), p);
			}
		}

		StringBuffer attributesLine = null;

		if (attributes != null) {
			attributesLine = new StringBuffer();
			int count = 0;
			Iterator i = attributes.iterator();
			while (i.hasNext()) {
				OrderProductAttribute opa = (OrderProductAttribute) i.next();
				String attrPriceText = opa.getPrice();
				BigDecimal attrPrice = null;
				if (attrPriceText != null) {
					attrPrice = CurrencyUtil.validateCurrency(attrPriceText,
							currency);
				} else {
					attrPrice = opa.getOptionValuePrice();
				}
				// get all information from the attribute
				ProductAttribute pa = (ProductAttribute) mapAttributes.get(opa
						.getProductOptionValueId());
				if (pa != null) {
					if (attrPrice == null) {
						attrPrice = pa.getOptionValuePrice();
					}
				}
				if (attrPrice != null) {
					opa.setOptionValuePrice(attrPrice);
					opa.setPrice(CurrencyUtil.displayFormatedAmountNoCurrency(
							attrPrice, currency));
					if (sumPrice == null) {
						// try {
						// sumPrice= new
						// BigDecimal(attrPrice.doubleValue()).setScale(BigDecimal.ROUND_UNNECESSARY);
						sumPrice = attrPrice;
						// } catch (Exception e) {
						// }

					} else {
						BigDecimal currentPrice = sumPrice;
						sumPrice = currentPrice.add(attrPrice);
					}
				}
				opa.setOrderProductId(product.getProductId());
				opa.setProductAttributeIsFree(pa.isProductAttributeIsFree());

				opa.setProductOption("");
				if (StringUtils.isBlank(opa.getProductOptionValue())) {
					opa.setProductOptionValue("");
				}

				ProductOption po = pa.getProductOption();
				Set poDescriptions = po.getDescriptions();
				if (poDescriptions != null) {
					Iterator pi = poDescriptions.iterator();
					while (pi.hasNext()) {
						ProductOptionDescription pod = (ProductOptionDescription) pi
								.next();
						if (pod.getId().getLanguageId() == LanguageUtil
								.getLanguageNumberCode(lang)) {
							opa.setProductOption(pod.getProductOptionName());
							break;
						}
					}
				}

				if (StringUtils.isBlank(opa.getProductOptionValue())) {
					ProductOptionValue pov = pa.getProductOptionValue();
					if (pov != null) {
						Set povDescriptions = pov.getDescriptions();
						if (povDescriptions != null) {
							Iterator povi = povDescriptions.iterator();
							while (povi.hasNext()) {
								ProductOptionValueDescription povd = (ProductOptionValueDescription) povi
										.next();
								if (povd.getId().getLanguageId() == LanguageUtil
										.getLanguageNumberCode(lang)) {
									opa.setProductOptionValue(povd
											.getProductOptionValueName());
									break;
								}
							}
						}
					}
				}
				opa.setProductAttributeWeight(pa.getProductAttributeWeight());
				if (count == 0) {
					attributesLine.append("[ ");
				}
				attributesLine.append(opa.getProductOption()).append(" -> ")
						.append(opa.getProductOptionValue());
				if (count + 1 == attributes.size()) {
					attributesLine.append("]");
				} else {
					attributesLine.append(", ");
				}
				count++;
			}
		}

		// add attribute price to productprice
		if (sumPrice != null) {

			// get product price
			BigDecimal productPrice = product.getProductPrice();
			productPrice = productPrice.add(sumPrice);

			// added
			product.setProductPrice(productPrice);

			BigDecimal finalPrice = productPrice.multiply(new BigDecimal(
					product.getProductQuantity()));

			product.setPriceText(CurrencyUtil.displayFormatedAmountNoCurrency(
					productPrice, currency));
			product.setPriceFormated(CurrencyUtil
					.displayFormatedAmountWithCurrency(finalPrice, currency));
		}

		if (attributesLine != null) {
			product.setAttributesLine(attributesLine.toString());
		}

		Set attributesSet = new HashSet(attributes);

		product.setOrderattributes(attributesSet);

		return product;
	}

	public String buildAttributesLine(
			Collection<OrderProductAttribute> attributes,
			OrderProduct orderProduct) {
		return null;
	}

	/**
	 * OrderProductAttribute is configured from javascript This code needs to
	 * invoke catalog objects because it requires getProductOptionValueId, will
	 * not change any price Add attributes to product, add attribute offset
	 * price to original product price
	 * 
	 * @param attributes
	 * @param lineId
	 * @param currency
	 * @param request
	 * @return
	 * @throws Exception
	 */
	public static OrderProduct addAttributesFromRawObjects(
			List<OrderProductAttribute> attributes, long productId,
			String lineId, String currency, HttpServletRequest request)
			throws Exception {

		Locale loc = request.getLocale();
		String lang = loc.getLanguage();

		HttpSession session = request.getSession();

		Map cartLines = SessionUtil.getOrderProducts(request);

		if (cartLines == null) {
			throw new Exception(
					"No OrderProduct exixt yet, cannot assign attributes");
		}

		OrderProduct scp = (OrderProduct) cartLines.get(lineId);

		if (scp == null) {
			throw new Exception("No OrderProduct exixt for lineId " + lineId);
		}

		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);

		BigDecimal sumPrice = null;

		// make sure OrderProduct and productId match
		if (scp.getProductId() == productId) {

			Locale locale = (Locale) request.getSession().getAttribute(
					"WW_TRANS_I18N_LOCALE");
			if (locale == null)
				locale = request.getLocale();

			// get attributes for this product
			Collection productAttributes = cservice.getProductAttributes(
					productId, locale.getLanguage());
			Map mapAttributes = new HashMap();
			if (productAttributes != null) {

				Iterator i = productAttributes.iterator();
				while (i.hasNext()) {
					ProductAttribute p = (ProductAttribute) i.next();
					mapAttributes.put(p.getOptionValueId(), p);
				}

			}

			if (scp != null) {

				StringBuffer attributesLine = null;

				if (attributes != null) {
					attributesLine = new StringBuffer();
					int count = 0;
					Iterator i = attributes.iterator();
					while (i.hasNext()) {
						OrderProductAttribute opa = (OrderProductAttribute) i
								.next();
						String attrPriceText = opa.getPrice();
						BigDecimal attrPrice = null;
						if (attrPriceText != null) {
							attrPrice = CurrencyUtil.validateCurrency(
									attrPriceText, currency);
						} else {
							attrPrice = opa.getOptionValuePrice();
						}
						// get all information from the attribute
						ProductAttribute pa = (ProductAttribute) mapAttributes
								.get(opa.getProductOptionValueId());
						if (pa != null) {
							if (attrPrice == null) {
								attrPrice = pa.getOptionValuePrice();
							}
						}
						if (attrPrice != null) {
							opa.setOptionValuePrice(attrPrice);
							opa.setPrice(CurrencyUtil
									.displayFormatedAmountNoCurrency(attrPrice,
											currency));
							if (sumPrice == null) {
								sumPrice = new BigDecimal(attrPrice
										.doubleValue()).setScale(2);
							} else {
								// double pr = sumPrice.doubleValue() +
								// attrPrice.doubleValue();
								// sumPrice = new
								// BigDecimal(sumPrice.doubleValue() +
								// attrPrice.doubleValue());
								BigDecimal currentPrice = sumPrice;
								sumPrice = currentPrice.add(attrPrice);
							}
						}
						// opa.setOrderId(Long.parseLong(orderId));
						opa.setOrderProductId(productId);
						opa.setProductAttributeIsFree(pa
								.isProductAttributeIsFree());

						opa.setProductOption("");
						if (StringUtils.isBlank(opa.getProductOptionValue())) {
							opa.setProductOptionValue("");
						}

						ProductOption po = pa.getProductOption();
						Set poDescriptions = po.getDescriptions();
						if (poDescriptions != null) {
							Iterator pi = poDescriptions.iterator();
							while (pi.hasNext()) {
								ProductOptionDescription pod = (ProductOptionDescription) pi
										.next();
								if (pod.getId().getLanguageId() == LanguageUtil
										.getLanguageNumberCode(lang)) {
									opa.setProductOption(pod
											.getProductOptionName());
									break;
								}
							}
						}

						if (StringUtils.isBlank(opa.getProductOptionValue())) {
							ProductOptionValue pov = pa.getProductOptionValue();
							if (pov != null) {
								Set povDescriptions = pov.getDescriptions();
								if (povDescriptions != null) {
									Iterator povi = povDescriptions.iterator();
									while (povi.hasNext()) {
										ProductOptionValueDescription povd = (ProductOptionValueDescription) povi
												.next();
										if (povd.getId().getLanguageId() == LanguageUtil
												.getLanguageNumberCode(lang)) {
											opa
													.setProductOptionValue(povd
															.getProductOptionValueName());
											break;
										}
									}
								}
							}
						}
						opa.setProductAttributeWeight(pa
								.getProductAttributeWeight());
						if (count == 0) {
							attributesLine.append("[ ");
						}
						attributesLine.append(opa.getProductOption()).append(
								" -> ").append(opa.getProductOptionValue());
						if (count + 1 == attributes.size()) {
							attributesLine.append("]");
						} else {
							attributesLine.append(", ");
						}
						count++;
					}
				}

				// add attribute price to productprice
				if (sumPrice != null) {
					// sumPrice = sumPrice.multiply(new
					// BigDecimal(scp.getProductQuantity()));

					scp.setAttributeAdditionalCost(sumPrice);// add additional
																// attribute
																// price

					// get product price
					BigDecimal productPrice = scp.getProductPrice();
					productPrice = productPrice.add(sumPrice);

					// added
					scp.setProductPrice(productPrice);

					// BigDecimal finalPrice = scp.getFinalPrice();
					BigDecimal finalPrice = productPrice
							.multiply(new BigDecimal(scp.getProductQuantity()));
					// finalPrice = finalPrice.add(sumPrice);
					// BigDecimal cost = scp.get

					scp.setPriceText(CurrencyUtil
							.displayFormatedAmountNoCurrency(productPrice,
									currency));
					scp.setPriceFormated(CurrencyUtil
							.displayFormatedAmountWithCurrency(finalPrice,
									currency));
				}

				if (attributesLine != null) {
					scp.setAttributesLine(attributesLine.toString());
				}

				Set attributesSet = new HashSet(attributes);

				scp.setOrderattributes(attributesSet);



			}

		}

		return scp;
	}

	/**
	 * OrderProductAttribute is configured from javascript This code needs to
	 * invoke catalog objects require getProductOptionValueId
	 * 
	 * @param attributes
	 * @param lineId
	 * @param currency
	 * @param request
	 * @return
	 * @throws Exception
	 */
	public static OrderProduct addAttributesFromRawObjects(
			List<OrderProductAttribute> attributes, OrderProduct scp,
			String currency, HttpServletRequest request) throws Exception {

		Locale loc = request.getLocale();
		String lang = loc.getLanguage();

		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);

		BigDecimal sumPrice = null;

		Locale locale = (Locale) request.getSession().getAttribute(
				"WW_TRANS_I18N_LOCALE");
		if (locale == null)
			locale = request.getLocale();

		// get attributes for this product
		Collection productAttributes = cservice.getProductAttributes(scp
				.getProductId(), locale.getLanguage());
		Map mapAttributes = new HashMap();
		if (productAttributes != null) {

			Iterator i = productAttributes.iterator();
			while (i.hasNext()) {
				ProductAttribute p = (ProductAttribute) i.next();
				mapAttributes.put(p.getOptionValueId(), p);
			}

		}

		if (scp != null) {

			StringBuffer attributesLine = null;

			if (attributes != null) {
				attributesLine = new StringBuffer();
				int count = 0;
				Iterator i = attributes.iterator();
				while (i.hasNext()) {
					OrderProductAttribute opa = (OrderProductAttribute) i
							.next();
					String attrPriceText = opa.getPrice();
					BigDecimal attrPrice = null;
					if (attrPriceText != null) {
						attrPrice = CurrencyUtil.validateCurrency(
								attrPriceText, currency);
					} else {
						attrPrice = opa.getOptionValuePrice();
					}
					// get all information from the attribute
					ProductAttribute pa = (ProductAttribute) mapAttributes
							.get(opa.getProductOptionValueId());
					if (pa != null) {
						if (attrPrice == null) {
							attrPrice = pa.getOptionValuePrice();
						}
					}
					if (attrPrice != null) {
						opa.setOptionValuePrice(attrPrice);
						opa.setPrice(CurrencyUtil
								.displayFormatedAmountNoCurrency(attrPrice,
										currency));
						if (sumPrice == null) {
							sumPrice = new BigDecimal(attrPrice.doubleValue())
									.setScale(2);
						} else {
		
							BigDecimal currentPrice = sumPrice;
							sumPrice = currentPrice.add(attrPrice);
						}
					}
					// opa.setOrderId(Long.parseLong(orderId));
					opa.setOrderProductId(scp.getProductId());
					opa
							.setProductAttributeIsFree(pa
									.isProductAttributeIsFree());

					opa.setProductOption("");
					if (StringUtils.isBlank(opa.getProductOptionValue())) {
						opa.setProductOptionValue("");
					}

					ProductOption po = pa.getProductOption();
					Set poDescriptions = po.getDescriptions();
					if (poDescriptions != null) {
						Iterator pi = poDescriptions.iterator();
						while (pi.hasNext()) {
							ProductOptionDescription pod = (ProductOptionDescription) pi
									.next();
							if (pod.getId().getLanguageId() == LanguageUtil
									.getLanguageNumberCode(lang)) {
								opa
										.setProductOption(pod
												.getProductOptionName());
								break;
							}
						}
					}

					if (StringUtils.isBlank(opa.getProductOptionValue())) {
						ProductOptionValue pov = pa.getProductOptionValue();
						if (pov != null) {
							Set povDescriptions = pov.getDescriptions();
							if (povDescriptions != null) {
								Iterator povi = povDescriptions.iterator();
								while (povi.hasNext()) {
									ProductOptionValueDescription povd = (ProductOptionValueDescription) povi
											.next();
									if (povd.getId().getLanguageId() == LanguageUtil
											.getLanguageNumberCode(lang)) {
										opa.setProductOptionValue(povd
												.getProductOptionValueName());
										break;
									}
								}
							}
						}
					}
					opa.setProductAttributeWeight(pa
							.getProductAttributeWeight());
					if (count == 0) {
						attributesLine.append("[ ");
					}
					attributesLine.append(opa.getProductOption())
							.append(" -> ").append(opa.getProductOptionValue());
					if (count + 1 == attributes.size()) {
						attributesLine.append("]");
					} else {
						attributesLine.append(", ");
					}
					count++;
				}
			}

			// add attribute price to productprice
			if (sumPrice != null) {
				// sumPrice = sumPrice.multiply(new
				// BigDecimal(scp.getProductQuantity()));

				// get product price
				BigDecimal productPrice = scp.getProductPrice();
				productPrice = productPrice.add(sumPrice);

				// added
				scp.setProductPrice(productPrice);

				// BigDecimal finalPrice = scp.getFinalPrice();
				BigDecimal finalPrice = productPrice.multiply(new BigDecimal(
						scp.getProductQuantity()));
				// finalPrice = finalPrice.add(sumPrice);
				// BigDecimal cost = scp.get

				scp.setPriceText(CurrencyUtil.displayFormatedAmountNoCurrency(
						productPrice, currency));
				scp
						.setPriceFormated(CurrencyUtil
								.displayFormatedAmountWithCurrency(finalPrice,
										currency));
			}

			if (attributesLine != null) {
				scp.setAttributesLine(attributesLine.toString());
			}

			Set attributesSet = new HashSet(attributes);

			scp.setOrderattributes(attributesSet);

		}

		return scp;
	}

}



```
