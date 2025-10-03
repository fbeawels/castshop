# InitCheckoutAction.java

## Review

## 1. Summary  

**Purpose**  
`InitCheckoutAction` is a Struts‑2 action (extending `CheckoutBaseAction`) that prepares the checkout wizard when a user clicks the “checkout” button from the shopping cart. It validates cart contents, sets up payment information, calculates order totals, and builds the sequence of checkout steps (billing, shipping, order summary).  

**Key Components**  

| Component | Role |
|-----------|------|
| `paymentMethod` | Holds the payment method selected by the user (or a default free payment). |
| `initCheckout()` | Main orchestration method invoked by the framework. |
| `ProcessStep` objects | Represent each page of the wizard (billing, shipping, summary). |
| `SessionUtil` | Static helper for reading/writing session attributes such as order, products, payment method, and flags (`HAS_SHIPPING`, `TRANSACTIONCOMITED`). |
| `OrderService` | Service that recalculates the order total. |
| `PaymentMethod` | Domain model for a payment gateway/module. |
| `PropertiesUtil` & `LabelUtil` | Retrieve internationalised labels and configuration values. |

**Notable Patterns / Libraries**  

* Uses **Struts‑2** action conventions (`SUCCESS`, `INPUT`, `GENERICERROR`).  
* **SessionUtil** implements a crude “session‑scoped” state machine.  
* Dependency injection is absent – services are obtained via `ServiceFactory.getService()` (a Service Locator).  
* Logging is performed with **log4j**.  

---

## 2. Detailed Description  

### 2.1 Flow of Execution  

1. **Cart Validation**  
   * Retrieve `orderProducts` from the session.  
   * Verify that the map is not empty and that each product quantity is within bounds.  
   * If invalid → add error message and return `INPUT`.

2. **Session Reset**  
   * Remove `TRANSACTIONCOMITED` flag (used to detect duplicate commits).

3. **Payment Method Handling**  
   * If `paymentMethod` is null, check if a payment method already exists in the session.  
   * If none → create a *free* payment method and store it.  
   * If a method exists → add error and return `INPUT`.  
   * If a method exists and is `"free"` → re‑create the free payment object, store it, and clear the `HAS_PAYMENT` flag.

4. **Identify Shipping Requirement**  
   * Iterate over products to set `hasShipping`.  
   * Build a list of `OrderProduct` objects (`productList`) for total calculation.

5. **Order Total Recalculation**  
   * Obtain the current `Order` and `MerchantStore`.  
   * If the order is not an invoice channel, call `OrderService.calculateTotal()` to compute the new total and update the `Order`.

6. **Prepare Checkout Steps**  
   * Create three `ProcessStep` instances: billing, shipping, summary.  
   * Set appropriate labels and URLs (derived from `PropertiesUtil`).  
   * Flag `HAS_SHIPPING` in the session.  
   * If the payment is PayPal Express, add the steps to the session and return `"payPalExpressCheckout"`.

7. **General Payment Setup**  
   * Call `preparePayments()` from the base class to load available payment methods.  
   * Resolve the chosen payment method’s configuration, name, etc.  
   * Store the resolved method back in the session.

8. **Store Steps**  
   * Put the list of steps into the session under the key `"STEPS"`.  

9. **Exception Handling**  
   * Any uncaught exception logs an error, sets a generic technical message, and returns `"GENERICERROR"`.

10. **Return**  
    * On success → return `SUCCESS`.

### 2.2 Dependencies & Assumptions  

| Dependency | Type | Notes |
|------------|------|-------|
| `SessionUtil` | Static helper | Assumes a single session per user; no concurrency control. |
| `ServiceFactory` | Service locator | Pulls services by string key; fragile if keys change. |
| `PropertiesUtil` | Config loader | Relies on external properties file for URLs. |
| `LabelUtil` | i18n | Requires locale to be correctly set in the action context. |
| `OrderService` | Business logic | Performs currency conversion, taxes, shipping, etc. |

Assumptions:  
* The session already contains a valid `Order` and `MerchantStore`.  
* The payment method is either already selected or defaults to free.  
* `OrderProduct.isShipping()` correctly indicates whether shipping is required.  

