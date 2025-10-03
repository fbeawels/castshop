# PaymentAction.java

## Review

## 1. Summary  

**Purpose**  
`PaymentAction` is a Struts‑style action that drives the payment selection stage of a checkout flow. It pulls the currently selected `PaymentMethod` from the HTTP session, displays the payment options, and validates the user’s selection before persisting it back to the session and routing to the appropriate payment gateway (e.g., PayPal Express).

**Key components**

| Component | Role |
|-----------|------|
| `displayPayment()` | Loads the current payment method, prepares the list of available methods and credit‑card data for the view. |
| `selectPayment()` | Handles the POSTed selection, validates it (including credit‑card checks), updates the session, and decides the next navigation step. |
| `PaymentMethod` | DTO that carries the selected payment module name, name, text, and configuration map. |
| `SessionUtil` | Utility class that abstracts the session read/write logic. |
| `PaymentUtil` | Static helper that identifies credit‑card payment modules. |
| `CheckoutBaseAction` | Super‑class that provides common checkout logic (e.g., `preparePayments()`, `prepareCreditCards()`, `validateCreditCard()`, `setTechnicalMessage()`). |

**Design patterns & frameworks**

* **Action (Command) pattern** – each method represents a user action that is executed by the Struts dispatcher.  
* **Session State pattern** – `SessionUtil` stores the payment selection across requests.  
* **Utility / Helper classes** – `PaymentUtil` and `SessionUtil` encapsulate reusable logic.  
* **Frameworks** – Apache Struts (implied by the action style), Log4j for logging, Apache Commons Lang for `StringUtils`.

---

## 2. Detailed Description  

### Initialization  
The action is instantiated per request by the Struts framework. No explicit constructor logic is present, so all state is set through request parameters or session attributes.

### Runtime Flow  

| Step | Method | Description |
|------|--------|-------------|
| **displayPayment** | Called when the user is presented with the payment selection page. | 1. Retrieve `PaymentMethod` from session. <br>2. If present, expose it to the request (`SELECTEDPAYMENT`). <br>3. If the payment module is a credit‑card type, populate the `CreditCard` configuration. <br>4. Prepare the full list of payment methods (`preparePayments()`) and available credit‑card types (`prepareCreditCards()`). <br>5. On success return `SUCCESS`; on any exception log and return `GENERICERROR`. |
| **selectPayment** | Handles the form POST where the user submits a chosen payment method. | 1. Validate that a payment module name has been provided. <br>2. If it is a credit‑card type, run `validateCreditCard()`. <br>3. Refresh the list of payment methods and copy display‑friendly fields (`name`, `text`, `config`) from the master list into the user’s selected instance. <br>4. Store the fully‑populated `PaymentMethod` back into the session. <br>5. If the module is PayPal, redirect to the Express Checkout flow; otherwise return `SUCCESS`. <br>6. Exceptions are logged and return `INPUT` to prompt the user to correct errors. |

### Cleanup  
No explicit cleanup logic exists; the action relies on the container’s request lifecycle to discard state after the response is rendered.

### Assumptions & Constraints  

* The application runs in a servlet container that supports HttpSession.  
* `PaymentMethod` objects are serializable (to be safely stored in session).  
* `SessionUtil` and `PaymentUtil` are thread‑safe.  
* The `PaymentConstants.PAYMENT_PAYPALNAME` constant contains the exact module name used by PayPal.  
* The view layer expects request attributes `SELECTEDPAYMENT`, `creditCard`, etc.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs / Side‑Effects |
|--------|-----------|---------|--------|------------------------|
| `displayPayment()` | `public String displayPayment()` | Prepares data for rendering the payment selection screen. | None (reads from session) | Sets request attributes, calls helper methods, returns navigation string (`SUCCESS` or `GENERICERROR`). |
| `selectPayment()` | `public String selectPayment()` | Processes the user’s payment choice, validates it, and persists it in the session. | None (reads request parameters mapped to `paymentMethod`) | Updates session, may set error messages, returns navigation string (`SUCCESS`, `INPUT`, or `"payPalExpressCheckout"`). |
| `getPaymentMethod()` | `public PaymentMethod getPaymentMethod()` | Getter used by the framework to bind form data to the action. | None | Returns the current `paymentMethod`. |
| `setPaymentMethod(PaymentMethod)` | `public void setPaymentMethod(PaymentMethod paymentMethod)` | Setter used by the framework to receive form data. | `PaymentMethod` from the form | Assigns to the action’s field. |

**Reusable / Utility Methods**  

The action relies on several inherited helper methods from `CheckoutBaseAction`:

* `preparePayments()` – loads the list of available payment methods.  
* `prepareCreditCards()` – loads available credit‑card configurations.  
* `validateCreditCard(PaymentMethod, long merchantId)` – performs credit‑card validation.  
* `setTechnicalMessage()` – flags a generic error for the view layer.

---

## 4. Dependencies  

| Library / Package | Type | Notes |
|-------------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Utility for string manipulation; used to check for blank module names. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework. |
| `com.salesmanager.checkout.CheckoutBaseAction` | Internal | Base action providing shared checkout logic. |
| `com.salesmanager.core.constants.PaymentConstants` | Internal | Holds constant strings (e.g., PayPal module name). |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Internal | Domain entity representing the merchant. |
| `com.salesmanager.core.entity.payment.CreditCard` | Internal | Domain entity holding credit‑card details. |
| `com.salesmanager.core.entity.payment.PaymentMethod` | Internal | Domain entity for payment modules. |
| `com.salesmanager.core.util.www.SessionUtil` | Internal | Utility to store/retrieve objects in the HttpSession. |
| `com.salesmanager.core.util.PaymentUtil` | Internal | Static helper to identify credit‑card payment modules. |

