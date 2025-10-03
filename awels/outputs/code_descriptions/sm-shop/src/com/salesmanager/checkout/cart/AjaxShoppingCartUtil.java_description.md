# AjaxShoppingCartUtil.java

## Review

## 1. Summary  
`AjaxShoppingCartUtil` is a helper class that manages the shopping‑cart logic for the Ajax‑based checkout flow.  
Its responsibilities are:  

| Responsibility | Where it lives in the code |
|----------------|---------------------------|
| **Session synchronization** – rebuilds the order‑product list stored in the HTTP session (`synchronizeProductList`). |
| **Product removal** – removes a line item from the order and updates the mini‑cart (`removeProduct`). |
| **Total calculation** – validates user input, applies shipping rules, updates the order and calculates the final totals (`calculate`). |

The class relies on a mixture of **domain entities** (`Order`, `OrderProduct`, `Shipping`, …), **service layer** (`OrderService` via `ServiceFactory`), and a set of **utility classes** (`SessionUtil`, `MiniShoppingCartUtil`, `CurrencyUtil`, etc.).  
It uses **Apache Log4j** for logging and **Apache Commons Lang** for string checks.  
The code is tightly coupled to the servlet container through `WebContextFactory.get()`, which exposes the current `HttpServletRequest` and `HttpServletResponse` via a thread‑local.  

---

## 2. Detailed Description  

### 2.1 Core Flow

1. **Initialization** – All public methods obtain the current `HttpServletRequest`/`HttpServletResponse` and the current session via `WebContextFactory`.  
2. **Session Retrieval** – `SessionUtil` is used everywhere to read/write session attributes (order, customer, shipping info, mini‑cart, etc.).  
3. **Business Logic** –  
   * `removeProduct` removes a line from the order, updates the mini‑cart, and recalculates totals.  
   * `calculate` performs a full recalculation: validates quantity & price, applies shipping, updates the mini‑cart, calls `OrderService.calculateTotal`, and persists the updated order and totals back into the session.  
4. **Cleanup** – After a recalculation, `synchronizeProductList` rebuilds the attribute `ORDER_PRODUCT_LIST` which is used by the front‑end.

### 2.2 Design Choices & Patterns

| Choice | Why it’s used | Possible drawback |
|--------|----------------|-------------------|
| **ServiceFactory** | Decouples implementation of `OrderService` | Global static state, hard to mock |
| **Thread‑local request context** (`WebContextFactory`) | Avoids passing request objects around | Hidden dependencies, hard to unit test |
| **SessionUtil** | Centralises session access | No separation of concerns between business logic and session handling |
| **Raw types** | Legacy code base, minimal type safety | Compiler warnings, potential `ClassCastException` |

### 2.3 Assumptions & Constraints

* The user is always logged in (`SessionUtil.getCustomer(req)` should not return `null`).  
* All monetary values are represented by `BigDecimal`.  
* Shipping options are stored in the session under `getShippingMethods(req)`.  
* The mini‑cart (`ShoppingCart`) is optional and may be `null`.  
* The code assumes a single thread per HTTP request – standard servlet behaviour.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `private void synchronizeProductList(HttpServletRequest req)` | Builds a `List<OrderProduct>` from the session map and stores it under `ORDER_PRODUCT_LIST`. | `HttpServletRequest` | `void` | Writes to session |
| `public void removeProduct(int lineId)` | Removes a line item from the order and updates the mini‑cart. | `int lineId` | `void` | Session modifications, logs |
| `public OrderTotalSummary calculate(OrderProduct[] products, ShippingInformation shippingMethodLine)` | Validates input, recalculates all totals, updates session, returns `OrderTotalSummary`. | `OrderProduct[] products`, `ShippingInformation shippingMethodLine` | `OrderTotalSummary` | Complex – modifies order, shipping, totals, mini‑cart, session attributes, logs |

### 3.1 `removeProduct`

* Calls `SessionUtil.removeOrderTotalLine()` to delete the order line.  
* Calls `synchronizeProductList()` to rebuild the product list.  
* Iterates over the mini‑cart to filter out the removed product, then recalculates its total.  
* Serialises the mini‑cart to JSON (`MiniShoppingCartSerializationUtil`).  
* Calls `MiniShoppingCartUtil.calculateTotal(cart, store)` – note the missing null‑check on `cart`.  
* Logs all exceptions.

