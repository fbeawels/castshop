# DisplayInvoiceSummaryAction.java

## Review

## 1. Summary

`DisplayInvoiceSummaryAction` is a Struts‑type action that prepares the data required to render an invoice summary page.  
- It extends `DisplayOrderSummaryAction`, inheriting helper methods such as `displayCartOrderSummaryAction()` and access to the current `HttpServletRequest`.  
- The action pulls the logged‑in `Customer` and the current `Order` from the HTTP session, extracts the latest order‑history comment, pulls payment information (credit‑card or other payment module), and places all of these objects into the request scope for the view.  
- The class relies on a few core business entities (`Customer`, `Order`, `OrderStatusHistory`, `PaymentMethod`, `CreditCard`) and on a simple `SessionUtil` helper to read data from the session.

The code is straightforward but shows a few antipatterns (raw types, magic numbers, generic exception handling) and lacks defensive programming.

---

## 2. Detailed Description

### Flow of execution

1. **Initialisation** – The method is invoked by the Struts framework and immediately calls  
   `super.displayCartOrderSummaryAction()` to perform any common cart‑summary logic.

2. **Session retrieval** –  
   ```java
   customer = SessionUtil.getCustomer(super.getServletRequest());
   order    = SessionUtil.getOrder(super.getServletRequest());
   ```  
   These helpers pull the current `Customer` and `Order` from the `HttpSession`.  No null checks are performed.

3. **Order history** –  
   ```java
   Set historySet = order.getOrderHistory();
   ```  
   The code iterates over the set, taking the *last* `OrderStatusHistory` entry (the comment says there should be only one).  The comment of that entry is stored in the request as `"HISTORY"`.

4. **Payment method** –  
   ```java
   PaymentMethod paymentMethod = SessionUtil.getPaymentMethod(getServletRequest());
   ```  
   If the payment method is a credit‑card (`type == 1`), the credit‑card config is fetched and stored as `"CARD"`.  Regardless of type the order’s `paymentMethodName` and `paymentModuleName` are copied onto the `Order` entity.

5. **Request attributes** – Finally, the `Customer`, `Order`, and any extracted data are added to the request so that the JSP/FreeMarker can display them.

6. **Error handling** – Any exception is caught, logged with `log.error(e)`, and the action silently returns `"SUCCESS"`.  No error state is propagated to the view.

### Architecture & Design Choices

- **Struts‑style action**: The class follows a classic Struts 1/2 pattern where the action extends a base action and returns a logical forward name.
- **SessionUtil helper**: Keeps the servlet‑specific session handling out of the action, which is good for testability, but the helper itself may still be tightly coupled to the HTTP session.
- **Raw types**: The use of `Set` and `Iterator` without generics indicates legacy code; this makes the code harder to read and increases the risk of `ClassCastException`.
- **Magic numbers**: `paymentMethod.getType() == 1` relies on an implicit meaning (“credit card”).  A constant would improve readability.
- **No validation or error propagation**: All exceptions are swallowed, making it difficult for the framework to display an error page or for a developer to troubleshoot.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `public String displayOrderSummaryAction()` | Main entry point; prepares data for the invoice summary view. | None (uses `HttpServletRequest` from context) | Returns `"SUCCESS"` | Sets request attributes; logs errors |
| `public Order getOrder()` | Getter for `order`. | None | `Order` instance | None |
| `public void setOrder(Order order)` | Setter for `order`. | `Order` instance | None | Assigns to member variable |
| `public Customer getCustomer()` | Getter for `customer`. | None | `Customer` instance | None |
| `public void setCustomer(Customer customer)` | Setter for `customer`. | `Customer` instance | None | Assigns to member variable |

The only “real” logic resides in `displayOrderSummaryAction()`. The getters/setters are standard JavaBeans properties.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Classic Log4j 1.x logger. |
| `com.salesmanager.core.entity.*` | Third‑party | Domain entities (`Customer`, `Order`, `OrderStatusHistory`, `PaymentMethod`, `CreditCard`). |
| `com.salesmanager.core.util.www.SessionUtil` | Third‑party | Utility for session handling. |
| `javax.servlet.http.HttpServletRequest` | Standard | Implicit via `getServletRequest()`. |
| `com.salesmanager.checkout.flow.DisplayOrderSummaryAction` | In‑project | Base action providing `displayCartOrderSummaryAction()`. |

No external APIs (e.g., web services) are invoked.

