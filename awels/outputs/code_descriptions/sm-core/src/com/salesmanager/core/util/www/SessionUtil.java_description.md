# SessionUtil.java

## Review

## 1. Summary
`SessionUtil` is a helper class that centralises all interactions with the HTTP session for a SalesManager e‑commerce application.  
It stores and retrieves objects that represent the state of a user’s shopping flow – cart items, order details, payment/ shipping information, customer data, merchant store data, etc. The class is heavily tied to the session scope and uses a number of hard‑coded attribute names.

**Key components**

| Purpose | Attribute key | Typical type |
|---------|---------------|--------------|
| Shopping cart | `"CART"` | `ShoppingCart` |
| Order products | `"ORDERPRODUCTS"` | `Map<String, OrderProduct>` |
| Order product attributes | `"ORDERPODUCTATTRIBUTES"` | `Map` |
| Order | `"ORDER"` | `Order` |
| Customer | `"CUSTOMER"` | `Customer` |
| Merchant store | `MERCHANT_STORE_SESSION_ATTR` | `MerchantStore` |
| Shipping | `"SHIPPINGINFORMATION"` / `"SHIPPINGMETHODS"` | `ShippingInformation` / `Map` |
| Payment | `"PAYMENTMETHOD"` / `"HASPAYMENT"` | `PaymentMethod` / `Boolean` |
| Totals | `"TOTALS"` | `Collection` |
| Order totals summary | `"ORDERRSUMMARY"` | `OrderTotalSummary` |
| Order status history | `"STATUSHISTORY"` | `OrderStatusHistory` |
| Misc flags | `"TOKEN"`, `"COMITED"`, `"HASSHIPPING"`, `"CARTLINE"` | `String` / `Boolean` |

The class is a pure utility (`static` methods only), so no instance state or lifecycle management is required. It relies on the standard servlet API and the `org.apache.log4j.Logger` for debugging.

---

## 2. Detailed Description
`SessionUtil` acts as a façade over the raw `HttpSession` attributes. The overall flow is:

1. **Initialisation** – When a user starts a checkout, the session attributes are either created (via `createSavedOrderProducts`) or cleared (via `resetCart` / `cleanCart`).  
2. **Runtime behaviour** –  
   * Items are added to the cart with `addOrderProduct` which also tracks a `CARTLINE` counter to generate unique line numbers.  
   * Order totals and summaries are stored with `setOrderTotals`, `setOrderTotalSummary`.  
   * Shipping/payment data is set/retrieved via dedicated getters/setters.  
   * Convenience helpers (`getCustomer`, `getMerchantStore`, etc.) expose the stored objects to other layers of the application.  
3. **Cleanup** – The `resetCart` / `cleanCart` methods remove session attributes to end a transaction or abandon a cart.

### Assumptions & Constraints
* **Single user session** – All attributes are tied to one session; concurrency between threads (e.g. async requests) is not considered.  
* **Attribute names are hard‑coded** – Any typo results in a silent failure or data loss.  
* **No type safety** – The class uses raw `Map`/`Collection` types; casting is required on every retrieval, which can lead to `ClassCastException`.  
* **Error handling** – Most methods declare `throws Exception` but never actually throw anything except in `resetProduct`; runtime exceptions propagate without graceful recovery.  
* **No null‑checking for primitives** – Methods like `setHasShipping` wrap a primitive in `new Boolean(shippingState)`, which is unnecessary (autoboxing).  

