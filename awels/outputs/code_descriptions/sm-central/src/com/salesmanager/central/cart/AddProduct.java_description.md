# AddProduct.java

## Review

**Code Review – `com.salesmanager.central.cart.AddProduct`**

---

## 1. Summary

| Aspect | Description |
|--------|-------------|
| **Purpose** | Handles AJAX / DWR calls for adding/removing products and product attributes, manipulating product relationships, and performing cart‑level calculations. |
| **Key Components** | <ul><li>Product relationship management (`addRelationshipItem`, `removeRelationshipItem`) </li><li>Cart product handling (`addProduct`, `removeProduct`, `removeAttributes`, `addAttributes`) </li><li>Price/total calculation (`calculate`) </li><li>Utility helpers (`createOrderProduct`, `validatePrices`, `getProductsByCategoryId`, etc.)</li></ul> |
| **Design patterns** | *Facade/Service wrapper* – the class acts as a thin façade over several service layers (`CatalogService`, `MerchantService`, `OrderService`).  It also uses a *Utility* style approach via `CheckoutUtil`. |
| **Frameworks / Libraries** | *Servlet API* (HTTP request/session handling) <br>*DWR* (via `WebContextFactory`) <br>*Apache Commons Lang* (`StringUtils`) <br>*log4j* <br>*SalesManager core* (entities, services, util classes) |

---

## 2. Detailed Description

1. **Initialization / Context retrieval**  
   Every public method starts by pulling the `HttpServletRequest`, session, and a custom `Context` object (containing the merchant id, language, and currency).  All data access is performed through statically obtained services (`ServiceFactory.getService(...)`).

2. **Relationship Management**  
   * `addRelationshipItem`: Creates a new `ProductRelationship` after verifying that it does not already exist.  It checks the product belongs to the current merchant.  
   * `removeRelationshipItem`: Deletes an existing relationship; logs a debug message if the relationship does not exist.

3. **Product Listing**  
   * `getProductsHtmlListByCategoryId`: Returns an array of `Product` objects whose `name` field is set to a simple HTML link.  
   * `getProductsByCategoryId`: Returns an array of `OrderProduct` DTOs (lightweight view objects) for a category.

4. **Cart Manipulation**  
   * `addProduct`: Instantiates a new `OrderProduct` (via `createOrderProduct`), assigns a line id and stores it in the session (`SessionUtil.addOrderTotalLine`).  
   * `removeProduct`: Removes the line from the session.  
   * `removeAttributes`: Replaces an `OrderProduct` with a freshly created one (no attributes).  
   * `addAttributes`: Uses `CheckoutUtil.addAttributesFromRawObjects` to merge raw attribute objects into an existing product line.

5. **Price / Total Calculation**  
   `calculate` is the most complex method:  
   * It gathers shipping info (possibly from the session or a passed `ShippingInformation` object).  
   * Validates and normalizes each `OrderProduct` (price format, quantity limits).  
   * Cleans up session state, removes deleted lines.  
   * Delegates to `OrderService.calculateTotal` to perform the actual summation of taxes, shipping, handling, etc.  
   * Formats all monetary values into locale‑aware strings.  

6. **Utility Helpers**  
   * `createOrderProduct` delegates to `CheckoutUtil.createOrderProduct`.  
   * `validatePrices` loops over a product array and marks those with invalid price text.  

7. **Error handling** – Most methods catch generic `Exception` and log it, then return a simple error string or an `OrderProduct` with an error message.  No re‑throwing or rollback logic is performed.

8. **Dependencies** – All external services are obtained via a static `ServiceFactory`.  No dependency injection framework is used.

---