---

## 5. Additional Notes

### Strengths

- Keeps session logic in a dedicated helper (`SessionUtil`).
- Clear separation between data retrieval (session) and request population.
- Uses a base class (`DisplayOrderSummaryAction`) to avoid duplication of common logic.

### Weaknesses & Edge Cases

1. **Raw types** – Using `Set` and `Iterator` without generics obscures the element type and can lead to `ClassCastException` if the underlying collection changes.
2. **Magic number** – `paymentMethod.getType() == 1` is brittle; if the type codes change, the logic will silently break.
3. **Exception swallowing** – Catching `Exception` and only logging prevents the framework from showing an error page. It also makes unit testing harder (no way to assert failure).
4. **Null handling** – None of the session objects (`customer`, `order`, `paymentMethod`) are checked for `null`.  A missing session attribute could lead to NPEs that are caught, but the user would still see a “success” page with incomplete data.
5. **Assumption of a single history entry** – The loop over `order.getOrderHistory()` merely keeps the last element. If the set contains multiple entries, only the most recent comment will be shown, but this may not be intentional.
6. **No request validation** – The action does not verify that the user actually owns the order or that the order is in a state that can be invoiced.

### Suggested Improvements

| Area | Recommendation |
|------|----------------|
| **Generics** | Replace raw `Set`/`Iterator` with `Set<OrderStatusHistory>` and enhanced for‑loops. |
| **Constants** | Define an enum or constant for payment types (`PAYMENT_TYPE_CREDIT_CARD = 1`). |
| **Error handling** | Propagate an error result (`ERROR` or a custom forward) when exceptions occur or when required session data is missing. |
| **Null checks** | Guard against `null` `customer`, `order`, `paymentMethod`. If any are missing, redirect to login or error page. |
| **Logging** | Log an informative message alongside the exception: `log.error("Failed to display invoice summary", e);` |
| **Session abstraction** | Consider injecting a session service instead of using a static helper. |
| **Unit testing** | Decouple the action logic from the servlet container to allow isolated unit tests. |
| **Comments & Javadoc** | Add method documentation and clarify intent, especially for the order‑history loop. |

### Future Extensions

- **Multi‑currency support**: Attach currency information to the order and format amounts accordingly.
- **Internationalisation**: Load locale‑specific messages and labels for the invoice view.
- **Security checks**: Verify that the current user has permission to view the order’s invoice.
- **Audit logging**: Record that the invoice was viewed for compliance purposes.

By addressing these issues, the action will become more robust, maintainable, and easier to test.

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

import java.util.Iterator;
import java.util.Set;

import org.apache.log4j.Logger;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.entity.payment.CreditCard;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.util.www.SessionUtil;

public class DisplayInvoiceSummaryAction extends DisplayOrderSummaryAction {

	private Logger log = Logger.getLogger(DisplayInvoiceSummaryAction.class);

	private Order order;
	private Customer customer;

	public String displayOrderSummaryAction() {

		try {

			super.displayCartOrderSummaryAction();

			/**
			 * Set objects in the HttpRequest
			 */

			customer = SessionUtil.getCustomer(super.getServletRequest());

			order = SessionUtil.getOrder(super.getServletRequest());

			// Comments
			Set historySet = order.getOrderHistory();
			if (historySet != null) {
				// there should be one history
				Iterator historySetIterator = historySet.iterator();
				while (historySetIterator.hasNext()) {// get the last entry
					OrderStatusHistory history = (OrderStatusHistory) historySetIterator
							.next();
					super.getServletRequest().setAttribute("HISTORY",
							history.getComments());
				}
			}

			// Payment
			PaymentMethod paymentMethod = SessionUtil
					.getPaymentMethod(getServletRequest());

			if (paymentMethod != null) {

				if (paymentMethod.getType() == 1) {// credit card
					CreditCard card = (CreditCard) paymentMethod
							.getConfig("CARD");
					getServletRequest().setAttribute("CARD", card);
				}
				order.setPaymentMethod(paymentMethod.getPaymentMethodName());
				order
						.setPaymentModuleCode(paymentMethod
								.getPaymentModuleName());
			}

			// Customer
			super.getServletRequest().setAttribute("CUSTOMER", customer);

			// Order
			super.getServletRequest().setAttribute("ORDER", order);

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	public Order getOrder() {
		return order;
	}

	public void setOrder(Order order) {
		this.order = order;
	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

}



```