The architecture is intentionally simple (a static helper) but can become a maintenance burden because it duplicates business logic (e.g., line numbering, attribute cleanup) that would be more cleanly expressed in a dedicated session‑scoped bean or a service.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side effects |
|--------|---------|------------|--------|--------------|
| `createSavedOrderProducts(HttpServletRequest)` | Initialises an empty `Map` for order products and stores it in session under `"ORDERPRODUCTS"`. | `request` | `Map` (raw) | Session attribute set |
| `addOrderTotalLine(OrderProduct, String, HttpServletRequest)` | Adds an `OrderProduct` to the `"ORDERPRODUCTS"` map, assigning a line id. | `product`, `lineId`, `request` | `void` | Modifies session map |
| `resetCart(HttpServletRequest)` | Removes a large set of attributes (mostly those related to an order) from session. | `request` | `void` | Session cleanup |
| `cleanCart(HttpServletRequest)` | Similar to `resetCart` but also removes `"PAYMENTMETHOD"`, `"COMITED"`, `"TOKEN"`. | `request` | `void` | Session cleanup |
| `setToken(HttpServletRequest)` | Puts a dummy `"TOKEN"` attribute (value `"TOKEN"`). | `request` | `void` | Session attribute set |
| `resetProduct(OrderProduct, long, String, HttpServletRequest)` | Updates an existing `OrderProduct`’s quantity, re‑inserting it into session, and clears any associated attributes. | `original`, `productId`, `lineId`, `request` | `OrderProduct` | Session modifications |
| `setMiniShoppingCart(ShoppingCart, HttpServletRequest)` | Stores a `ShoppingCart` in session under `"CART"`. | `cart`, `request` | `void` | Session attribute set |
| `getMiniShoppingCart(HttpServletRequest)` | Retrieves the `ShoppingCart`. | `request` | `ShoppingCart` | None |
| `removeMiniShoppingCart(HttpServletRequest)` | Removes the cart attribute. | `request` | `void` | Session attribute removed |
| `removeOrderTotalLine(String, HttpServletRequest)` | Removes a line from the order products map (and attributes map). | `lineId`, `request` | `void` | Session map mutation |
| `removeShippingInformation(HttpServletRequest)` | Deletes shipping information and methods. | `request` | `void` | Session attribute removal |
| `getOrderProducts(HttpServletRequest)` | Gets the map of order products. | `request` | `Map` | None |
| `getCustomer(HttpServletRequest)` | Retrieves the `Customer`. | `request` | `Customer` | None |
| `getOrderProductAttributes(HttpServletRequest)` | Retrieves order‑product attribute map. | `request` | `Map` | None |
| `setCustomer(Customer, HttpServletRequest)` | Stores the `Customer`. | `customer`, `request` | `void` | Session attribute set |
| `getOrder(HttpServletRequest)` | Retrieves the `Order`. | `request` | `Order` | None |
| `setOrder(Order, HttpServletRequest)` | Stores the `Order`. | `order`, `request` | `void` | Session attribute set |
| `setMerchantStore(MerchantStore, HttpServletRequest)` | Stores the merchant store. | `store`, `request` | `void` | Session attribute set |
| `getMerchantStore(HttpServletRequest)` | Retrieves the merchant store. | `request` | `MerchantStore` | None |
| `getShippingMethods(HttpServletRequest)` | Retrieves shipping methods map. | `request` | `Map` | None |
| `setShippingMethods(Map, HttpServletRequest)` | Stores shipping methods map. | `shippingMethods`, `request` | `void` | Session attribute set |
| `getShippingInformation(HttpServletRequest)` | Retrieves `ShippingInformation`. | `request` | `ShippingInformation` | None |
| `setShippingInformation(ShippingInformation, HttpServletRequest)` | Stores shipping information. | `info`, `request` | `void` | Session attribute set |
| `setHasShipping(boolean, HttpServletRequest)` | Flags that a shipping address exists. | `shippingState`, `request` | `void` | Session attribute set |
| `getIsShipping(HttpServletRequest)` | Returns shipping flag (default `false`). | `request` | `boolean` | None |
| `addOrderProduct(OrderProduct, HttpServletRequest)` | Adds a product to the order map while incrementing the cart line counter. | `product`, `request` | `void` | Session and cart manipulation |
| `setOrderTotals(Collection, HttpServletRequest)` | Stores the collection of totals. | `totals`, `request` | `void` | Session attribute set |
| `getOrderTotals(HttpServletRequest)` | Retrieves the totals collection. | `request` | `Collection` | None |
| `setPaymentMethod(PaymentMethod, HttpServletRequest)` | Stores the chosen payment method. | `method`, `request` | `void` | Session attribute set |
| `getPaymentMethod(HttpServletRequest)` | Retrieves the payment method. | `request` | `PaymentMethod` | None |
| `setHasPayment(boolean, HttpServletRequest)` | Flags that a payment method exists. | `hasPayment`, `request` | `void` | Session attribute set |
| `isHasPayment(HttpServletRequest)` | Checks payment flag (defaults to `true`). | `request` | `boolean` | None |
| `setOrderTotalSummary(OrderTotalSummary, HttpServletRequest)` | Stores the totals summary. | `summary`, `request` | `void` | Session attribute set |
| `getOrderTotalSummary(HttpServletRequest)` | Retrieves the totals summary. | `request` | `OrderTotalSummary` | None |
| `setOrderStatusHistory(OrderStatusHistory, HttpServletRequest)` | Stores status history. | `history`, `request` | `void` | Session attribute set |
| `getOrderStatusHistory(HttpServletRequest)` | Retrieves status history. | `request` | `OrderStatusHistory` | None |
| `setComited(HttpServletRequest)` | Flags that the order has been committed. | `request` | `void` | Session attribute set |
| `isComited(HttpServletRequest)` | Checks commit flag. | `request` | `boolean` | None |