### 2.3 Design Observations  

* **Imperative, procedural code** – All logic is inside a single method; readability suffers.  
* **Tight coupling** – The action directly manipulates session attributes, calls static utilities, and uses hard‑coded property keys.  
* **No separation of concerns** – Business logic (order total calculation, payment configuration) is interleaved with UI‑oriented code (step URLs, labels).  
* **Lack of input validation framework** – Manual checks could be replaced by validation annotations or Struts 2 validator XML.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `initCheckout()` | Main entry point; orchestrates checkout initialization. | None | `String` (action result) | Modifies session, calculates totals, creates `ProcessStep` list, sets payment method. |
| `getPaymentMethod()` | Getter for `paymentMethod`. | None | `PaymentMethod` | None |
| `setPaymentMethod(PaymentMethod)` | Setter for `paymentMethod`. | `PaymentMethod` | None | None |

The class contains only these three methods; the rest of the logic lives inside `initCheckout()`.  

---

## 4. Dependencies  

| Library | Category | Notes |
|---------|----------|-------|
| `org.apache.log4j.Logger` | Logging | Classic log4j 1.x; considered outdated. |
| `com.salesmanager.*` | Internal | Domain models (`MerchantStore`, `Order`, etc.), services (`OrderService`), utilities (`LabelUtil`, `PaymentUtil`, `PropertiesUtil`). |
| `org.apache.struts2` | Web framework | Struts‑2 action inheritance, result constants. |
| `java.util.*` | Core Java | Collections, maps, iterators. |

All dependencies are **third‑party** or **internal**; no platform‑specific APIs are used. The code is tightly coupled to the SalesManager framework.  

---

## 5. Additional Notes  

### 5.1 Edge Cases & Robustness  

| Edge Case | Current Behavior | Recommendation |
|-----------|------------------|----------------|
| `orderProducts` contains null keys or values | `NullPointerException` may occur when iterating. | Validate map contents or use safer iteration. |
| `orderProducts` contains product quantities outside acceptable range | Adds error and returns `INPUT`. | Good, but consider more informative message (product name). |
| Multiple simultaneous checkout attempts | Shared session flag `TRANSACTIONCOMITED` may be overwritten. | Use a request‑scoped token or CSRF token to guard against duplicates. |
| Missing `Order` or `MerchantStore` in session | `NullPointerException` at `SessionUtil.getOrder()`. | Verify presence and handle gracefully. |
| Property keys missing or malformed | `NullPointerException` or wrong URLs. | Validate config values early or provide defaults. |
| PayPal Express path – missing `STEPS` in session | Session may be corrupted. | Ensure steps are always stored before returning. |

### 5.2 Potential Refactoring & Enhancements  

1. **Extract Helper Classes**  
   * Create a `CheckoutContext` bean to hold the session attributes and services, reducing static dependencies.  
   * Split `initCheckout()` into smaller private methods: `validateCart()`, `setupPaymentMethod()`, `calculateTotals()`, `buildSteps()`.  

2. **Use Dependency Injection**  
   * Replace `ServiceFactory` and static util calls with constructor‑injected services (`OrderService`, `LabelService`, `ConfigService`).  

3. **Adopt Struts‑2 Validators**  
   * Move quantity checks into a validator XML or annotation to keep action logic lean.  

4. **Modernize Logging**  
   * Upgrade to SLF4J + Logback; use parameterised logging (`log.error("No products in checkout: {}", e)`).  

5. **Handle Internationalisation Better**  
   * Centralise label resolution, maybe via a `LabelResolver` service.  

6. **Introduce Constants for Session Keys**  
   * Avoid magic strings like `"STEPS"` or `"TRANSACTIONCOMITED"`; use `public static final String STEPS_KEY = "STEPS";`.  

7. **Unit Tests**  
   * Write tests for each extracted helper method.  
   * Mock `SessionUtil`, `OrderService`, and the config utilities.  

8. **Error Reporting**  
   * Provide more user‑friendly messages (e.g., “Product X quantity exceeds available stock”).  

9. **Concurrency Safety**  
   * Ensure session updates are atomic or use synchronized blocks if necessary.  