## 3. Functions / Methods

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `removeRelationshipItem(String, String, String)` | Deletes a product relationship. | Product id, related product id, relationship type | Error message string (empty if success) | Logs, calls `CatalogService.removeProductRelationship` |
| `addRelationshipItem(String, String, String)` | Adds a new relationship after checking for duplicates. | Product id, related product id, relationship type | Error message string | Calls `CatalogService.saveOrUpdateProductRelationship` |
| `getProductsHtmlListByCategoryId(String)` | Retrieves products of a category and wraps name in `<a>` tags. | Category id | Array of `Product` (or null) | Sets product `name` property |
| `addAttributes(OrderProductAttribute[], long, int)` | Merges raw attributes into an existing cart line. | Attributes array, product id, line id | Updated `OrderProduct` | Calls `CheckoutUtil.addAttributesFromRawObjects` |
| `calculate(OrderProduct[], ShippingInformation)` | Recalculates cart totals, tax, shipping, handling. | Product array, optional shipping info | `OrderTotalSummary` | Session cleanup, invokes `OrderService.calculateTotal` |
| `removeAttributes(long, int)` | Restores product to original state (no attributes). | Product id, line id | Reset `OrderProduct` | Calls `SessionUtil.resetProduct` |
| `removeProduct(int)` | Deletes a cart line from session. | Line id | void | Calls `SessionUtil.removeOrderTotalLine` |
| `validatePrices(OrderProduct[])` | Validates all price strings. | Array of products | Same array, with error messages set | Logs errors |
| `addProduct(long, int)` | Adds a new line to cart. | Product id, line id | `OrderProduct` | Calls `SessionUtil.addOrderTotalLine` |
| `createOrderProduct(long)` | Factory for a clean `OrderProduct`. | Product id | `OrderProduct` | Delegates to `CheckoutUtil.createOrderProduct` |
| `getProductsByCategoryId(String)` | Retrieves lightweight product DTOs for a category. | Category id | Array of `OrderProduct` | None |

---

## 4. Dependencies

| Library / API | Type | Notes |
|---------------|------|-------|
| `javax.servlet.*` | Standard | Request / session handling |
| `org.apache.commons.lang.StringUtils` | 3rd‑party | Basic string utilities |
| `org.apache.log4j.Logger` | 3rd‑party | Logging |
| `uk.ltd.getahead.dwr.WebContextFactory` | 3rd‑party | DWR request context |
| `com.salesmanager.core.*` | 3rd‑party (core module) | Entities, services, util classes (e.g., `CatalogService`, `OrderService`, `CurrencyUtil`, `LabelUtil`) |
| `com.salesmanager.central.profile.*` | 3rd‑party (profile module) | Context and constants |
| `ServiceFactory` (static) | 3rd‑party | Service locator pattern |
| `SessionUtil` | 3rd‑party | Session helper utilities |
| Java SE (e.g., `BigDecimal`, `Locale`) | Standard | General utilities |

All dependencies are either part of the Java SE runtime, well‑known open‑source libraries, or internal SalesManager modules.

---

## 5. Additional Notes & Recommendations

### 5.1. **Design & Architecture**

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Tight coupling to static services** | Hard to test, no ability to swap implementations. | Adopt dependency injection (e.g., Spring or CDI) and inject `CatalogService`, `OrderService`, etc. |
| **No interface / abstraction** | Client code cannot be swapped for a mock implementation. | Expose `AddProduct` via an interface or wrap it in a façade that can be mocked. |
| **Mix of business logic and HTTP/session handling** | Violates separation of concerns; makes unit testing impossible. | Extract business logic into pure service classes that accept domain objects, and keep a thin servlet/DWR adapter that only deals with request/session. |
| **Use of `WebContextFactory`** | Adds hidden dependency on DWR; not portable to other frameworks. | Replace with explicit request injection or use a framework‑agnostic controller. |

### 5.2. **Error Handling**

* Generic `catch (Exception e)` is used everywhere.  
  * **Problem**: Masks different failure modes (e.g., `NumberFormatException`, `NullPointerException`, database errors).  
  * **Fix**: Catch specific exceptions, log them separately, and return meaningful error codes or messages. Consider throwing custom exceptions to propagate failure state.

* Methods often return a string or an `OrderProduct` with an error message, but callers rarely inspect the error.  
  * **Fix**: Define a result wrapper (`OperationResult<T>`) that contains `success`, `data`, and `error` fields.

### 5.3. **Null‑Safety & Validation**

