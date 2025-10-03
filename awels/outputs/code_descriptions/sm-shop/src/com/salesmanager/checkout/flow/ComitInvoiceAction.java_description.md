# ComitInvoiceAction.java

## Review

## 1. Summary  

`ComitInvoiceAction` is a Struts‑style action that finalises an order by invoking a workflow (`invoiceWorkflow`).  
It pulls the `Order`, `Customer`, `MerchantStore`, `PaymentMethod` and the list of `OrderProduct`s from the HTTP session, assembles a `ProcessorContext`, and hands it to the workflow processor. The action also:

- Marks the transaction as committed in the session to guard against duplicate submits.
- Persists order status history if comments exist.
- Handles `TransactionException` and `OrderException` separately, sending error emails when necessary.
- Returns a string that maps to a Struts result page (`SUCCESS`, `PAYMENTERROR`, or `GENERICERROR`).

Key libraries/frameworks:  
- **Apache Commons Lang** (StringUtils)  
- **Apache Log4j** for logging  
- **Spring** (via `SpringUtil.getBean`) to obtain the workflow processor  
- **SalesManager Core** (entity and service classes)

The class follows a “command‑handler” pattern: the action orchestrates services but does not contain business logic itself.

---

## 2. Detailed Description  

### Core Flow

| Step | What happens | Purpose |
|------|--------------|---------|
| **1. Retrieve context objects** | `Order`, `MerchantStore`, `PaymentMethod`, `Customer`, and `orderProducts` are pulled from `SessionUtil`. | Pulls all data that was previously stored during the checkout process. |
| **2. Prevent duplicate commits** | Checks for a session attribute `TRANSACTIONCOMITED`. If present, returns an error. | Guarantees idempotence of the commit operation. |
| **3. Populate `Order`** | Sets payment method name and module code on the order. Converts the `orderProducts` map into a `Set` and assigns it. | Prepares the order for the workflow. |
| **4. Build `ProcessorContext`** | Adds the order, customer, store, payment method, locale, comments, downloadable files, and products to the context. | Provides all data the workflow needs. |
| **5. Execute workflow** | Obtains the `invoiceWorkflow` bean and calls `doWorkflow(context)`. | Executes all business rules, persisting the order, charging the payment, sending emails, etc. |
| **6. Post‑processing** | Sets the `TRANSACTIONCOMITED` flag, stores status history if comments are present. | Finalises the session state. |
| **7. Exception handling** | - `TransactionException` → payment error page. <br> - `OrderException` → send order‑problem e‑mail. <br> - Other exceptions → generic error page. | Communicates failure modes to the user and logs details. |

### Assumptions & Constraints

1. **Session‑centric data flow** – All required objects are already present in the session; no request parameters are validated here.  
2. **Single‑threaded action instances** – Struts actions are stateless per request, so shared fields (none in this class) are safe.  
3. **Hard‑coded strings** – Attribute names and workflow bean names are hard‑coded, which can lead to typos (`TRANSACTIONCOMITED`, `"prooducts"`).  
4. **Exception hierarchy** – Only `TransactionException` and `OrderException` are treated specially; all other exceptions are collapsed into a generic error.  
5. **Dependency resolution** – `ServiceFactory` and `SpringUtil` are static singletons; no dependency injection in the action.

### Architecture & Design Choices

- **Separation of Concerns**: The action focuses on orchestration; business logic lives in the workflow and services.  
- **Workflow Processor**: The `WorkflowProcessor` abstraction decouples the action from the concrete order‑processing logic.  
- **Session Utilisation**: Using `SessionUtil` centralises session access but also couples the code tightly to the session storage format.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `comitOrder()` | Main entry point; orchestrates the order commit. | None (reads from `HttpServletRequest`/session). | `String` – Struts result (`SUCCESS`, `PAYMENTERROR`, `GENERICERROR`). | Logs messages, sets session attributes, updates the order object, triggers the workflow. |

**Helper methods (inherited from `ComitOrderAction`)**

