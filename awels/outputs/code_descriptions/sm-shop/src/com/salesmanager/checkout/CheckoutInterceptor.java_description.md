# CheckoutInterceptor.java

## Review

## 1. Summary

`CheckoutInterceptor` is an **abstract Struts 2 interceptor** that prepares the HTTP request with all the objects required for the checkout pages of the SalesManager application.  
It pulls data from the session (merchant store, customer, order, totals, etc.), enriches the request with analytics settings, and then hands control over to concrete subclasses via the abstract `doIntercept` method.  

Key responsibilities  
* Retrieve and expose the `MerchantStore`, `Customer`, `Order`, totals, and status history in request attributes.  
* Resolve analytics configuration for the current merchant.  
* Handle any exception that occurs during the preparation phase and redirect to a generic error page.

The class leverages the following patterns/libraries:  
* **Template Method** – `baseIntercept` performs common work while delegating the final decision to `doIntercept`.  
* **Service Locator** – `ServiceFactory.getService` is used to obtain the `MerchantService`.  
* **Session Utility** – `SessionUtil` encapsulates session‐state extraction logic.  
* **Struts 2 Interceptor API** – `ActionInvocation`, `ActionSupport`, etc.

---

## 2. Detailed Description

### Flow of Execution

1. **Initialization** – The interceptor has empty `init`/`destroy` hooks (inherited from `SalesManagerInterceptor`).  
2. **Request Decoration** – `baseIntercept` is invoked for every request that matches the interceptor’s configuration.  
   * The current `MerchantStore` is fetched from the session and attached to the request as `STORE`.  
   * If a store exists, its analytics configuration is retrieved via `MerchantService` and exposed as `ANALYTICS`.  
   * The current `Customer`, the list of order products, order totals, the `Order` itself, and the latest `OrderStatusHistory` are all retrieved from the session and added to the request under the keys `CUSTOMER`, `ORDERPRODUCTS`, `TOTALS`, `ORDER`, and `HISTORY`.  
3. **Delegate** – The abstract method `doIntercept` is called, allowing subclasses to add business‑specific logic or alter the flow.  
4. **Exception Handling** – Any exception thrown in the above steps is caught; an error is logged, an action error is added, and the result string `"GENERICERROR"` is returned.

### Dependencies & Constraints

* **Thread safety** – All data is stored in the request scope, so the interceptor is thread‑safe for the typical servlet environment.  
* **Session state** – The interceptor assumes that `SessionUtil` correctly returns non‑null objects for the current user; if the session has expired, many attributes will be `null`.  
* **Service availability** – It relies on a correctly configured `ServiceFactory` that can supply a `MerchantService` instance.  
* **Struts 2** – Requires a Struts 2 action to be an instance of `ActionSupport` (or a subclass) because of the cast in the exception block.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `destroy()` | Interceptor lifecycle hook – currently no cleanup logic. | – | – | None |
| `init()` | Interceptor lifecycle hook – currently no init logic. | – | – | None |
| `baseIntercept(ActionInvocation invoke, HttpServletRequest req, HttpServletResponse resp)` | Core interception logic: populates request attributes, handles analytics, delegates to `doIntercept`, and handles errors. | *invoke* – the Struts 2 action invocation<br>*req* – current HTTP request<br>*resp* – current HTTP response | `String` – the result name for Struts 2 | Sets numerous request attributes; logs errors; may throw checked exception |
| `doIntercept(ActionInvocation invoke, HttpServletRequest req, HttpServletResponse resp)` | Abstract hook for subclasses to implement custom interception logic. | Same as `baseIntercept` | `String` – result name | Subclasses decide the outcome |

### Reusable / Utility Methods

* There are no explicit utility methods in this class; however, the entire request‑population logic could be extracted into a protected helper method to reduce duplication in subclasses.

---

## 4. Dependencies

| External | Type | Notes |
|----------|------|-------|
| `org.apache.log4j.Logger` | Third‑party logging | Classic Log4j; modern applications might prefer SLF4J + Logback. |
| `com.opensymphony.xwork2.*` | Struts 2 core | Provides the interceptor API and action support. |
| `com.salesmanager.core.*` | Internal | Service factory, entity classes, and session utilities. |
| `javax.servlet.http.*` | Java EE | Standard servlet API. |
| `java.util.*` | Standard | Raw collections used – should be parameterized. |

All dependencies are either standard Java EE APIs or internal project libraries, so there are no external network or platform constraints beyond a typical servlet container.

---

## 5. Additional Notes & Recommendations

### 5.1 Bug: Incorrect Request Attribute for Order Products

```java
List prds = new ArrayList();
prds.addAll(products.entrySet());
req.setAttribute("ORDERPRODUCTS", customer);   // ← wrong value
```

The attribute is mistakenly set to the `Customer` object instead of the `prds` list (or the original `products` map).  
**Fix**: `req.setAttribute("ORDERPRODUCTS", prds);` or simply `req.setAttribute("ORDERPRODUCTS", products);`

### 5.2 Use of Raw Types

