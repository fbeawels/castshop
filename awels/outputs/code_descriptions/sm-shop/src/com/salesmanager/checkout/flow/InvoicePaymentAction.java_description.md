# InvoicePaymentAction.java

## Review

## 1. Summary

`InvoicePaymentAction` is a Struts‑style action class that prepares the data required to display the payment page for an order that is paid by invoice.  
It pulls the current `Customer` and `Order` from the HTTP session, exposes the order history, and sets the necessary request attributes for the view. If the selected payment method is a money‑order it forwards straight to the order summary; otherwise it delegates to the superclass `PaymentAction` to perform the standard payment‑page logic.

Key components  
| Component | Role |
|-----------|------|
| `SessionUtil` | Helper to fetch `Customer`, `Order`, and `PaymentMethod` from the session |
| `PaymentConstants` | Constant holding the money‑order module name |
| `OrderStatusHistory` | Holds comments/history for an order |
| `Order`, `Customer` | Domain entities representing the order and the customer |

The class is thin and focuses on data‑preparation; the heavy lifting is delegated to the parent class.

---

## 2. Detailed Description

### Execution Flow

1. **Entry Point** – `displayPayment()` is invoked by the Struts framework.  
2. **Session Retrieval** –  
   * `Customer` and `Order` are fetched from the session using `SessionUtil`.  
3. **Order History** –  
   * If the order has any status history, the last entry’s comments are extracted and stored in the request attribute `HISTORY`.  
4. **Request Attribute Population** –  
   * The `CUSTOMER` and `ORDER` attributes are set on the request.  
5. **Payment Method Check** –  
   * The selected `PaymentMethod` is retrieved. If it is a money‑order, the action returns `"summary"` immediately.  
   * Otherwise, the payment method is stored in the action and the parent `displayPayment()` is called.  
6. **Error Handling** – Any exception causes an error log and the action returns `INPUT`.  
7. **Success** – If no exception and not a money‑order, the action ultimately returns `SUCCESS`.

### Assumptions & Constraints

* The action relies on the session being populated with a valid `Order` and `Customer`.  
* `SessionUtil` provides thread‑safe access to the session; the action itself holds instance fields (`order`, `customer`) which are **not** thread‑safe.  
* The order’s history set contains at most one entry (as commented), but the code iterates over all entries.  
* The application uses the Struts 1/2 convention where action return strings map to result pages.  
* The action does not validate the state of the `Order` or `Customer` (e.g., null checks).  

### Architecture & Design Choices

* **MVC** – The action acts as the controller, preparing data for the view.  
* **Inheritance** – It extends `PaymentAction` to reuse common payment logic.  
* **Utility Class** – `SessionUtil` abstracts session handling, keeping the action clean.  
* **Constants** – `PaymentConstants` centralises string literals for payment modules.  

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `public String displayPayment()` | Main entry point; populates request, checks payment method, delegates to parent. | None (uses `getServletRequest()`/`SessionUtil`) | `"summary"`, `"SUCCESS"`, or `"INPUT"` string used by Struts to resolve view | Sets request attributes (`CUSTOMER`, `ORDER`, `HISTORY`), logs errors |
| `public Order getOrder()` | Getter for the `order` instance field (used by views/JSP). | None | `Order` instance | None |
| `public void setOrder(Order order)` | Setter for the `order` field. | `Order` | None | Updates instance field |
| `public Customer getCustomer()` | Getter for the `customer` field. | None | `Customer` | None |
| `public void setCustomer(Customer customer)` | Setter for the `customer` field. | `Customer` | None | Updates instance field |

> **Reusable Utilities**  
> *`SessionUtil.getCustomer(request)`*, *`SessionUtil.getOrder(request)`*, *`SessionUtil.getPaymentMethod(request)`* are used repeatedly across actions to fetch session objects.

---

## 4. Dependencies

| Library/Package | Type | Notes |
|-----------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Logging framework |
| `com.salesmanager.core.constants.PaymentConstants` | Third‑party | Holds payment module constants |
| `com.salesmanager.core.entity.*` | Core domain | `Customer`, `Order`, `OrderStatusHistory`, `PaymentMethod` |
| `com.salesmanager.core.util.www.SessionUtil` | Core utility | Session helper |
| `java.util.*` | JDK | `Set`, `Iterator` |
| `javax.servlet.*` (implied via `getServletRequest()`) | JDK | Servlet API |

The code is not tied to any particular platform beyond a servlet container (e.g., Tomcat). It is framework‑agnostic aside from the Struts/Action convention.

---

## 5. Additional Notes & Recommendations

### 5.1 Thread‑Safety & Instance Fields
`order` and `customer` are instance fields. In a servlet/action environment these objects may be reused across requests, leading to data leakage or race conditions.  
**Fix:** Use local variables inside `displayPayment()` or mark the action as `RequestScoped` if the framework supports it.

### 5.2 Generics & Raw Types
The code uses raw `Set` and `Iterator`.  
**Fix:**  
```java
Set<OrderStatusHistory> historySet = order.getOrderHistory();
for (OrderStatusHistory history : historySet) { … }
```

### 5.3 History Handling
The comment suggests there should be *one* history entry, yet the loop iterates over all.  
**Fix:** Break after the first iteration or fetch the most recent history directly.

### 5.4 Null‑Pointer Checks
`order`, `customer`, and `pm` are used without null checks.  
**Fix:** Validate each before use and handle missing values gracefully (e.g., redirect to an error page).

### 5.5 Return Value Logic
`super.displayPayment()` is called but its return value is ignored; the method always returns `SUCCESS`.  
**Fix:** Return the value from the superclass or handle its possible failures.

### 5.6 Logging
`log.error(e)` logs only the exception, not the stack trace.  
**Fix:**  
```java
log.error("Error displaying payment", e);
```

### 5.7 Code Style
* Inconsistent use of `super.getServletRequest()` vs. `getServletRequest()`.  
* Magic string comparison via `equals()` on `PaymentConstants.PAYMENT_MONEYORDERNAME`.  
* Unnecessary `try/catch` around the entire method – consider narrower exception handling.

### 5.8 Potential Enhancements
* **Validation Layer:** Add a dedicated validator to ensure order and customer states are valid before displaying payment.  
* **Unit Tests:** Mock `SessionUtil` and `Order` objects to test different scenarios (money‑order vs. other payments).  
* **Async Payment Integration:** If the application expands to support online payments, refactor `displayPayment()` to use a strategy pattern.  
* **Security:** Ensure that sensitive customer data is not inadvertently exposed in the request scope.

---

**Overall Verdict:**  
The action serves its purpose in the current application context but suffers from a few code‑quality issues (thread‑safety, raw types, missing null checks, and return‑value handling). Addressing these will make the code more robust, maintainable, and easier to test.

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

import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderStatusHistory;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.util.www.SessionUtil;

public class InvoicePaymentAction extends PaymentAction {

	private Logger log = Logger.getLogger(PaymentAction.class);

	private Order order;
	private Customer customer;

	public String displayPayment() {

		try {

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

			// Customer
			super.getServletRequest().setAttribute("CUSTOMER", customer);

			// Order
			super.getServletRequest().setAttribute("ORDER", order);

			// if money order go to summary
			PaymentMethod pm = SessionUtil
					.getPaymentMethod(getServletRequest());
			if (pm != null) {
				if (pm.getPaymentModuleName().equals(
						PaymentConstants.PAYMENT_MONEYORDERNAME)) {
					return "summary";
				}

				super.setPaymentMethod(pm);
			}
			super.displayPayment();
		} catch (Exception e) {
			log.error(e);
			return INPUT;
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