### 3.2 `calculate`

* Retrieves store, currency, order, customer, shipping information.  
* Handles the case where a shipping method has been reset (`shippingMethodLine.getShippingMethodId() == null`).  
* Validates each product: product ID, quantity (max order quantity), price.  
* Updates the product list and mini‑cart accordingly.  
* Uses `OrderService.calculateTotal` to compute all line totals.  
* Persists the updated order and totals back into the session.  
* Formats all monetary values for display.  
* Finally calls `synchronizeProductList(req)` to keep the session in sync.

---

## 4. Dependencies  

| Library / Class | Category | Comments |
|-----------------|----------|----------|
| `org.apache.commons.lang.StringUtils` | 3rd‑party | Classic library, but not generics‑safe. |
| `org.apache.log4j.Logger` | 3rd‑party | Old Log4j 1.x, consider upgrading to Log4j 2 or SLF4J. |
| `uk.ltd.getahead.dwr.WebContextFactory` | 3rd‑party | DWR framework for Ajax – provides thread‑local request/response. |
| Domain classes (`Order`, `OrderProduct`, etc.) | Application | JPA / Hibernate entities. |
| `ServiceFactory` / `OrderService` | Application | Service layer abstraction. |
| `SessionUtil`, `MiniShoppingCartUtil`, `CurrencyUtil`, `LabelUtil`, `LocaleUtil`, `OrderUtil`, `MiniShoppingCartSerializationUtil` | Application | Helper utilities that hide business logic. |
| `javax.servlet.http.*` | Java EE | Servlet API. |

All dependencies are **third‑party** except for the application‑specific utilities and domain classes. No native OS or platform dependencies are present.

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (`Map orderProducts`, `List products`) | Compile‑time warnings, possible `ClassCastException` | Use generics: `Map<String, OrderProduct> orderProducts` etc. |
| **Magic Strings** (`"ORDER_PRODUCT_LIST"`, `"PRODUCTLOADED"`) | Hard to refactor, typo‑prone | Extract constants. |
| **Large method (`calculate`)** | Hard to understand, unit‑test, maintain | Split into small private helper methods (e.g., `validateProducts`, `applyShipping`, `updateMiniCart`). |
| **Double → BigDecimal conversions** (`double finalPrice = price.doubleValue() * product.getProductQuantity();`) | Floating‑point errors | Perform all arithmetic with `BigDecimal` (e.g., `price.multiply(BigDecimal.valueOf(product.getProductQuantity()))`). |
| **Exception handling** (`catch (Exception e) { log.error(e); }`) | Swallows all exceptions, hides real problems | Catch specific exceptions, propagate or wrap in custom exception. |
| **Thread‑local context** (`WebContextFactory`) | Hidden dependency, hard to test | Pass `HttpServletRequest`/`HttpSession` as parameters or use a façade. |
| **Null checks** | Potential NPE (`MiniShoppingCartUtil.calculateTotal(cart, store)` when `cart==null`) | Guard against `null`. |
| **Logging level** | Too noisy (`log.error(e)`) | Use more granular levels (debug, warn). |
| **Unit tests** | None shown | Write tests for each public method, mock session data. |
| **Internationalisation** | Hardcoded currency formatting via `CurrencyUtil` | Use `NumberFormat` with locale, or a dedicated `Money` library. |
| **Code duplication** (`shippingInfo` handling) | Hard to maintain | Refactor into a dedicated `ShippingProcessor`. |
| **Hard‑coded business rules** (max quantity, product validation) | Violates separation of concerns | Move into domain/service layer. |

### 5.2 Performance & Thread‑Safety

* The class is **stateless** (only the logger is static). Thread‑safety is mainly a concern around the session objects.  
* Repeated session look‑ups (e.g., `SessionUtil.getOrder(req)`) could be cached locally inside the method for readability.  
* The heavy `calculate` method may become a bottleneck on high traffic sites; consider caching intermediate results or asynchronous recalculations.

### 5.3 Future Enhancements