The code uses raw collections (`Map`, `List`, `Collection`) which bypasses compile‑time type safety.  
**Recommendation**: Replace with generics, e.g.:

```java
Map<Long, OrderProduct> products = SessionUtil.getOrderProducts(req);
List<Map.Entry<Long, OrderProduct>> prds = new ArrayList<>(products.entrySet());
```

### 5.3 Exception Casting to `ActionSupport`

```java
ActionSupport action = (ActionSupport) invoke.getAction();
```

If the action does not extend `ActionSupport`, a `ClassCastException` will occur.  
**Mitigation**: Check `instanceof` before casting or design the interceptor to work with a more generic interface (`com.opensymphony.xwork2.Action`).

### 5.4 Logging Practices

`log.error(e)` prints the stack trace but not a descriptive message.  
**Suggestion**: `log.error("Error during checkout interception", e);`

### 5.5 Service Locator vs Dependency Injection

Obtaining `MerchantService` via a static `ServiceFactory` couples the interceptor to the service locator pattern.  
**Future enhancement**: Inject the service (e.g., via Spring) to improve testability and flexibility.

### 5.6 Method Visibility

`baseIntercept` is `protected`, but it is the public entry point used by Struts 2.  
Consider marking it `public` or documenting that the framework invokes it via reflection.

### 5.7 Documentation

No Javadoc comments are present. Adding class‑level and method‑level Javadoc would greatly aid maintainability, especially for the abstract method contract.

### 5.8 Thread Safety & Reuse

The interceptor is stateless (apart from the logger) and safe for concurrent use. No changes needed.

### 5.9 Return Value Conventions

The method returns `"GENERICERROR"` on failure, which assumes that this result name is mapped in the Struts 2 configuration. Ensure that all actions that can be intercepted have this result defined.

---

### Summary of Actions to Take

| Item | Action | Priority |
|------|--------|----------|
| Fix ORDERPRODUCTS attribute bug | Update `req.setAttribute` | High |
| Parameterize collections | Replace raw types with generics | Medium |
| Defensive casting | Add `instanceof` check or use generic `Action` | Medium |
| Logging | Enhance error message | Low |
| Refactor service lookup | Introduce DI or expose setter | Low |
| Add Javadoc | Document public API | Low |

Implementing these changes will make the interceptor more robust, maintainable, and easier to test.

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
package com.salesmanager.checkout;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import org.apache.log4j.Logger;
import com.opensymphony.xwork2.ActionInvocation;
import com.opensymphony.xwork2.ActionSupport;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.www.SalesManagerInterceptor;
import com.salesmanager.core.util.www.SessionUtil;

public abstract class CheckoutInterceptor extends SalesManagerInterceptor {

	private Logger log = Logger.getLogger(CheckoutInterceptor.class);

	public void destroy() {
		// TODO Auto-generated method stub

	}

	public void init() {
		// TODO Auto-generated method stub

	}

	protected String baseIntercept(ActionInvocation invoke,
			HttpServletRequest req, HttpServletResponse resp) throws Exception {

		try {


			MerchantStore store = SessionUtil.getMerchantStore(req);
			req.setAttribute("STORE", store);
			
			
			// get analytics
			if(store!=null) {
				ConfigurationRequest request = new ConfigurationRequest(store
						.getMerchantId());// get all configurations
				MerchantService mservice = (MerchantService) ServiceFactory
						.getService(ServiceFactory.MerchantService);
				ConfigurationResponse vo = mservice.getConfiguration(request);
	
				if (vo != null) {
					MerchantConfiguration merchantConfiguration = vo
							.getMerchantConfiguration(ConfigurationConstants.G_API);
					if (merchantConfiguration != null) {
						String analytics = merchantConfiguration
								.getConfigurationValue();
						req.setAttribute("ANALYTICS", analytics);
					}
	
				}
			}
			

			// set objects in http request
			Customer customer = SessionUtil.getCustomer(req);
			req.setAttribute("CUSTOMER", customer);

			Map products = SessionUtil.getOrderProducts(req);
			if (products != null) {
				List prds = new ArrayList();
				prds.addAll(products.entrySet());
				req.setAttribute("ORDERPRODUCTS", customer);
			}

			Collection totals = SessionUtil.getOrderTotals(req);
			req.setAttribute("TOTALS", totals);

			Order order = SessionUtil.getOrder(req);
			req.setAttribute("ORDER", order);

			OrderStatusHistory comments = SessionUtil
					.getOrderStatusHistory(req);
			req.setAttribute("HISTORY", comments);
			
			String r = doIntercept(invoke, req, resp);
			return r;


		} catch (Exception e) {

			log.error(e);
			ActionSupport action = (ActionSupport) invoke.getAction();
			action.addActionError(action.getText("errors.technical") + " "
					+ e.getMessage());
			return "GENERICERROR";

		}

	}
	
	protected abstract String doIntercept(ActionInvocation invoke,
			HttpServletRequest req, HttpServletResponse resp) throws Exception;

}



```