* Several places access collections or session attributes without null checks (`SessionUtil.getShippingMethods(req)`, `SessionUtil.getOrderProducts(req)`).  
  * **Problem**: Can lead to `NullPointerException`.  
  * **Fix**: Guard against null or provide default empty collections.

* Parsing strings to numbers (`Long.parseLong`, `Integer.parseInt`) is done without handling `NumberFormatException`.  
  * **Fix**: Validate input strings or catch parsing errors to return user‑friendly messages.

### 5.4. **Performance & Scalability**

* `getProductsHtmlListByCategoryId` builds HTML inside a service method; this mixes view concerns with business logic.  
  * **Problem**: Re‑uses `Product` DTO for a different purpose, confusing callers.  
  * **Fix**: Return plain `Product` objects and let the front end build the link, or create a dedicated DTO.

* In `calculate`, the method iterates over `products` and `savedOrderProducts` in multiple passes.  
  * **Problem**: O(n²) style lookups.  
  * **Fix**: Use a `Map<String, OrderProduct>` for `savedOrderProducts` and fetch directly.

* Currency conversions and formatting are repeated multiple times.  
  * **Fix**: Encapsulate formatting logic in a utility service.

### 5.5. **Security**

* The merchant id is obtained from the session (`ctx.getMerchantid()`).  
  * **Risk**: If the session can be hijacked, an attacker could manipulate product relationships for another merchant.  
  * **Mitigation**: Validate the merchant id against the authenticated user’s permissions before performing operations.

### 5.6. **Coding Style**

* Mixed use of `ArrayList`, `HashMap`, `Collection` with raw types.  
  * **Fix**: Use generics everywhere (`List<OrderProduct>`, `Map<String, OrderProduct>`) to avoid unchecked warnings.

* Magic numbers (e.g., `"0"` for default BigDecimal) should be constants.

* Logging messages use concatenation instead of placeholders. Use `log.debug("Error: {}", msg)` to avoid expensive string building when debug is off.

* In many places, `try` blocks wrap the entire method body, which hides the actual line causing the exception.  Move the try‑catch around the minimal code that can throw.

### 5.7. **Unit Testability**

* Current design requires a live servlet context and session.  
  * **Recommendation**: Extract a pure `CartService` that works with plain Java objects.  The DWR servlet can delegate to it.  Write tests against this service with mocked services.

### 5.8. **Potential NullPointerExceptions & Edge Cases**

| Method | Likely NPE | Fix |
|--------|------------|-----|
| `removeRelationshipItem` – `relationship.getRelationshipId()` if relationship is null. | Guard earlier. |
| `calculate` – `SessionUtil.getOrder(req)` used after a successful calculation. | Ensure order exists before using it. |
| `addAttributes` – `CheckoutUtil.addAttributesFromRawObjects` may throw unchecked exceptions. | Wrap with fine‑grained try‑catch. |
| `validatePrices` – `product.getPriceText()` may be null. | Check for null before validation. |

---

### 5.8. Summary of Suggested Refactor Path

1. **Create a `CartService` (or `OrderService`) layer** that contains all business logic (product lookup, relationship handling, price validation, total calculation).  
2. **Inject services** using a DI container; replace the static `ServiceFactory`.  
3. **Remove HTML generation** from service methods; return plain DTOs.  
4. **Define a clear error model** (`OperationResult<T>` or similar).  
5. **Use generics and Java 8+ streams** to simplify loops and reduce boilerplate.  
6. **Add unit tests** for the extracted service; mock dependencies via a mocking framework.  

---

**Final Verdict**

`AddProduct` functions correctly in a production environment but suffers from several design smells that impede maintenance, testability, and security.  A focused refactor that decouples business logic from the servlet/session layer, adopts dependency injection, and introduces stricter error handling will make the codebase far more robust and future‑proof.

## Code Critique



## Code Preview