**Reusable/Utility Methods**  
- `createSavedOrderProducts` and `getOrderProducts` are the primary “factory” and “accessor” for the order products map.  
- `resetCart` / `cleanCart` are used to clear state; they could be factored into a single method that accepts a list of attribute names.

---

## 4. Dependencies
| Dependency | Type | Comments |
|------------|------|----------|
| `javax.servlet.http.HttpServletRequest / HttpSession` | Standard Servlet API | Required for session handling |
| `org.apache.log4j.Logger` | Third‑party | Simple logging; could be replaced by SLF4J or java.util.logging |
| Domain classes (`Customer`, `MerchantStore`, `Order`, etc.) | Third‑party | Part of the SalesManager domain model |
| No external configuration files or frameworks (e.g. Spring) | — | The class is completely self‑contained |

The only platform‑specific assumption is that the servlet container supports the standard session API (virtually all Java EE / Jakarta EE containers do).

---

## 5. Additional Notes & Recommendations

### 5.1. Raw Types & Type Safety
The class uses raw `Map` and `Collection` everywhere, forcing casts on retrieval. Modern Java should use generics:

```java
Map<String, OrderProduct> orderProducts = 
    (Map<String, OrderProduct>) session.getAttribute("ORDERPRODUCTS");
```

would become:

```java
Map<String, OrderProduct> orderProducts = 
    (Map<String, OrderProduct>) session.getAttribute("ORDERPRODUCTS");
```

or better, expose type‑safe methods:

```java
public static Map<String, OrderProduct> getOrderProducts(HttpServletRequest request) {
    return (Map<String, OrderProduct>) request.getSession()
        .getAttribute("ORDERPRODUCTS");
}
```

### 5.2. Hard‑coded Attribute Names
Using string literals scattered throughout increases the risk of typos (e.g., `"ORDERPODUCTATTRIBUTES"` vs. `"ORDERPRODUCTATTRIBUTES"`). Centralise all attribute names as constants and reference them exclusively.

### 5.3. Session Concurrency
The session may be accessed concurrently (e.g., AJAX checkout steps). The map mutations are not synchronized, which could lead to lost updates. Consider using `ConcurrentHashMap` or synchronising on the session object.

### 5.4. Error Handling
Only `resetProduct` throws a checked `Exception`; the rest swallow or propagate runtime exceptions silently. A better approach would be to:

* Validate input parameters (non‑null, positive IDs).  
* Return meaningful status codes or throw custom unchecked exceptions (e.g., `SessionAttributeNotFoundException`).  

### 5.5. Unused or Redundant Code
* `setToken` stores a constant `"TOKEN"` string—its purpose is unclear.  
* The constructor‐style `new Boolean(shippingState)` is unnecessary; autoboxing handles this.  
* Several `remove*` methods duplicate logic; a single `clearSessionAttributes(String... keys)` helper would reduce duplication.

### 5.6. Naming & Documentation
* Method names mix camelCase and all‑caps (e.g., `MERRCHANT_STORE_SESSION_ATTR`).  
* Javadoc comments are minimal; documenting each method’s contract, especially the session keys it uses, would improve maintainability.

### 5.7. Potential Edge Cases
* **Missing session**: If `request.getSession(false)` is called (not present in this code) the attribute retrieval could return `null`.  
* **Large cart**: Storing all cart lines in session may consume significant memory for heavy users; consider persisting to a DB or using a cache.  
* **Line number overflow**: `CARTLINE` is stored as a string of an integer; after many additions it could exceed `Integer.MAX_VALUE`, though unlikely.