1. **Replace DWR with modern frameworks** – e.g., Spring MVC + Jackson.  
2. **Introduce DTOs** to decouple domain entities from the UI.  
3. **Refactor shipping logic** into a strategy pattern, making it easier to plug new shipping providers.  
4. **Adopt a proper logging framework** (SLF4J + Logback/Log4j2).  
5. **Add unit and integration tests** using JUnit + Mockito, focusing on the `calculate` method’s different branches.  
6. **Use Java 8+ streams & Optional** to simplify collection handling.  
7. **Encapsulate session access** in a dedicated `CartSession` object, making the code more testable.  
8. **Remove magic numbers** and make configuration external (e.g., max quantity validation).  
9. **Handle multi‑currency** more cleanly, possibly with the `javax.money` API.

### 5.4 Edge Cases Not Handled

* **Concurrent modifications** – two simultaneous AJAX requests could corrupt the order.  
* **Large cart** – performance of `synchronizeProductList` could degrade.  
* **Invalid session** – missing order or customer leads to `NullPointerException` later.  
* **Negative or zero quantity** – not explicitly checked.  
* **Shipping method not found** – silent fallback to default values may hide errors.

---

### Bottom Line  

`AjaxShoppingCartUtil` is a functional, albeit monolithic, helper that bridges the servlet layer and the business service layer for a legacy e‑commerce checkout flow. Refactoring the class into smaller, testable units, replacing raw types with generics, modernising the request handling, and cleaning up duplicated logic would greatly improve maintainability and robustness.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 25, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.checkout.cart;

/**
 * Main Shopping Cart
 */
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;