| Method | Purpose |
|--------|---------|
| `addActionError(String)` | Adds a user‑visible error to the action. |
| `addErrorMessage(String)` | Adds an error message for logging/display. |
| `getText(String, Object[])` | Retrieves internationalised message text. |
| `getOrderHistory()` | Retrieves the order history (comments) for the current order. |
| `getLocale()` | Returns the current `Locale` object. |
| `getServletRequest()` | Provides access to the underlying `HttpServletRequest`. |

---

## 4. Dependencies

| Library / Package | Purpose | Type |
|-------------------|---------|------|
| `org.apache.commons.lang.StringUtils` | String helper methods (`isBlank`) | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `com.salesmanager.core.*` | Core entity classes (`Order`, `Customer`, etc.) and services (`OrderService`, `TransactionException`) | Third‑party (SalesManager Core) |
| `com.salesmanager.core.service.workflow.*` | Workflow processing (`ProcessorContext`, `WorkflowProcessor`) | Third‑party |
| `com.salesmanager.core.util.SpringUtil` | Spring bean lookup | Third‑party |
| `com.salesmanager.core.util.www.SessionUtil` | Session helper | Third‑party |
| `com.salesmanager.core.service.ServiceFactory` | Service factory (static lookup) | Third‑party |
| `javax.servlet.http.HttpServletRequest` | HTTP request access (via `getServletRequest()`) | Java EE |

All dependencies are third‑party; none are standard Java SE.

---

## 5. Additional Notes

### Strengths

* **Clear orchestration** – The action is a single point of entry that clearly lists all steps.  
* **Error segregation** – Distinguishes between payment and order problems, providing appropriate user feedback.  
* **Use of workflow** – Business rules are externalised, making the action lightweight.

### Issues & Edge Cases

1. **Hard‑coded Typos**  
   * `TRANSACTIONCOMITED` and `"prooducts"` are misspelled. If other parts of the system expect the correct spelling, the duplicate‑commit guard or the product list retrieval may fail silently.

2. **Raw Types & Casting**  
   * `Map orderProducts` and the subsequent `for` loop use raw types. This leads to unchecked casts (`(OrderProduct)`) and potential `ClassCastException`s if the map contains unexpected values.

3. **Missing Null Checks**  
   * The code assumes all session objects (`order`, `store`, `payment`, `customer`) are non‑null. A missing attribute would result in a `NullPointerException` that is swallowed by the generic catch block, leading to a generic error page without context.

4. **Session Attribute Race Conditions**  
   * The duplicate‑commit flag is stored in the session, but the check and set are not atomic. Two concurrent requests for the same order could bypass the guard. A more robust solution would use a database flag or a distributed lock.

5. **Logging**  
   * The log statements often print only the exception (`log.error(e)`) but not the exception’s stack trace. This may hinder debugging. Use `log.error("msg", e)`.

6. **Deprecated Logging API**  
   * Log4j 1.x is end‑of‑life. Consider migrating to SLF4J with Logback or Log4j 2.x.

7. **Exception Propagation**  
   * The outermost `catch` logs the exception and then returns `SUCCESS`. This is likely a bug: an exception should not result in a successful outcome. The flow should return an error result.

8. **Hard‑coded Workflow Bean**  
   * `"invoiceWorkflow"` is hard‑coded. If the workflow name changes, the code must be updated. A constant or a property file would be safer.

9. **No Unit Tests**  
   * The action mixes controller logic with business logic, making it difficult to unit‑test without a web container or session mocks. Extracting the core logic into a service class would improve testability.

### Suggested Improvements

| Area | Recommendation |
|------|----------------|
| **Code style** | Fix typos (`TRANSACTIONCOMITED` → `TRANSACTION_COMMITTED`, `"prooducts"` → `"products"`), use constants for attribute names. |
| **Generics** | Replace raw types with `Map<String, OrderProduct>` and `Set<OrderProduct>` to avoid unchecked casts. |
| **Null handling** | Validate all session attributes; return a user‑friendly error if any are missing. |
| **Duplicate guard** | Use a database flag or a synchronized block to ensure atomicity. |
| **Logging** | Include stack traces and contextual information (`log.error("Failed to commit order", e)`). |
| **Exception handling** | Do not return `SUCCESS` when an exception occurs; always return an error result. |
| **Dependency injection** | Inject `OrderService`, `WorkflowProcessor`, etc., instead of static factories. |
| **Unit testability** | Move the commit logic into a service class (`OrderCommitService`) that the action simply delegates to. |
| **Modernize logging** | Switch to SLF4J/Logback or Log4j 2.x. |
| **Documentation** | Add JavaDoc to the `comitOrder()` method explaining its contract and side effects. |