10. **Logging Best Practices**  
    * Log at appropriate levels; avoid logging entire stack traces unless needed.  

### 5.3 Design Pattern Opportunities  

* **Builder** – For constructing the `ProcessStep` list.  
* **Strategy** – Different payment modules could implement a common interface for preparing the checkout.  
* **Template Method** – Base action could define a skeleton, with subclasses providing specific payment handling.  

---  

**Overall**, the class accomplishes its goal but would benefit significantly from modularisation, better separation of concerns, and modern Java practices. Refactoring will improve maintainability, testability, and reduce the risk of subtle bugs in a complex checkout flow.

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
import com.salesmanager.core.constants.OrderConstants;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.PaymentUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class InitCheckoutAction extends CheckoutBaseAction {

	private PaymentMethod paymentMethod;// submited

	private Logger log = Logger.getLogger(InitCheckoutAction.class);

	/**
	 * Invoked from shopping cart when clicking on checkout
	 * 
	 * @return
	 */
	public String initCheckout() {

		try {
			
			
			//validate products
			Map orderProducts = SessionUtil.getOrderProducts(super.getServletRequest());
			
			if (orderProducts == null || orderProducts.size() == 0) {
				log.error("No prroducts in checkout !");
				super.setTechnicalMessage();
				return "GENERICERROR";
			}
			
			for(Object o:orderProducts.keySet()) {
				
				String line = (String)o;
				OrderProduct op = (OrderProduct)orderProducts.get(line);
				if(op.getProductQuantity()==0 || op.getProductQuantity()>op.getProductQuantityOrderMax()) {
					super.addErrorMessage("messages.invalid.quantity");
					return INPUT;
				}
				
			}
			
			

			super.getServletRequest().getSession().removeAttribute(
					"TRANSACTIONCOMITED");

			if (paymentMethod == null) {

				// check if has payment

				Boolean hasPayment = SessionUtil
						.isHasPayment(getServletRequest());
				if (hasPayment == false) {
					// set free payment
					PaymentMethod pm = new PaymentMethod();
					pm.setPaymentMethodName(LabelUtil.getInstance().getText(
							super.getLocale(), "module.free"));
					pm.setPaymentModuleName(PaymentConstants.PAYMENT_FREE);
					SessionUtil.setPaymentMethod(pm, getServletRequest());
					this.setPaymentMethod(pm);
				} else {
					super.addErrorMessage("error.nopaymentmethod");
					return INPUT;
				}
			} else {
				if (paymentMethod.getPaymentModuleName().equals("free")) {
					PaymentMethod pm = new PaymentMethod();
					pm.setPaymentMethodName(LabelUtil.getInstance().getText(
							super.getLocale(), "module.free"));
					pm.setPaymentModuleName(PaymentConstants.PAYMENT_FREE);
					SessionUtil.setPaymentMethod(pm, getServletRequest());
					SessionUtil.setHasPayment(false, getServletRequest());
					this.setPaymentMethod(pm);
				}
			}

			/**
			 * For checkout steps
			 */
			ProcessStep billing = new ProcessStep();

			boolean hasShipping = false;


			ArrayList productList = new ArrayList();

			Iterator i = orderProducts.keySet().iterator();
			while (i.hasNext()) {
				String line = (String) i.next();
				OrderProduct op = (OrderProduct) orderProducts.get(line);
				if (op.isShipping()) {
					hasShipping = true;
				}
				productList.add(op);
			}

			// populate Order total information
			Order order = SessionUtil.getOrder(getServletRequest());
			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());

			// can't re-calculate if an invoice which is already calculated
			if (order.getChannel() != OrderConstants.INVOICE_CHANNEL) {
				OrderService oservice = (OrderService) ServiceFactory
						.getService(ServiceFactory.OrderService);
				OrderTotalSummary total = oservice.calculateTotal(order,
						productList, null, null, store.getCurrency(), super
								.getServletRequest().getLocale());
				order.setTotal(total.getTotal());
			}

			String billingText = LabelUtil.getInstance().getText(
					super.getLocale(), "label.checkout.billinginfo");
			if (hasShipping) {
				SessionUtil.setHasShipping(true, getServletRequest());
				billingText = LabelUtil.getInstance()
						.getText(super.getLocale(),
								"label.checkout.shippingbillinginfo");
			} else {
				SessionUtil.setHasShipping(false, getServletRequest());
			}
			billing.setLabel(billingText);
			// billing.setUrl(new
			// StringBuffer().append(PropertiesUtil.getConfiguration().getString("core.salesmanager.checkout.uri")).append(PropertiesUtil.getConfiguration().getString("core.salesmanager.checkout.customerAction")).toString());
			// billing.setUrl(new
			// StringBuffer().append(PropertiesUtil.getConfiguration().getString("core.salesmanager.checkout.uri")).append(PropertiesUtil.getConfiguration().getString("core.salesmanager.checkout.customerAction")).toString());
			billing
					.setUrl(new StringBuffer()
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.catalog.url"))
							.append("/")
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.checkout.uri"))
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.checkout.customerAction"))
							.toString());

			ProcessStep shipping = new ProcessStep();
			shipping.setLabel(LabelUtil.getInstance().getText(
					super.getLocale(), "label.cart.shipingoptions"));
			// shipping.setUrl(new
			// StringBuffer().append(PropertiesUtil.getConfiguration().getString("core.salesmanager.checkout.uri")).append(PropertiesUtil.getConfiguration().getString("core.salesmanager.checkout.shippingAction")).toString());
			shipping
					.setUrl(new StringBuffer()
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.catalog.url"))
							.append("/")
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.checkout.uri"))
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.checkout.shippingAction"))
							.toString());

			ProcessStep summary = new ProcessStep();
			summary.setLabel(LabelUtil.getInstance().getText(super.getLocale(),
					"label.checkout.ordersummary"));
			// summary.setUrl(new
			// StringBuffer().append(PropertiesUtil.getConfiguration().getString("core.salesmanager.catalog.url")).append(PropertiesUtil.getConfiguration().getString("core.salesmanager.checkout.summaryAction")).toString());
			summary
					.setUrl(new StringBuffer()
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.catalog.url"))
							.append("/")
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.checkout.uri"))
							.append(
									PropertiesUtil
											.getConfiguration()
											.getString(
													"core.salesmanager.checkout.summaryAction"))
							.toString());

			super.preparePayments();
			Map paymentMethods = super.getPaymentMethods();

			PaymentMethod pm = (PaymentMethod) paymentMethods.get(this
					.getPaymentMethod().getPaymentModuleName());
			if (PaymentUtil.isPaymentModuleCreditCardType(this
					.getPaymentMethod().getPaymentModuleName())) {
				pm = (PaymentMethod) paymentMethods.get("GATEWAY");
			}

			if (pm != null) {
				this.getPaymentMethod().setPaymentMethodConfig(
						pm.getPaymentMethodConfig());
				this.getPaymentMethod().setPaymentMethodName(
						pm.getPaymentMethodName());
			}

			SessionUtil.setPaymentMethod(this.getPaymentMethod(), super
					.getServletRequest());

			if (paymentMethod.getPaymentModuleName().equals(
					PaymentConstants.PAYMENT_PAYPALNAME)) {
				// set the number of steps
				List steps = new ArrayList();
				steps.add(billing);
				if (hasShipping) {
					steps.add(shipping);
				}
				steps.add(summary);
				super.getServletRequest().getSession().setAttribute("STEPS",
						steps);
				return "payPalExpressCheckout";
			}

			// Prepare steps
			// ---------------
			// 1) Billing & Shipping information
			// 2) Shipping cost
			// 3) Order Summary
			List steps = new ArrayList();
			steps.add(billing);
			if (hasShipping) {
				steps.add(shipping);
			}
			steps.add(summary);
			super.getServletRequest().getSession().setAttribute("STEPS", steps);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "GENERICERROR";
		}

		return SUCCESS;
	}

	public PaymentMethod getPaymentMethod() {
		return paymentMethod;
	}

	public void setPaymentMethod(PaymentMethod paymentMethod) {
		this.paymentMethod = paymentMethod;
	}

}



```