### 5.8. Future Enhancements
1. **Refactor into a session‑scoped bean** (e.g., in Spring MVC) to encapsulate state and eliminate static access.  
2. **Introduce a SessionManager interface** that could be swapped with a distributed cache or a database.  
3. **Move attribute names into an enum** to provide compile‑time safety.  
4. **Add unit tests** for each method, mocking `HttpServletRequest`/`HttpSession`.  
5. **Use SLF4J** for logging to decouple from Log4j.  
6. **Persist critical data** (order totals, payment method) early to avoid data loss on session invalidation.

---

### 5.9. Summary of Strengths
* Centralised session handling reduces code duplication.  
* Clear separation of concerns: cart, order, customer, shipping, payment.  
* Uses standard Java EE APIs, making it portable across servlet containers.

### 5.10. Summary of Weaknesses
* Heavy reliance on raw types and magic strings.  
* Lack of type safety and input validation.  
* Redundant and duplicated code across `resetCart`/`cleanCart`.  
* No concurrency handling for session attributes.  
* Outdated coding style (explicit `new Boolean`, string constants with typos).

Addressing these issues would significantly improve maintainability, robustness, and testability of the session‑management layer.

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
package com.salesmanager.core.util.www;

import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.log4j.Logger;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.orders.ShoppingCart;
import com.salesmanager.core.entity.orders.ShoppingCartProduct;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.shipping.ShippingInformation;

public class SessionUtil {

	public static final String MERCHANT_STORE_SESSION_ATTR = "STORE";
	public static final String ORDER_PRODUCT_LIST_SESSION_ATTR = "ORDER_PRODUCT_LIST";

	private static Logger log = Logger.getLogger(SessionUtil.class);

	public static Map createSavedOrderProducts(HttpServletRequest request) {

		HttpSession session = request.getSession();
		Map cartLines = new HashMap();
		session.setAttribute("ORDERPRODUCTS", cartLines);
		return cartLines;

	}

	/**
	 * Stores the shopping cart product in the HttpSession 1- Check if the
	 * product is already in the session using the lineId
	 * 
	 */
	public static void addOrderTotalLine(OrderProduct product, String lineId,
			HttpServletRequest request) throws Exception {

		// check if the product is already in the HttpSession

		HttpSession session = request.getSession();

		Map cartLines = (Map) session.getAttribute("ORDERPRODUCTS");

		if (cartLines == null) {
			cartLines = createSavedOrderProducts(request);
		}

		product.setLineId(Integer.parseInt(lineId));

		cartLines.put(lineId, product);

		/**
		 * Map opaMap = (Map)session.getAttribute("ORDERPODUCTATTRIBUTES");
		 * 
		 * if(opaMap==null) { opaMap = new HashMap();
		 * session.setAttribute("ORDERPODUCTATTRIBUTES", opaMap);
		 * 
		 * }
		 **/

	}

	public static void resetCart(HttpServletRequest request) throws Exception {

		// request.getSession().removeAttribute("ORDER");//and payment method
		// are not removed
		// request.getSession().removeAttribute("CUSTOMER");
		request.getSession().removeAttribute("ORDERPRODUCTS");
		request.getSession().removeAttribute("ORDERPODUCTATTRIBUTES");
		request.getSession().removeAttribute("SHIPPINGMETHODS");
		request.getSession().removeAttribute("SHIPPINGINFORMATION");
		request.getSession().removeAttribute("TOTALS");
		request.getSession().removeAttribute("ORDER_PRODUCT_LIST");
		request.getSession().removeAttribute("CARTLINE");
		request.getSession().removeAttribute("TOTALS");
		request.getSession().removeAttribute("HASPAYMENT");
		request.getSession().removeAttribute("MERRCHANT_STORE_SESSION_ATTR");
		request.getSession().removeAttribute("LOGGEDINCUSTOMER");
		request.getSession().removeAttribute("STATUSHISTORY");
		request.getSession().removeAttribute("CART");

	}