import com.salesmanager.checkout.util.MiniShoppingCartUtil;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderTotal;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.orders.ShoppingCart;
import com.salesmanager.core.entity.orders.ShoppingCartProduct;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.entity.shipping.ShippingOption;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.MiniShoppingCartSerializationUtil;
import com.salesmanager.core.util.OrderUtil;
import com.salesmanager.core.util.StringUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class AjaxShoppingCartUtil {

	private static Logger log = Logger.getLogger(AjaxShoppingCartUtil.class);

	private void synchronizeProductList(HttpServletRequest req) {

		// rebuild list
		HttpSession session = WebContextFactory.get().getSession();

		Map orderProducts = SessionUtil.getOrderProducts(req);
		// transform to a list
		List products = new ArrayList();

		if (orderProducts != null) {
			Iterator i = orderProducts.keySet().iterator();
			while (i.hasNext()) {
				String line = (String) i.next();
				OrderProduct op = (OrderProduct) orderProducts.get(line);
				products.add(op);
			}

		}

		session.setAttribute("ORDER_PRODUCT_LIST", products);

	}

	public void removeProduct(int lineId) {

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();
		HttpServletResponse resp = WebContextFactory.get()
		.getHttpServletResponse();
		Locale locale = LocaleUtil.getLocale(req);
		HttpSession session = WebContextFactory.get().getSession();
		MerchantStore mStore = SessionUtil.getMerchantStore(req);

		try {

			SessionUtil.removeOrderTotalLine(String.valueOf(lineId), req);
			synchronizeProductList(req);

			// mini cart
			ShoppingCart cart = SessionUtil.getMiniShoppingCart(req);
			if (cart != null) {
				Collection prds = cart.getProducts();
				List prdscart = new ArrayList();
				if (prds != null) {
					Iterator iprd = prds.iterator();
					while (iprd.hasNext()) {
						ShoppingCartProduct scp = (ShoppingCartProduct) iprd
								.next();
						if (scp.getMainCartLine()!=null && !scp.getMainCartLine().equals(
								String.valueOf(lineId))) {
							prdscart.add(scp);
						}
					}
				}
				cart.setProducts(prdscart);
				String sc = MiniShoppingCartSerializationUtil.serializeToJSON(cart);
			
				
				if(prdscart!=null && prdscart.size()>0) {
					SessionUtil.removeMiniShoppingCart(req);
				}
			}
			
			MerchantStore store = SessionUtil.getMerchantStore(req);

			MiniShoppingCartUtil.calculateTotal(cart, store);


			


		} catch (Exception e) {
			log.error(e);
		}

	}

	/**
	 * Synchronize Session objects with passed parameters Validates input
	 * parameters Then delegates to OrderService for OrderTotalSummary
	 * calculation. Invoked when recalculate, remove product and changing quantities
	 * 
	 * @param products
	 */
	public OrderTotalSummary calculate(OrderProduct[] products,
			ShippingInformation shippingMethodLine) {

		// subtotal
		// quantity
		// tax
		// shipping
		// handling
		// other prices

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();

		MerchantStore store = SessionUtil.getMerchantStore(req);
		
		String currency = store.getCurrency();
		
		
		// requires order from http session
		Order order = SessionUtil.getOrder(req);
		
		if(order!=null && !StringUtils.isBlank(order.getCurrency())) {
			currency = order.getCurrency();
		}

		OrderTotalSummary total = new OrderTotalSummary(currency);

		Customer customer = SessionUtil.getCustomer(req);

		// Shipping
		ShippingInformation shippingInfo = SessionUtil
				.getShippingInformation(req);

		Shipping shipping = null;

		if (shippingInfo == null) {
			shippingInfo = new ShippingInformation();
		}

		if (shippingMethodLine != null
				&& shippingMethodLine.getShippingMethodId() == null) {// reset
																		// shipping
			// shippingMethodLine = new ShippingInformation();

			if (req.getSession().getAttribute("PRODUCTLOADED") != null) {
				shipping = new Shipping();
				shipping.setHandlingCost(shippingInfo.getHandlingCost());
				shipping.setShippingCost(shippingInfo.getShippingCost());
				shipping.setShippingDescription(shippingInfo
						.getShippingMethod());
				shipping.setShippingModule(shippingInfo.getShippingModule());
				req.getSession().removeAttribute("PRODUCTLOADED");

			} else {

				shippingInfo.setShippingCostText(CurrencyUtil
						.displayFormatedAmountWithCurrency(new BigDecimal("0"),
								store.getCurrency()));
				shippingInfo.setShippingMethodId(null);
				shippingInfo.setShippingMethod(null);
				shippingInfo.setShippingCost(new BigDecimal("0"));
				try {
					SessionUtil.removeShippingInformation(req);
				} catch (Exception e) {
					log.error(e);
				}

			}

		} else { // retreive shipping info in http session
			shipping = new Shipping();
			Map shippingOptionsMap = SessionUtil.getShippingMethods(req);
			String method = shippingMethodLine.getShippingMethodId();

			if (shippingInfo.getShippingCost() != null
					&& shippingInfo.getShippingMethod() != null) {

				shipping.setHandlingCost(shippingInfo.getHandlingCost());
				shipping.setShippingCost(shippingInfo.getShippingCost());
				shipping.setShippingDescription(shippingInfo
						.getShippingMethod());
				shipping.setShippingModule(shippingInfo.getShippingModule());

			} else {

				if (shippingOptionsMap == null || method == null) {
					shippingMethodLine.setShippingCostText(CurrencyUtil
							.displayFormatedAmountWithCurrency(new BigDecimal(
									"0"), store.getCurrency()));
					// total.setShippingLine(shippingMethodLine);
					shippingInfo = shippingMethodLine;
				} else {// after a selection
					// retreive shipping option
					ShippingOption option = (ShippingOption) shippingOptionsMap
							.get(method);

					// get the latest shipping information (handling, free ...)

					shippingInfo.setShippingMethodId(option.getOptionId());
					shippingInfo.setShippingOptionSelected(option);
					shippingInfo.setShippingMethod(option.getDescription());

					shippingInfo.setShippingCost(option.getOptionPrice());
					shippingInfo.setShippingModule(option.getModule());

					shipping.setHandlingCost(shippingInfo.getHandlingCost());
					shipping.setShippingCost(shippingInfo.getShippingCost());
					shipping.setShippingDescription(option.getDescription());
					shipping.setShippingModule(option.getModule());

					// total.setShipping(true);

				}

			}
		}

		List productList = new ArrayList();

		try {

			// validate numeric quantity

			// validate numeric price
			if (products != null) {

				// get products from httpsession
				Map savedOrderProducts = SessionUtil.getOrderProducts(req);
				Map currentProducts = new HashMap();

				if (savedOrderProducts == null) {
					savedOrderProducts = SessionUtil
							.createSavedOrderProducts(req);
				}



				total.setOrderProducts(products);

				if (order == null) {
					log.error("No order exist for the price calculation");
					total.setErrorMessage(LabelUtil.getInstance().getText(
							req.getLocale(), "messages.genericmessage"));
					return total;
				}

				// validates amounts
				BigDecimal oneTimeSubTotal = total.getOneTimeSubTotal();

				List prdscart = null;
				ShoppingCart cart = SessionUtil.getMiniShoppingCart(req);

				for (int i = 0; i < products.length; i++) {
					//get product submited
					OrderProduct product = products[i];

					currentProducts.put(String.valueOf(product.getLineId()),
							product);

					// get the original line
					OrderProduct oproduct = (OrderProduct) savedOrderProducts
							.get(String.valueOf(product.getLineId()));
					
					oproduct.setPriceErrorMessage(null);
					oproduct.setErrorMessage(null);

					productList.add(oproduct);

					// check that productid match
					if (product.getProductId() != oproduct.getProductId()) {// set
																			// an
																			// error
																			// message
						oproduct.setErrorMessage(LabelUtil.getInstance()
								.getText(req.getLocale(),
										"messages.invoice.product.invalid"));
						//oproduct.setPriceText("0");
						//oproduct.setProductPrice(new BigDecimal(0));
						oproduct
								.setPriceFormated(CurrencyUtil
										.displayFormatedAmountWithCurrency(
												new BigDecimal(0), store
														.getCurrency()));
						continue;
					}
					
					//validate if quantity is valid
					if(product.getProductQuantity()>oproduct.getProductQuantityOrderMax()) {
						
						oproduct.setErrorMessage(LabelUtil.getInstance()
								.getText(req.getLocale(),
										"messages.invalid.quantity"));
						product.setProductQuantity(product.getProductQuantityOrderMax());
						continue;
						
					}
					

					// validate and set the final price
					try {
						product.setPriceErrorMessage(null);// reset any error
															// message
						product.setErrorMessage(null);
						// set price submited

						BigDecimal price = oproduct.getProductPrice();

						oproduct.setPriceText(product.getPriceText());
						oproduct.setProductPrice(price);
						oproduct.setProductQuantity(product
								.getProductQuantity());


						double finalPrice = price.doubleValue()
								* product.getProductQuantity();
						BigDecimal bdFinalPrice = new BigDecimal(finalPrice);

						// price calculated @todo can remove, use priceFormated
						oproduct.setCostText(CurrencyUtil
								.displayFormatedAmountWithCurrency(
										bdFinalPrice, store.getCurrency()));
						oproduct.setPriceFormated(CurrencyUtil
								.displayFormatedAmountWithCurrency(
										bdFinalPrice, store.getCurrency()));
						// final price is price * quantity
						oproduct.setFinalPrice(bdFinalPrice);

					} catch (NumberFormatException nfe) {
						oproduct.setPriceErrorMessage(LabelUtil.getInstance()
								.getText(req.getLocale(),
										"messages.price.invalid"));
						oproduct.setPriceText("0");
						oproduct.setProductPrice(new BigDecimal(0));
						oproduct
								.setCostText(CurrencyUtil
										.displayFormatedAmountWithCurrency(
												new BigDecimal(0), store
														.getCurrency()));
						oproduct
								.setPriceFormated(CurrencyUtil
										.displayFormatedAmountWithCurrency(
												new BigDecimal(0), store
														.getCurrency()));
						// set shipping to 0
						ShippingInformation info = new ShippingInformation();
						shippingMethodLine.setShippingCostText(CurrencyUtil
								.displayFormatedAmountWithCurrency(
										new BigDecimal("0"), store
												.getCurrency()));
						total.setShippingLine(info);
						total.setShippingTotal(new BigDecimal("0"));

					}

					// check mini cart products and adjust quantity

					if (cart != null) {
						Collection prds = cart.getProducts();
						if (prds != null) {
							Iterator iprd = prds.iterator();
							while (iprd.hasNext()) {
								ShoppingCartProduct scp = (ShoppingCartProduct) iprd
										.next();
								if (scp.getMainCartLine()!=null && scp.getMainCartLine()
										.equals(
												String.valueOf(products[i]
														.getLineId()))) {
									scp.setQuantity(products[i]
											.getProductQuantity());
								}
							}
						}
					}

				}

				// save mini cart
				if (cart != null) {
					MiniShoppingCartUtil.calculateTotal(cart, store);
				}

				List removable = null;
				// cleanup http session
				Iterator it = savedOrderProducts.keySet().iterator();
				while (it.hasNext()) {
					String key = (String) it.next();
					if (!currentProducts.containsKey(key)) {
						if (removable == null) {
							removable = new ArrayList();
						}
						removable.add(key);
					}
				}

				if (removable != null) {
					Iterator removIt = removable.iterator();
					while (removIt.hasNext()) {
						String key = (String) removIt.next();
						SessionUtil.removeOrderTotalLine(key, req);
					}
				}

				OrderService oservice = (OrderService) ServiceFactory
						.getService(ServiceFactory.OrderService);
				total = oservice.calculateTotal(order, productList, customer,
						shipping, store.getCurrency(), LocaleUtil
								.getLocale(req));

				OrderProduct[] opArray = new OrderProduct[productList.size()];
				OrderProduct[] o = (OrderProduct[]) productList
						.toArray(opArray);
				total.setOrderProducts(o);

				total.setShippingLine(shippingInfo);

				Order savedOrder = SessionUtil.getOrder(req);
				savedOrder.setTotal(total.getTotal());
				savedOrder.setOrderTax(total.getTaxTotal());
				savedOrder.setRecursiveAmount(total.getRecursiveSubTotal());

				SessionUtil.setOrder(savedOrder, req);

				Map totals = OrderUtil.getOrderTotals(order.getOrderId(),
						total, store.getCurrency(), LocaleUtil.getLocale(req));

				// transform totals to a list
				List totalsList = new ArrayList();
				if (totals != null) {
					Iterator totalsIterator = totals.keySet().iterator();
					while (totalsIterator.hasNext()) {
						String key = (String) totalsIterator.next();
						OrderTotal t = (OrderTotal) totals.get(key);
						totalsList.add(t);
					}
				}

				SessionUtil.setOrderTotals(totalsList, req);

			}

		} catch (Exception e) {
			log.error(e);
			total = new OrderTotalSummary(store.getCurrency());
			total.setErrorMessage(LabelUtil.getInstance().getText(
					req.getLocale(), "messages.genericmessage"));
		}

		ShippingInformation shippingLine = total.getShippingLine();
		if (shippingLine != null) {
			shippingLine.setShippingCostText(CurrencyUtil
					.displayFormatedAmountWithCurrency(shippingLine
							.getShippingCost(), store.getCurrency()));
		} else {
			shippingLine = new ShippingInformation();
			shippingLine.setShippingCostText(CurrencyUtil
					.displayFormatedAmountWithCurrency(new BigDecimal("0"),
							store.getCurrency()));
		}

		if (shippingLine.getHandlingCost() != null) {
			shippingLine.setHandlingCostText(CurrencyUtil
					.displayFormatedAmountWithCurrency(shippingMethodLine
							.getHandlingCost(), store.getCurrency()));
		}

		if (total.getShippingTotal() != null) {
			total.setShippingTotalText(CurrencyUtil
					.displayFormatedAmountWithCurrency(
							total.getShippingTotal(), store.getCurrency()));
		}

		if (total.getOneTimeSubTotal() != null) {
			total.setOneTimeSubTotalText(CurrencyUtil
					.displayFormatedAmountWithCurrency(total
							.getOneTimeSubTotal(), store.getCurrency()));
		}

		if (total.getRecursiveSubTotal() != null) {
			total.setRecursiveSubTotalText(CurrencyUtil
					.displayFormatedAmountWithCurrency(total
							.getRecursiveSubTotal(), store.getCurrency()));
		}

		if (total.getTotal() != null) {
			total.setTotalText(CurrencyUtil.displayFormatedAmountWithCurrency(
					total.getTotal(), store.getCurrency()));
		}

		synchronizeProductList(req);

		return total;

	}

}


```