---

**Verdict** – The action fulfills its intended purpose but suffers from several maintenance and robustness issues. Addressing the typos, null checks, and generics will improve reliability. Refactoring for dependency injection and unit‑testability will make the code more sustainable in the long run.

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

import java.util.Arrays;
import java.util.Collection;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.order.OrderException;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.service.payment.TransactionException;
import com.salesmanager.core.service.system.SystemService;
import com.salesmanager.core.service.workflow.ProcessorContext;
import com.salesmanager.core.service.workflow.WorkflowProcessor;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class ComitInvoiceAction extends ComitOrderAction {

	private Logger log = Logger.getLogger(ComitInvoiceAction.class);

	/** Complete overwrite **/
	public String comitOrder() {

		try {

			boolean paymentProcessed = false;

			// Get all entities

			Order order = SessionUtil.getOrder(getServletRequest());
			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());

			PaymentMethod payment = SessionUtil
					.getPaymentMethod(getServletRequest());

			order.setPaymentMethod(payment.getPaymentMethodName());
			order.setPaymentModuleCode(payment.getPaymentModuleName());

			Customer customer = SessionUtil.getCustomer(getServletRequest());

			if (super.getServletRequest().getSession().getAttribute(
					"TRANSACTIONCOMITED") != null) {
				addActionError(getText("error.transaction.duplicate",
						new String[] { String.valueOf(order.getOrderId()),
								store.getStoreemailaddress() }));
				return "GENERICERROR";
			}

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			try {

				Map orderProducts = SessionUtil
						.getOrderProducts(getServletRequest());
				Set s = new HashSet();
				
				for(Object o: orderProducts.values()) {
					
					OrderProduct op = (OrderProduct)o;
					s.add(op);
				}
				

				order.setOrderProducts(s);

				String comments = null;
				if (this.getOrderHistory() != null) {
					comments = this.getOrderHistory().getComments();
				}

				// Order, PaymentMethod,
				ProcessorContext context = new ProcessorContext();

				Collection files = oservice.getOrderProductDownloads(order
						.getOrderId());
				if (files != null && files.size() > 0) {
					context.addObject("files", files);

				}

				context.addObject("Order", order);
				context.addObject("Customer", customer);
				context.addObject("MerchantStore", store);
				context.addObject("PaymentMethod", payment);
				context.addObject("Locale", super.getLocale());
				context.addObject("comments", comments);
				context.addObject("prooducts", orderProducts.values());

				WorkflowProcessor wp = (WorkflowProcessor) SpringUtil
						.getBean("invoiceWorkflow");
				wp.doWorkflow(context);

				paymentProcessed = true;

				// set an indicator in HTTPSession to prevent duplicates
				super.getServletRequest().getSession().setAttribute(
						"TRANSACTIONCOMITED", "true");

				if (!StringUtils.isBlank(comments)) {
					SessionUtil.setOrderStatusHistory(this.getOrderHistory(),
							getServletRequest());
				}

			} catch (Exception e) {
				if (e instanceof TransactionException) {
					super.addErrorMessage("error.payment.paymenterror");
					return "PAYMENTERROR";
				}

				if (e instanceof OrderException) {
					try {
						oservice.sendOrderProblemEmail(order.getMerchantId(),
								order, customer, store);
					} catch (Exception ee) {
						log.error(ee);
					}
				}

				addActionError(getText("message.error.comitorder.error",
						new String[] { String.valueOf(order.getOrderId()),
								store.getStoreemailaddress() }));
				log.error(e);
				return "GENERICERROR";
			}

			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}
}



```
