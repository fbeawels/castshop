# DisplayOrderSummaryAction.java

## Review

## 1. Summary  
`DisplayOrderSummaryAction` is an action class used in the checkout flow of a sales‑manager web application.  
Its primary purpose is to prepare the data that will be shown on the order‑summary page:

| Component | Role |
|-----------|------|
| `displayOrderSummaryAction()` | Builds a summary of the current order (products, shipping, payment, taxes) and stores it in the request. |
| `displayCartOrderSummaryAction()` | A lightweight variant that simply sets a request attribute for the view. |
| `Order`, `OrderStatusHistory` | Internal state kept by the action (populated from the session). |
| `OrderService` | External service used to update order totals. |
| `SessionUtil` | Helper for pulling session‑level objects (order, store, customer, etc.). |

The class is tightly coupled to the MVC framework used by the project (likely Struts or a similar Java web framework) and relies heavily on request/session attributes for data passing between layers.  

Notable patterns:  
* **Command/Action pattern** – the class represents a single user request.  
* **Service locator** – obtains `OrderService` via `ServiceFactory`.  
* **Session/Request attribute usage** – passes data across layers through the `HttpServletRequest`.  

## 2. Detailed Description  
### Flow of Execution – `displayOrderSummaryAction()`
1. **Payment Preparation** – `super.preparePayments()` is called, which presumably loads payment information into the session/request.  
2. **Context Setup** – request attributes `STEP` and `ADDRESSTYPE` are initialized.  
3. **Session Retrieval** – `Order`, `MerchantStore`, `Customer`, `PaymentMethod`, order‑products map, and shipping information are all pulled from the session via `SessionUtil`.  
4. **Credit‑Card Handling** – If the payment method is a credit card, its config entry `CARD` is extracted and set on the request.  
5. **Shipping Construction** – If shipping information exists, a `Shipping` object is created and its fields are populated from the selected `ShippingOption`. The request attribute `STEP` is updated to `3` and `ADDRESSTYPE` is set to `BOTH`.  
6. **Product List Building** – The order‑products map (raw type) is iterated to produce a `List<OrderProduct>`.  
7. **Order Totals Update** – `super.updateOrderTotal()` recalculates tax, shipping, and other totals and returns an `OrderTotalSummary`.  
8. **Result** – The summary is stored in the session via `SessionUtil.setOrderTotalSummary()`. On success the action returns `"SUCCESS"`.  
9. **Error Handling** – Any exception is logged and the action returns `"GLOBALERROR"`.

### Dependencies and Assumptions
* All objects retrieved from the session (`Order`, `Customer`, etc.) are expected to be non‑null; the code does **not** guard against `null` for many of them.  
* The `OrderService` is fetched via a static service locator.  
* The framework guarantees a fresh action instance per request (typical for Struts), so instance fields are not shared across users.  
* The `updateOrderTotal()` method is assumed to be side‑effect free (i.e., it does not persist changes), as the code never calls a save/update on the `OrderService`.  
* The request attribute names (`STEP`, `ADDRESSTYPE`, `CARD`, etc.) are hard‑coded and assumed to be known by the view layer.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `displayOrderSummaryAction()` | Builds the full order‑summary view model, updates totals, and sets request/session attributes. | None (uses session). | Sets request attributes; returns `"SUCCESS"` or `"GLOBALERROR"`; logs exceptions. |
| `displayCartOrderSummaryAction()` | Sets a minimal request attribute for a cart‑only summary view. | None. | Sets request attribute `ADDRESSTYPE` to `"BOTH"`; returns `"SUCCESS"`. |
| `getOrder()` | Getter for internal `order` field. | None. | Returns `Order`. |
| `setOrder(Order)` | Setter for internal `order` field. | `Order`. | None. |
| `getOrderHistory()` | Getter for internal `orderHistory` field. | None. | Returns `OrderStatusHistory`. |
| `setOrderHistory(OrderStatusHistory)` | Setter for internal `orderHistory` field. | `OrderStatusHistory`. | None. |

**Reusable/Utility Methods**  
* None – all logic is embedded in the action.  

## 4. Dependencies  
| External Library / API | Usage | Standard / Third‑Party |
|------------------------|-------|------------------------|
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `com.salesmanager.core.*` | Domain entities (`Order`, `Customer`, `PaymentMethod`, etc.) and services | Third‑party (project specific) |
| `com.salesmanager.core.service.ServiceFactory` | Service locator for `OrderService` | Project specific |
| `com.salesmanager.core.util.www.SessionUtil` | Helper to pull and store session objects | Project specific |
| `javax.servlet.http.HttpServletRequest` | Implicit via `CheckoutBaseAction` | Standard |
| `org.apache.struts.action.Action` (implied by action naming) | Web framework base class | Third‑party (Struts) |

No platform‑specific APIs beyond the typical servlet environment.

## 5. Additional Notes  

### Strengths  
* Clear separation of concerns: the action delegates business logic (e.g., updating totals) to the base class and services.  
* Uses request/session attributes consistently for view data.  
* Provides a simple, readable flow for building the summary page.  