	public static void cleanCart(HttpServletRequest request) throws Exception {

		request.getSession().removeAttribute("ORDER");
		// request.getSession().removeAttribute("CUSTOMER");
		request.getSession().removeAttribute("ORDERPRODUCTS");
		request.getSession().removeAttribute("ORDERPODUCTATTRIBUTES");
		request.getSession().removeAttribute("SHIPPINGMETHODS");
		request.getSession().removeAttribute("SHIPPINGINFORMATION");
		request.getSession().removeAttribute("TOTALS");
		request.getSession().removeAttribute("ORDER_PRODUCT_LIST");
		request.getSession().removeAttribute("CARTLINE");
		request.getSession().removeAttribute("TOTALS");
		request.getSession().removeAttribute("PAYMENTMETHOD");
		request.getSession().removeAttribute("HASPAYMENT");
		request.getSession().removeAttribute("MERRCHANT_STORE_SESSION_ATTR");
		request.getSession().removeAttribute("LOGGEDINCUSTOMER");
		request.getSession().removeAttribute("STATUSHISTORY");
		request.getSession().removeAttribute("COMITED");
		request.getSession().removeAttribute("TOKEN");

	}

	public static void setToken(HttpServletRequest request) {
		request.getSession().setAttribute("TOKEN", "TOKEN");
	}

	public static OrderProduct resetProduct(OrderProduct original,
			long productId, String lineId, HttpServletRequest request)
			throws Exception {

		HttpSession session = request.getSession();

		Map cartLines = (Map) session.getAttribute("ORDERPRODUCTS");

		if (cartLines == null) {
			throw new Exception(
					"No OrderProduct exixt yet, cannot assign attributes");
		}

		OrderProduct scp = (OrderProduct) cartLines.get(lineId);

		if (scp == null) {
			throw new Exception("No OrderProduct exixt for lineId " + lineId);
		}

		original.setProductQuantity(scp.getProductQuantity());

		cartLines.put(lineId, original);

		Map opaMap = (Map) session.getAttribute("ORDERPODUCTATTRIBUTES");

		if (opaMap != null) {
			opaMap.remove(lineId);
		}

		return original;

	}

	public static void setMiniShoppingCart(ShoppingCart cart,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("CART", cart);
	}

	public static ShoppingCart getMiniShoppingCart(HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (ShoppingCart) session.getAttribute("CART");
	}
	
	public static void removeMiniShoppingCart(HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.removeAttribute("CART");
	}

	public static void removeOrderTotalLine(String lineId,
			HttpServletRequest request) throws Exception {

		HttpSession session = request.getSession();

		Map products = (Map) session.getAttribute("ORDERPRODUCTS");
		if (products != null) {
			products.remove(lineId);

		}

		ShoppingCart cart = SessionUtil.getMiniShoppingCart(request);
		if (cart != null) {

		}

		Map opaMap = (Map) session.getAttribute("ORDERPODUCTATTRIBUTES");

		if (opaMap != null) {
			opaMap.remove(lineId);
		}

	}

	public static void removeShippingInformation(HttpServletRequest request)
			throws Exception {

		HttpSession session = request.getSession();

		session.removeAttribute("SHIPPINGINFORMATION");
		session.removeAttribute("SHIPPINGMETHODS");

	}

	public static Map getOrderProducts(HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (Map) session.getAttribute("ORDERPRODUCTS");
	}

	public static Customer getCustomer(HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (Customer) session.getAttribute("CUSTOMER");
	}

	public static Map getOrderProductAttributes(HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (Map) session.getAttribute("ORDERPODUCTATTRIBUTES");
	}

	public static void setCustomer(Customer customer, HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("CUSTOMER", customer);
	}

	public static Order getOrder(HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (Order) session.getAttribute("ORDER");
	}

	public static void setOrder(Order order, HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("ORDER", order);
	}

	public static void setMerchantStore(MerchantStore store,
			HttpServletRequest request) {
		request.getSession().setAttribute(MERCHANT_STORE_SESSION_ATTR, store);
	}

	public static MerchantStore getMerchantStore(HttpServletRequest request) {
		return (MerchantStore) request.getSession().getAttribute(
				MERCHANT_STORE_SESSION_ATTR);
	}

	public static Map getShippingMethods(HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (Map) session.getAttribute("SHIPPINGMETHODS");
	}

	public static void setShippingMethods(Map shippingMethods,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("SHIPPINGMETHODS", shippingMethods);
	}

	public static ShippingInformation getShippingInformation(
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (ShippingInformation) session
				.getAttribute("SHIPPINGINFORMATION");
	}