```java
/*
 * Provided by CSTI Consulting 
 * Following GNU LESSER GENERAL PUBLIC LICENSE
 * You may obtain more details at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.central.cart;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.catalog.ProductRelationship;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductAttribute;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.entity.shipping.ShippingOption;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.www.SessionUtil;


/**
 * Used with AJAX / DWR requests
 * @author Carl Samson
 *
 */
public class AddProduct {

	private Logger log = Logger.getLogger(AddProduct.class);
	
	
	public String removeRelationshipItem(String productId,String relatedProductId, String relationShipType) {
		
		
		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();


		Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);

		CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);
		
		Locale locale = LocaleUtil.getLocale(req);
		
		try {
			
			long lProductId = Long.parseLong(productId);
			int iRelationShipType = Integer.parseInt(relationShipType);
			long lRelatedProductId = Long.parseLong(relatedProductId);
			ProductRelationship pr = cservice.getProductRelationship(lProductId,lRelatedProductId,iRelationShipType,ctx.getMerchantid());
			
			if(pr==null) {
				log.debug("Error removing relation : Relationship type " + iRelationShipType + " for product id " + lProductId + " and related productId " + lRelatedProductId);
				return LabelUtil.getInstance().getText(locale,"error.message.invalidrelationship.remove");
			} else {
				cservice.removeProductRelationship(pr);
			}
			
		} catch (Exception e) {
			log.error(e);
			return LabelUtil.getInstance().getText(locale,"error.message.invalidrelationship.remove");
		}
		
		return "";
		
	}

	public String addRelationshipItem(String productId,String relatedProductId,String relationShipType) {
		
		
		
		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();


		Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);

		CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);
		
		Locale locale = LocaleUtil.getLocale(req);
		
		try {
			long lProductId = Long.parseLong(productId);
			long lRelatedProductId = Long.parseLong(relatedProductId);
			int iRelationShipType = Integer.parseInt(relationShipType);
			

			Product p = cservice.getProduct(lProductId);
			
			if(p.getMerchantId()==ctx.getMerchantid()) {
				
				
				//check if the relationship exist
				ProductRelationship prExist = cservice.getProductRelationship(lProductId,lRelatedProductId,iRelationShipType,ctx.getMerchantid());
				if(prExist!=null) {
					return LabelUtil.getInstance().getText(locale,"error.message.invalidrelationship.exist");
				}
				
				ProductRelationship pr = new ProductRelationship();
				pr.setMerchantId(ctx.getMerchantid());
				pr.setProductId(p.getProductId());
				pr.setRelatedProductId(lRelatedProductId);
				pr.setRelationshipType(iRelationShipType);
				cservice.saveOrUpdateProductRelationship(pr);
				
				
			}
			
			return "";
			
		} catch (Exception e) {
			log.error(e);
			return LabelUtil.getInstance().getText(locale,"error.message.invalidrelationship");
		}
		
	}
	
	public Product[] getProductsHtmlListByCategoryId(String categoryId) {
		
		
		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();


		Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);


		Product[] returnArray = null;
		
		CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);

		Locale locale = LocaleUtil.getLocale(req);
		try {
			long lCategoryId = Long.parseLong(categoryId);
			Collection products = cservice.getProductsByMerchantIdAndCategoryIdAndLanguageId(ctx.getMerchantid(),lCategoryId,LanguageUtil.getLanguageNumberCode(ctx.getLang()));
			if(products!=null && products.size()>0) {
				
				returnArray = new Product[products.size()];
				
				Iterator i = products.iterator();
				
				int count = 0;
				
				while(i.hasNext()) {
					StringBuffer productLine = new StringBuffer();
					com.salesmanager.core.entity.catalog.Product d = (com.salesmanager.core.entity.catalog.Product)i.next();
					
					ProductDescription desc = d.getProductDescription();
					

					if(desc == null) {
						desc = new ProductDescription();
						desc.setProductName(String.valueOf(d.getProductId()));
					}
					
					

					productLine.append("<a href=\"#\" rel=\"").append(desc.getProductName()).append("\">").append(desc.getProductName()).append("</a>");
					d.setName(productLine.toString());
						
					returnArray[count] = d;
					count++;
					
				}
			}
		
		} catch(Exception e) {
			log.error(e);
		}
			return returnArray;
		
	}

	/**
	 * Add OrderAttributes to an existing OrderProduct
	 * @param attributes
	 * @param productId
	 * @param lineId
	 * @return
	 */
	public OrderProduct addAttributes(OrderProductAttribute attributes[],long productId, int lineId) {


		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();


		HttpSession session = WebContextFactory.get().getSession();

		Context ctx = (Context)session.getAttribute(ProfileConstants.context);

		Locale locale = LocaleUtil.getLocale(req);

		try {



			List attrList = Arrays.asList(attributes);


			OrderProduct op = com.salesmanager.core.util.CheckoutUtil.addAttributesFromRawObjects(attrList, productId, String.valueOf(lineId), ctx.getCurrency(), req);


			return op;

		} catch (Exception e) {
			log.error(e);
			OrderProduct op = new OrderProduct();
			op.setErrorMessage(LabelUtil.getInstance().getText(locale,"messages.genericmessage"));
			return op;

		}

	}




	/**
	 * Synchronize Session objects with passed parameters
	 * Validates input parameters
	 * Then delegates to OrderService for OrderTotalSummary calculation
	 * @param products
	 */
	public OrderTotalSummary calculate(OrderProduct[] products, ShippingInformation shippingMethodLine) {

		//subtotal
		//quantity
		//tax
		//shipping
		//handling
		//other prices


		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();

		Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);

		Order order = SessionUtil.getOrder(req);
		
		String currency = null;
		
		try {
			
			MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice.getMerchantStore(ctx.getMerchantid());
			
			currency = store.getCurrency();
			

			if(order!=null && !StringUtils.isBlank(order.getCurrency())) {
				currency = order.getCurrency();
			}
			
			
			
			
		} catch (Exception e) {
			log.error(e);
		}
		
		OrderTotalSummary total = new OrderTotalSummary(currency);
		
		


		Customer customer = SessionUtil.getCustomer(req);
		
		Locale locale = LocaleUtil.getLocale(req);

		//Shipping
		ShippingInformation shippingInfo = SessionUtil.getShippingInformation(req);

		Shipping shipping = null;

		if(shippingInfo==null) {
			shippingInfo = new ShippingInformation();
		}

		if(shippingMethodLine != null && shippingMethodLine.getShippingMethodId()==null) {//reset shipping
			//shippingMethodLine = new ShippingInformation();

			if(req.getSession().getAttribute("PRODUCTLOADED")!=null) {
				shipping = new Shipping();
				shipping.setHandlingCost(shippingInfo.getHandlingCost());
				shipping.setShippingCost(shippingInfo.getShippingCost());
				shipping.setShippingDescription(shippingInfo.getShippingMethod());
				shipping.setShippingModule(shippingInfo.getShippingModule());
				req.getSession().removeAttribute("PRODUCTLOADED");

			} else {

				shippingInfo.setShippingCostText(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal("0"), ctx.getCurrency()));
				shippingInfo.setShippingMethodId(null);
				shippingInfo.setShippingMethod(null);
				shippingInfo.setShippingCost(new BigDecimal("0"));
				try {
					SessionUtil.removeShippingInformation(req);
				} catch (Exception e) {
					log.error(e);
				}



			}


		} else { //retreive shipping info in http session
			shipping = new Shipping();
			Map shippingOptionsMap = SessionUtil.getShippingMethods(req);
			String method = shippingMethodLine.getShippingMethodId();


			if(shippingInfo.getShippingCost()!=null && shippingInfo.getShippingMethod()!=null) {

				shipping.setHandlingCost(shippingInfo.getHandlingCost());
				shipping.setShippingCost(shippingInfo.getShippingCost());
				shipping.setShippingDescription(shippingInfo.getShippingMethod());
				shipping.setShippingModule(shippingInfo.getShippingModule());

			} else {

				if(shippingOptionsMap==null || method==null) {
					shippingMethodLine.setShippingCostText(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal("0"), ctx.getCurrency()));
					shippingInfo = shippingMethodLine;
				} else {//after a selection
					//retreive shipping option
					ShippingOption option = (ShippingOption)shippingOptionsMap.get(method);

					//get the latest shipping information (handling, free ...)

					shippingInfo.setShippingMethodId(option.getOptionId());
					shippingInfo.setShippingOptionSelected(option);
					shippingInfo.setShippingMethod(option.getDescription());

					shippingInfo.setShippingCost(option.getOptionPrice());
					shippingInfo.setShippingModule(option.getModule());

					shipping.setHandlingCost(shippingInfo.getHandlingCost());
					shipping.setShippingCost(shippingInfo.getShippingCost());
					shipping.setShippingDescription(option.getDescription());
					shipping.setShippingModule(option.getModule());

					//total.setShipping(true);



				}

			}
		}



		List productList = new ArrayList();

		try {

			//validate numeric quantity

			//validate numeric price
			if(products!=null) {

				//get products from httpsession
				Map savedOrderProducts = SessionUtil.getOrderProducts(req);
				Map currentProducts = new HashMap();



				if(savedOrderProducts==null) {
					savedOrderProducts = SessionUtil.createSavedOrderProducts(req);
				}




				total.setOrderProducts(products);

				if(order==null) {
					log.error("No order exist for the price calculation");
					total.setErrorMessage(LabelUtil.getInstance().getText(locale,"messages.genericmessage"));
					return total;
				}

				//validates amounts
				BigDecimal oneTimeSubTotal = total.getOneTimeSubTotal();

				for(int i=0;i<products.length;i++) {
					OrderProduct product = products[i];

					currentProducts.put(String.valueOf(product.getLineId()), product);

					//get the original line
					OrderProduct oproduct = (OrderProduct)savedOrderProducts.get(String.valueOf(product.getLineId()));

					if(oproduct==null) {
						oproduct=this.createOrderProduct(product.getProductId());
					}
					
					if(product.getProductQuantity()>oproduct.getProductQuantityOrderMax()) {
						product.setProductQuantity(oproduct.getProductQuantityOrderMax());
					}

					productList.add(oproduct);

					//check that productid match
					if(product.getProductId()!=oproduct.getProductId()) {//set an error message
						oproduct.setErrorMessage(LabelUtil.getInstance().getText(locale,"messages.invoice.product.invalid"));
						oproduct.setPriceText("0");
						oproduct.setProductPrice(new BigDecimal(0));
						oproduct.setPriceFormated(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal(0), ctx.getCurrency()));
						continue;
					}


					//validate and set the final price
					try {
						product.setPriceErrorMessage(null);//reset any error message
						product.setErrorMessage(null);
						//set price submited
						BigDecimal price = CurrencyUtil.validateCurrency(product.getPriceText(), ctx.getCurrency());
						oproduct.setPriceText(product.getPriceText());
						oproduct.setProductPrice(price);
						oproduct.setPriceFormated(CurrencyUtil.displayFormatedAmountWithCurrency(price, ctx.getCurrency()));
						oproduct.setProductQuantity(product.getProductQuantity());
						oproduct.setPriceErrorMessage(null);
						oproduct.setErrorMessage(null);


						double finalPrice = price.doubleValue() * product.getProductQuantity();
						BigDecimal bdFinalPrice = new BigDecimal(finalPrice);

						//price calculated
						oproduct.setCostText(CurrencyUtil.displayFormatedAmountWithCurrency(bdFinalPrice, ctx.getCurrency()));
						oproduct.setFinalPrice(bdFinalPrice);


					} catch (NumberFormatException nfe) {
						oproduct.setPriceErrorMessage(LabelUtil.getInstance().getText(locale,"messages.price.invalid"));
						oproduct.setPriceText("0");
						oproduct.setProductPrice(new BigDecimal(0));
						oproduct.setCostText(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal(0), ctx.getCurrency()));
						oproduct.setPriceFormated(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal(0), ctx.getCurrency()));
						//set shipping to 0
						ShippingInformation info = new ShippingInformation();
						shippingMethodLine.setShippingCostText(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal("0"), ctx.getCurrency()));
						total.setShippingLine(info);
						total.setShippingTotal(new BigDecimal("0"));

					} catch(com.opensymphony.xwork2.validator.ValidationException e) {
						oproduct.setPriceErrorMessage(LabelUtil.getInstance().getText(locale,"messages.price.invalid"));
						oproduct.setPriceText("0");
						oproduct.setProductPrice(new BigDecimal(0));
						oproduct.setCostText(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal(0), ctx.getCurrency()));
						oproduct.setPriceFormated(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal(0), ctx.getCurrency()));
						//set shipping to 0
						ShippingInformation info = new ShippingInformation();
						shippingMethodLine.setShippingCostText(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal("0"), ctx.getCurrency()));
						total.setShippingLine(info);
						total.setShippingTotal(new BigDecimal("0"));

					} catch(Exception e) {
						log.error(e);
					}




				}


				List removable = null;
				//cleanup http session
				Iterator it = savedOrderProducts.keySet().iterator();
				while(it.hasNext()) {
					String key = (String)it.next();
					if(!currentProducts.containsKey(key)) {
						if(removable==null) {
							removable = new ArrayList();
						}
						removable.add(key);
					}
				}

				if(removable!=null) {
					Iterator removIt = removable.iterator();
					while(removIt.hasNext()) {
						String key = (String)removIt.next();
						SessionUtil.removeOrderTotalLine(key, req);
					}
				}

				OrderService oservice = (OrderService)ServiceFactory.getService(ServiceFactory.OrderService);
				total = oservice.calculateTotal(order, productList, customer,shipping,ctx.getCurrency(), LocaleUtil.getLocale(req));


				OrderProduct[] opArray = new OrderProduct[productList.size()];
				OrderProduct[] o = (OrderProduct[])productList.toArray(opArray);
				total.setOrderProducts(o);

				total.setShippingLine(shippingInfo);


				Order savedOrder = SessionUtil.getOrder(req);
				savedOrder.setTotal(total.getTotal());
				savedOrder.setOrderTax(total.getTaxTotal());
				SessionUtil.setOrder(savedOrder,req);

			}


		} catch (Exception e) {
			log.error(e);
			total = new OrderTotalSummary(currency);
			total.setErrorMessage(LabelUtil.getInstance().getText(locale,"messages.genericmessage"));
		}

		ShippingInformation shippingLine = total.getShippingLine();
		if(shippingLine!=null) {
			shippingLine.setShippingCostText(CurrencyUtil.displayFormatedAmountWithCurrency(shippingLine.getShippingCost(), ctx.getCurrency()));
		} else {
			shippingLine = new ShippingInformation();
			shippingLine.setShippingCostText(CurrencyUtil.displayFormatedAmountWithCurrency(new BigDecimal("0"), ctx.getCurrency()));
		}

		if(shippingLine.getHandlingCost()!=null) {
			shippingLine.setHandlingCostText(CurrencyUtil.displayFormatedAmountWithCurrency(shippingMethodLine.getHandlingCost(), ctx.getCurrency()));
		}

		if(total.getShippingTotal()!=null) {
			total.setShippingTotalText(CurrencyUtil.displayFormatedAmountWithCurrency(total.getShippingTotal(), ctx.getCurrency()));
		}


		if(total.getOneTimeSubTotal()!=null){
			total.setOneTimeSubTotalText(CurrencyUtil.displayFormatedAmountWithCurrency(total.getOneTimeSubTotal(), ctx.getCurrency()));
		}

		if(total.getRecursiveSubTotal()!=null){
			total.setRecursiveSubTotalText(CurrencyUtil.displayFormatedAmountWithCurrency(total.getRecursiveSubTotal(), ctx.getCurrency()));
		}


		if(total.getTotal()!=null) {
			total.setTotalText(CurrencyUtil.displayFormatedAmountWithCurrency(total.getTotal(), ctx.getCurrency()));
		}
		return total;

	}

	public OrderProduct removeAttributes(long productId, int lineId) {

		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();


		HttpSession session = WebContextFactory.get().getSession();

		OrderProduct op;




		try {


			//revert back to original product
			op = this.createOrderProduct(productId);
			op.setLineId(lineId);
			return SessionUtil.resetProduct(op,productId,String.valueOf(lineId),req);

		} catch (Exception e) {
			log.error(e);
			op = new OrderProduct();
			op.setErrorMessage(LabelUtil.getInstance().getText(LocaleUtil.getLocale(req),"error.cart.noproduct"));
			return op;
		}






	}



	public void removeProduct(int lineId) {


		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();

		HttpSession session = WebContextFactory.get().getSession();

		try {

			SessionUtil.removeOrderTotalLine(String.valueOf(lineId), req);

		} catch (Exception e) {
			log.error(e);
		}

	}

	/**
	 * Validates prices inputed by the end user (for invoice)
	 * @param selections
	 * @return
	 */
	public OrderProduct[] validatePrices(OrderProduct[] selections) {


		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();


		Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);

		try {


			if(selections!=null) {

				for(int i=0;i<selections.length;i++) {
					OrderProduct product = selections[i];
					try {
						CurrencyUtil.validateCurrency(product.getPriceText(), ctx.getCurrency());
					} catch (Exception e) {
						product.setPriceErrorMessage(LabelUtil.getInstance().getText(LocaleUtil.getLocale(req),"messages.price.invalid"));
					}
				}

			}


		} catch (Exception e) {
			log.error(e);
		}

		return selections;



	}


	/**
	 * Creates a ShoppingCart line
	 * @param orderId
	 * @param productId
	 * @param lineId
	 * @return
	 */
	public OrderProduct addProduct(long productId,int lineId) {



		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();




		HttpSession session = WebContextFactory.get().getSession();
		try {


				Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);


				OrderProduct op = this.createOrderProduct(productId);


				op.setLineId(lineId);

				SessionUtil.addOrderTotalLine(op, String.valueOf(lineId),req);

				return op;






		} catch (Exception e) {
			log.error(e);
			OrderProduct scp = new OrderProduct();
			scp.setErrorMessage(LabelUtil.getInstance().getText(LocaleUtil.getLocale(req),"error.cart.addproducterror"));
		}

		OrderProduct scp = new OrderProduct();
		scp.setErrorMessage(LabelUtil.getInstance().getText(LocaleUtil.getLocale(req),"error.cart.noproduct"));

		return scp;

	}

	public OrderProduct createOrderProduct(long productId) throws Exception {


		HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();


		Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);

		//Locale locale = req.getLocale();

		
		Locale locale = LocaleUtil.getLocale(req);

		
		

		return com.salesmanager.core.util.CheckoutUtil.createOrderProduct(productId,locale,ctx.getCurrency());




	}

	public OrderProduct[] getProductsByCategoryId(String categoryId) {

		try {

			HttpServletRequest req = WebContextFactory.get().getHttpServletRequest();


			Context ctx = (Context)req.getSession().getAttribute(ProfileConstants.context);



			CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);

			Collection coll = cservice.getProductsByMerchantIdAndCategoryIdAndLanguageId(ctx.getMerchantid(),Long.parseLong(categoryId),LanguageUtil.getLanguageNumberCode(ctx.getLang()));



			if(coll!=null && coll.size()>0) {

				OrderProduct[] scparray = new OrderProduct[coll.size()];
				Iterator i = coll.iterator();
				int count = 0;
				while(i.hasNext()) {
					Product d = (Product)i.next();
					ProductDescription pd = d.getProductDescription();
					OrderProduct scp = new OrderProduct();
					scp.setProductId(d.getProductId());
					scp.setProductName(pd.getProductName());
					scp.setProductDescription(pd.getProductDescription());
					scp.setProductImage(d.getProductImage());
					scparray[count]=scp;
					count++;
				}

				return scparray;
			} else {
				log.warn("No prooducts belong to categoryId " + categoryId);
			}






		} catch (Exception e) {
			log.error(e);
		}

		return null;

	}

}



```