### Potential Issues & Edge Cases  
| Issue | Why it matters | Suggested Fix |
|-------|----------------|--------------|
| **Raw Types** – `Map orderProducts`, `List products` | Unchecked casts can lead to `ClassCastException` at runtime. | Use generics: `Map<String, OrderProduct>` and `List<OrderProduct>`. |
| **Null Checks** – `paymentMethod`, `store`, `customer`, `shippingInformation`, `shippingOption` | Any of these being `null` will throw `NullPointerException`. | Add defensive checks; log and fallback to defaults or error pages. |
| **Hard‑coded Request Attributes** – `"STEP"`, `"ADDRESSTYPE"` | Tight coupling between action and view; hard to refactor. | Define constants in a common class or enum. |
| **Exception Handling** – Logs only the exception object. | Stack traces may be truncated. | Log the full stack trace (`log.error("...", e)`). |
| **Thread‑Safety / Action Instance Reuse** – Fields `order` and `orderHistory` are instance variables. | In some frameworks actions may be reused. | Make these method‑local or use request/session scope only. |
| **Service Locator** – `ServiceFactory.getService(...)` | Harder to test and to swap implementations. | Use dependency injection (e.g., Spring) or a constructor injection. |
| **No Persistence** – After `updateOrderTotal()`, changes to `Order` may not be saved. | Users might see stale totals if the order isn't persisted. | Call `oservice.updateOrder(order)` or equivalent after updating. |
| **Magic Numbers** – `paymentMethod.getType() == 1` to identify credit cards. | Non‑intuitive. | Use an enum or constants. |
| **Duplicated Logic** – `displayCartOrderSummaryAction()` only sets an attribute. | Redundant if the view logic can handle defaults. | Merge into a single method or use a default attribute set. |

### Future Enhancements  
1. **Introduce DTOs** – Create a view‑model DTO that aggregates all necessary fields instead of scattering attributes.  
2. **Validation Layer** – Ensure all required session objects exist before proceeding; otherwise redirect to an error or checkout start page.  
3. **Unit Tests** – Refactor code to allow easier unit testing by injecting mock services and session objects.  
4. **Internationalization** – Externalize all user‑visible strings (e.g., `"STEP"`, `"ADDRESSTYPE"`) into resource bundles.  
5. **Logging Improvements** – Add more context to logs (order ID, customer ID) to aid debugging.  

Overall, the class is functional but would benefit from modern Java practices (generics, dependency injection, defensive coding) to improve robustness, testability, and maintainability.

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
package com.salesmanager.checkout.flow;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.log4j.Logger;

import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.payment.CreditCard;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.entity.shipping.ShippingOption;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.www.SessionUtil;

public class DisplayOrderSummaryAction extends CheckoutBaseAction {

	private Logger log = Logger.getLogger(DisplayOrderSummaryAction.class);
	private OrderService oservice = (OrderService) ServiceFactory
			.getService(ServiceFactory.OrderService);

	private Order order;
	private OrderStatusHistory orderHistory;

	public String displayOrderSummaryAction() {

		try {

			super.preparePayments();

			// getServletRequest().setAttribute("REQUESTTYPE", "subscription");
			getServletRequest().setAttribute("STEP", 2);
			getServletRequest().setAttribute("ADDRESSTYPE", "BILLING");

			order = SessionUtil.getOrder(getServletRequest());
			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());

			Customer customer = SessionUtil.getCustomer(getServletRequest());

			PaymentMethod paymentMethod = SessionUtil
					.getPaymentMethod(getServletRequest());
			if (paymentMethod.getType() == 1) {// credit card
				CreditCard card = (CreditCard) paymentMethod.getConfig("CARD");
				getServletRequest().setAttribute("CARD", card);
			}

			Map orderProducts = SessionUtil.getOrderProducts(super
					.getServletRequest());

			ShippingInformation shippingInformation = SessionUtil
					.getShippingInformation(getServletRequest());

			Shipping shipping = null;
			if (shippingInformation != null) {

				ShippingOption option = shippingInformation
						.getShippingOptionSelected();

				shipping = new Shipping();
				shipping.setHandlingCost(shippingInformation.getHandlingCost());
				shipping.setShippingCost(option.getOptionPrice());
				shipping.setShippingModule(option.getModule());
				shipping.setShippingDescription(option.getDescription());
				getServletRequest().setAttribute("STEP", 3);
				getServletRequest().setAttribute("ADDRESSTYPE", "BOTH");

			}

			List products = new ArrayList();
			if (orderProducts != null) {
				Iterator i = orderProducts.keySet().iterator();
				while (i.hasNext()) {
					String line = (String) i.next();
					OrderProduct op = (OrderProduct) orderProducts.get(line);
					products.add(op);
				}
			}

			// update order with tax if it applies
			OrderTotalSummary summary = super.updateOrderTotal(order, products,
					customer, shipping, store);
			SessionUtil.setOrderTotalSummary(summary, getServletRequest());

		} catch (Exception e) {
			log.error(e);
			return "GLOBALERROR";
		}

		return SUCCESS;

	}

	public String displayCartOrderSummaryAction() {

		getServletRequest().setAttribute("ADDRESSTYPE", "BOTH");
		return SUCCESS;

	}

	public Order getOrder() {
		return order;
	}

	public void setOrder(Order order) {
		this.order = order;
	}

	public OrderStatusHistory getOrderHistory() {
		return orderHistory;
	}

	public void setOrderHistory(OrderStatusHistory orderHistory) {
		this.orderHistory = orderHistory;
	}

}



```