	public static void setShippingInformation(
			ShippingInformation shippingInformation, HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("SHIPPINGINFORMATION", shippingInformation);
	}

	public static void setHasShipping(boolean shippingState,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("HASSHIPPING", new Boolean(shippingState));
	}

	public static boolean getIsShipping(HttpServletRequest request) {
		HttpSession session = request.getSession();
		Boolean hasShipping = (Boolean) session.getAttribute("HASSHIPPING");
		if (hasShipping != null) {
			return hasShipping.booleanValue();
		} else {
			return false;
		}
	}

	public static void addOrderProduct(OrderProduct product,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		Map orderProducts = getOrderProducts(request);
		String line = "0";
		if (orderProducts != null && orderProducts.size() > 0) {
			// get the current line
			String currentLine = (String) request.getSession().getAttribute(
					"CARTLINE");
			if (currentLine != null) {
				try {
					int iline = Integer.parseInt(currentLine);
					iline = iline + 1;
					line = String.valueOf(iline);
					request.getSession().setAttribute("CARTLINE", line);

				} catch (Exception e) {
					log.error("Cannot set cartline ", e);
				}
			}

		} else {
			orderProducts = new HashMap();
			int iline = Integer.parseInt(line);
			iline = iline + 1;
			line = String.valueOf(iline);
			request.getSession().setAttribute("CARTLINE", line);

		}

		ShoppingCart cart = SessionUtil.getMiniShoppingCart(request);
		if (cart != null) {
			Collection prds = cart.getProducts();
			if (prds != null) {
				Iterator iprd = prds.iterator();
				while (iprd.hasNext()) {
					ShoppingCartProduct scp = (ShoppingCartProduct) iprd.next();
					if (scp.getProductId() == product.getProductId()) {
						scp.setMainCartLine(line);
						break;
					}
				}
			}
		}

		product.setLineId(Integer.valueOf(line));
		orderProducts.put(line, product);
		session.setAttribute("ORDERPRODUCTS", orderProducts);

		request.getSession().setAttribute("CARTLINE", line);

	}

	public static void setOrderTotals(Collection totals,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("TOTALS", totals);
	}

	public static Collection getOrderTotals(HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (Collection) session.getAttribute("TOTALS");
	}

	public static void setPaymentMethod(PaymentMethod method,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("PAYMENTMETHOD", method);
	}

	public static PaymentMethod getPaymentMethod(HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (PaymentMethod) session.getAttribute("PAYMENTMETHOD");
	}

	public static void setHasPayment(boolean hasPayment,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("HASPAYMENT", hasPayment);
	}

	// public static void setLoggedInCustomer(HttpServletRequest
	// request,Customer customer) {
	// HttpSession session = request.getSession();
	// session.setAttribute("LOGGEDINCUSTOMER",customer);
	// }

	// public static Customer getLoggedInCustomer(HttpServletRequest request) {
	// HttpSession session = request.getSession();
	// return (Customer)session.getAttribute("LOGGEDINCUSTOMER");
	// }

	public static boolean isHasPayment(HttpServletRequest request) {
		HttpSession session = request.getSession();
		Boolean bPayment = (Boolean) session.getAttribute("HASPAYMENT");
		if (bPayment == null) {
			bPayment = true;
		}
		return bPayment;
	}

	public static void setOrderTotalSummary(OrderTotalSummary summary,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("ORDERRSUMMARY", summary);
	}

	public static OrderTotalSummary getOrderTotalSummary(
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (OrderTotalSummary) session.getAttribute("ORDERRSUMMARY");
	}

	public static void setOrderStatusHistory(OrderStatusHistory history,
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("STATUSHISTORY", history);
	}

	public static OrderStatusHistory getOrderStatusHistory(
			HttpServletRequest request) {
		HttpSession session = request.getSession();
		return (OrderStatusHistory) session.getAttribute("STATUSHISTORY");
	}

	public static void setComited(HttpServletRequest request) {
		HttpSession session = request.getSession();
		session.setAttribute("COMITED", "TRUE");
	}

	public static boolean isComited(HttpServletRequest request) {
		HttpSession session = request.getSession();
		if (session.getAttribute("COMITED") != null) {
			return true;
		} else {
			return false;
		}
	}

}


```