All dependencies are either part of the same application (internal) or widely used Java libraries; no external REST APIs or database drivers are referenced directly in this class.

---

## 5. Additional Notes  

### Edge Cases & Potential Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw Map usage** | `Map pms = super.getPaymentMethods();` – raw types lose compile‑time type safety and may cause unchecked warnings. | Use generics: `Map<String, PaymentMethod> pms = super.getPaymentMethods();` |
| **Null‑checking** | `paymentMethod.getPaymentModuleName()` is called after a null check on `paymentMethod` but the method itself could return `null`. The code already guards with `StringUtils.isBlank`, so this is safe. | Keep the guard; consider defensive null checks if `paymentMethod` might change between calls. |
| **Exception granularity** | Catching `Exception` hides the root cause and makes debugging harder. | Narrow to `RuntimeException` or specific checked exceptions thrown by called methods. |
| **Hard‑coded PayPal string** | Tightly coupling to `PaymentConstants.PAYMENT_PAYPALNAME` can break if the constant changes. | Keep the constant and expose it via a property or enum for readability. |
| **Commented‑out code** | Legacy code left in the method may confuse future developers. | Remove unused blocks or document why they’re retained. |
| **Session handling** | No validation that the session is present; `SessionUtil.getPaymentMethod()` may return `null` if the session expired. | Handle `null` explicitly or redirect to login. |
| **Magic strings in navigation** | Returning `"payPalExpressCheckout"` is a magic value that must be mapped in the Struts config. | Use constants or annotations to avoid typos. |

### Potential Enhancements  

1. **Dependency Injection** – Inject `SessionUtil`, `PaymentUtil`, and logging via a framework (e.g., Spring) to improve testability.  
2. **Service Layer** – Move business logic (validation, payment method retrieval) into a dedicated service instead of keeping it in the action.  
3. **Unit Tests** – Write tests for `displayPayment()` and `selectPayment()` using a mock servlet request/response and session.  
4. **Internationalisation** – Centralise error keys (`error.nopaymentmethod`) in a resource bundle.  
5. **Refactor navigation** – Replace string literals with enum values or constants to reduce errors.  
6. **Exception Handling** – Create custom exceptions for payment validation failures and handle them in a global exception handler.  

Overall, the class fulfills its role in the checkout flow, but modernizing the code (generics, better exception handling, separation of concerns) would make it more robust, maintainable, and testable.

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

import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.payment.CreditCard;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.util.www.SessionUtil;

public class PaymentAction extends CheckoutBaseAction {
	private PaymentMethod paymentMethod;// submited

	private Logger log = Logger.getLogger(PaymentAction.class);

	public String displayPayment() {
		try {
			paymentMethod = SessionUtil.getPaymentMethod(getServletRequest());
			if (paymentMethod != null) {
				super.getServletRequest().setAttribute("SELECTEDPAYMENT",
						paymentMethod);
				if (com.salesmanager.core.util.PaymentUtil
						.isPaymentModuleCreditCardType(paymentMethod
								.getPaymentModuleName())) {
					super.setCreditCard((CreditCard) paymentMethod
							.getConfig("CARD"));
				}

			}
			super.preparePayments();
			super.prepareCreditCards();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "GENERICERROR";
		}

		return SUCCESS;
	}

	public String selectPayment() {
		try {

			boolean isCreditCardPayment = false;

			if (this.getPaymentMethod() == null
					|| StringUtils.isBlank(this.getPaymentMethod()
							.getPaymentModuleName())) {
				super.addErrorMessage("error.nopaymentmethod");
				return INPUT;
			}

			if (com.salesmanager.core.util.PaymentUtil
					.isPaymentModuleCreditCardType(this.getPaymentMethod()
							.getPaymentModuleName())) {
				isCreditCardPayment = true;
				MerchantStore store = SessionUtil
						.getMerchantStore(getServletRequest());
				super.validateCreditCard(this.getPaymentMethod(), store
						.getMerchantId());

			}

			this.preparePayments();
			Map pms = super.getPaymentMethods();

			PaymentMethod tmpMethod = (PaymentMethod) pms.get(this
					.getPaymentMethod().getPaymentModuleName());

			/*
			 * if(tmpMethod==null && isCreditCardPayment) { tmpMethod =
			 * (PaymentMethod) pms.get("GATEWAY"); if(tmpMethod!=null) {
			 * if(!tmpMethod
			 * .getPaymentModuleName().equals(this.getPaymentMethod(
			 * ).getPaymentModuleName())) { tmpMethod = null; } } }
			 */

			if (tmpMethod != null) {
				this.getPaymentMethod().setPaymentMethodName(
						tmpMethod.getPaymentMethodName());
				this.getPaymentMethod().setPaymentModuleText(
						tmpMethod.getPaymentModuleText());
				this.getPaymentMethod().setPaymentMethodConfig(
						tmpMethod.getPaymentMethodConfig());
			}

			SessionUtil.setPaymentMethod(this.getPaymentMethod(),
					getServletRequest());

			// check paypal
			if (paymentMethod.getPaymentModuleName().equals(
					PaymentConstants.PAYMENT_PAYPALNAME)) {
				// set the number of steps
				return "payPalExpressCheckout";
			}

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
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
