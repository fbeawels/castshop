# TransactionDetailsAction.java

## Review

## 1. Summary  

**Purpose**  
`TransactionDetailsAction` is a Struts‑2 action that handles transaction‑related operations for an e‑commerce order.  
- It can **display** all transactions belonging to an order.  
- It can **process** a new transaction (capture or refund) for a pre‑authorized order.  

**Key Components**  
| Component | Role |
|-----------|------|
| `prepareOrderDetails()` | Loads the order, validates merchant ownership, sets page title and order total. |
| `processTransaction()` | Validates input, determines transaction type (`CAPTURE` or `REFUND`), performs the transaction via `PaymentService`. |
| `displayTransactions()` | Retrieves all transactions for the order, determines the next allowed action (capture or refund), flags whether any credit‑card transaction exists. |
| Getters/Setters | Expose properties to the view layer (order, transactions, flags, amounts, etc.). |

**Frameworks / Libraries**  
- **Struts‑2** (`com.opensymphony.xwork2.validator.ValidationException`, action class).  
- **Apache Commons Lang** (`StringUtils`).  
- **Log4j** for logging.  
- Custom services from the `salesmanager` core (e.g., `PaymentService`, `OrderService`, `MerchantService`).  
- Utility classes (`CurrencyUtil`, `LabelUtil`, `MessageUtil`) for i18n, formatting, and error handling.  

## 2. Detailed Description  

### Initialization  
`TransactionDetailsAction` extends `BaseAction`, which supplies context handling (`getContext()`), authorization checks, and UI messaging.  
Before any business logic, the action verifies that the session contains a valid `Context` and that the order ID is non‑zero.

### `prepareOrderDetails()`  
1. Sets the page title (`label.order.paymentdetails.title`).  
2. Retrieves the current `Context` and merchant ID.  
3. Loads the `Order` using `OrderService`.  
4. Sets the formatted order total if it is greater than zero.  
5. Checks that the order is not `null` and belongs to the current merchant (`super.authorize(o)`).  
6. Stores the order in the action for later use.

### `processTransaction()`  
1. **Authorization & order loading** – uses `prepareOrderDetails()` and checks for an existing order.  
2. **Input validation** – verifies that the `process` parameter is present, numeric, and within the supported range (2 or 3).  
3. **Transaction execution**  
   - **Capture** (`PaymentConstants.CAPTURE`): calls `PaymentService.captureTransaction`.  
   - **Refund** (`PaymentConstants.REFUND`):  
     - Validates that `refundAmount` is supplied.  
     - Parses and validates the amount against the order total.  
     - Calls `OrderService.refundOrder` to create the refund transaction.  
4. **Success / Error handling** – sets success or technical messages, logs errors, and translates `TransactionException` codes into user‑friendly messages.  
5. Returns `SUCCESS` (or `INPUT` on validation error).

### `displayTransactions()`  
1. Loads the order as above.  
2. Calls `PaymentService.getTransactions(order)` to obtain all transaction VOs.  
3. Iterates through the list, identifies gateway transactions (`GatewayTransactionVO`), and populates:
   - `gatewaytransactions` (list of VOs).  
   - `creditcardtransaction` flag if any gateway transaction is found.  
4. Determines the most recent transaction (`lasttransaction`) based on `dateAdded`.  
5. Calculates `nextaction`:
   - If the last transaction was a **pre‑auth**, the next action is **capture**.  
   - If the last transaction was a **sale** or **capture**, the next action is **refund**.  
   - If the last transaction was a **refund** with a non‑zero amount, the next action remains **refund** (allowing partial refunds).  
   - Otherwise, `nextaction` is set to `-1` (no further action).

### Cleanup  
No explicit cleanup; the action relies on Struts‑2 lifecycle for request‑scoped data.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `prepareOrderDetails()` | Loads and validates the order; sets page title & total | None (uses `this.order`) | None | Throws `AuthorizationException` if unauthorized; updates `this.order` and `orderTotal`. |
| `processTransaction()` | Handles capture or refund request | None (uses action fields) | String (`SUCCESS`, `INPUT`, `AUTHORIZATIONEXCEPTION`) | Updates UI messages; may create/modify transactions. |
| `displayTransactions()` | Loads all transactions for the order; determines next action | None | String (`SUCCESS`, `AUTHORIZATIONEXCEPTION`) | Populates `gatewaytransactions`, `creditcardtransaction`, `nextaction`. |
| `getTransactions()` | Getter for `gatewaytransactions` | None | `List` | None |
| `isCreditcardtransaction()` | Flag indicating existence of gateway transaction | None | `boolean` | None |
| `getNextaction()` | Getter for next allowed action | None | `int` | None |
| `getProcess()` / `setProcess(String)` | Accessors for the process parameter | `String` | `String` | None |
| `getOrder()` / `setOrder(Order)` | Accessors for order object | `Order` | `Order` | None |
| `getOrderTotal()` / `setOrderTotal(String)` | Accessors for formatted order total | `String` | `String` | None |
| `getRefundAmount()` / `setRefundAmount(String)` | Accessors for refund amount entered by user | `String` | `String` | None |

**Reusable / Utility Methods**  
- `CurrencyUtil.displayFormatedAmountNoCurrency(...)` – formats amounts.  
- `CurrencyUtil.validateCurrency(...)` – parses and validates currency strings.  
- `LabelUtil.getInstance().getText(...)` – retrieves i18n strings.  
- `MessageUtil.addErrorMessage(...)` – helper for setting error messages in the request.  

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For string blank checks. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework. |
| `com.opensymphony.xwork2.validator.ValidationException` | Third‑party | Struts‑2 validation. |
| `com.salesmanager.*` | Internal | Core services, entities, constants, utilities. |
| `javax.servlet.http.HttpServletRequest` (via `BaseAction`) | Standard | Request handling. |

All dependencies are either standard J2EE APIs or well‑established open‑source libraries. The code assumes a servlet environment (session, request attributes).

## 5. Additional Notes  

### Strengths  
- **Clear separation of concerns**: business logic lives in service classes; action only orchestrates.  
- **Robust error handling** – custom `TransactionException` codes are translated into localized messages.  
- **Internationalization** – uses `LabelUtil` for all UI strings.  

### Potential Issues / Edge Cases  
1. **Floating‑point comparisons** – The code frequently uses `doubleValue()` on `BigDecimal` to compare amounts. This can lead to precision errors; it should use `compareTo` instead.  
2. **Hard‑coded numeric ranges** – `process` is validated only against values 2 and 3; adding new transaction types would require code changes. Consider using enums or a lookup table.  
3. **Thread safety** – `gatewaytransactions` is an instance field but is re‑used across requests (Struts‑2 actions are request‑scoped by default, but any pooling could expose concurrency issues). Ensure the action is not reused.  
4. **Missing null checks** – In `displayTransactions()`, `lasttransaction` may remain `null` if no gateway transactions exist; calling `lasttransaction.getMerchantPaymentGwAuthtype()` would throw a `NullPointerException`.  
5. **Magic numbers** – Many literal values (`-1`, `0`, `"0"`) are used without explanation; constants would improve readability.  
6. **Security** – The code trusts `this.getOrder().getOrderId()` and the `process` parameter without sanitization beyond numeric checks. Validate against the order’s state (e.g., ensure capture only on pre‑auth).  
7. **Exception handling** – The outer catch block swallows all exceptions except `AuthorizationException`. Logging is performed but the user receives a generic technical message; consider more granular feedback.  
8. **Internationalization of numeric validation errors** – The field error `"transaction.error.transactionamount"` is generic; providing context (e.g., required amount format) would help users.  

### Suggested Enhancements  
- **Use `BigDecimal.compareTo()`** for all monetary comparisons.  
- Replace magic numbers with named constants or an enum for transaction types.  
- Add null‑guard around `lasttransaction` when determining `nextaction`.  
- Validate the order state (e.g., `order.isAuthorized()`) before allowing capture or refund.  
- Extract transaction logic into a dedicated service method (`executeTransaction(Order, ProcessType, BigDecimal)`) to centralize validation and reduce duplication.  
- Implement unit tests for the action methods to cover all branches, especially the edge cases noted.  
- Introduce a `TransactionContext` object to encapsulate transaction state (type, amount, status) and pass it to services, improving readability and testability.  

Overall, the action is functional and adheres to standard Struts‑2 patterns, but refining the numeric handling, error resilience, and code maintainability would strengthen its robustness in a production environment.

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
package com.salesmanager.central.orders;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.validator.ValidationException;
import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.payment.MerchantPaymentGatewayTrx;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.service.payment.GatewayTransactionVO;
import com.salesmanager.core.service.payment.PaymentService;
import com.salesmanager.core.service.payment.SalesManagerTransactionVO;
import com.salesmanager.core.service.payment.TransactionException;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;

public class TransactionDetailsAction extends BaseAction {

	private Logger log = Logger.getLogger(TransactionDetailsAction.class);

	private Order order = null;

	private List gatewaytransactions = new ArrayList();
	private boolean creditcardtransaction = false;
	private int nextaction = -1;
	private String process;

	private String orderTotal = null;
	private String refundAmount = null;

	private void prepareOrderDetails() throws Exception {
		
		super.setPageTitle("label.order.paymentdetails.title");

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		// Get the order
		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);

		Order o = oservice.getOrder(this.getOrder().getOrderId());

		if(o.getTotal().doubleValue() > new BigDecimal("0").doubleValue()) {
			this.setOrderTotal(CurrencyUtil.displayFormatedAmountNoCurrency(o
				.getTotal(), ctx.getCurrency()));
		}

		// check if that entity realy belongs to merchantid
		if (o == null) {
			throw new AuthorizationException("Order is null for orderId "
					+ this.getOrder().getOrderId());
		}

		// Check if user is authorized (entity belongs to merchant)
		super.authorize(o);

		this.setOrder(o);

	}

	/**
	 * Process a new transaction, currently supports refund and capture
	 * following a pre-authorize
	 * 
	 * @return
	 * @throws Exception
	 */
	public String processTransaction() throws Exception {

		Context ctx = super.getContext();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(ctx.getMerchantid());

		try {

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			prepareOrderDetails();

			/** INPUT VALIDATION **/
			// validate the presence of a transaction type in the request
			// parameter
			if (this.getProcess() == null) {
				log
						.error("No transaction process id in request parameter. Require &process=1 or &process=2 or &process=3");
				return SUCCESS;
			}

			int process = -1;
			try {
				process = Integer.parseInt(this.getProcess());
			} catch (NumberFormatException nfe) {
				log.error("Can't parse process id in request parameter ["
						+ this.getProcess()
						+ "],  require &process=1 or &process=2 or &process=3");
				return SUCCESS;
			}

			// process 2 and 3 supported
			if (process < 2 || process > 3) {
				log.error("Transaction process type not supported "
						+ this.getProcess());
				return SUCCESS;
			}

			/** END VALIDATION **/

			PaymentService service = (PaymentService) ServiceFactory
					.getService(ServiceFactory.PaymentService);

			switch (process) {

			case PaymentConstants.CAPTURE:
				service.captureTransaction(store, this.getOrder());
				break;

			case PaymentConstants.REFUND:

				// get amount

				if (StringUtils.isBlank(this.getRefundAmount())) {
					super.addFieldError("orderTotal",
							"transaction.error.transactionamount");
					return INPUT;
				}

				BigDecimal originalAmount = this.getOrder().getTotal();

				BigDecimal newAmount = null;
				try {

					newAmount = CurrencyUtil.validateCurrency(this
							.getRefundAmount(), order.getCurrency());
				} catch (ValidationException e) {
					super.addFieldError("orderTotal",
							"transaction.error.transactionamount");
					return INPUT;
				}

				if (newAmount == null
						|| newAmount.floatValue() > originalAmount.floatValue()) {
					super.addFieldError("orderTotal",
							"transaction.error.transactionamounttoohigh");
					return INPUT;
				}

				OrderService oservice = (OrderService) ServiceFactory
						.getService(ServiceFactory.OrderService);
				oservice.refundOrder(this.getOrder(), newAmount, super
						.getLocale());

				break;

			default:
				log.error("Transaction process type not supported "
						+ this.getProcess());
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText("errors.technical"));

			}

			super.setSuccessMessage();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			if (e instanceof TransactionException) {
				TransactionException te = (TransactionException) e;
				// Display appropriate message to end user
				if (te.getErrorcode() != null
						&& !te.getErrorcode().trim().equals("")) {
					String textkey = "transaction.errors." + te.getErrorcode();
					if (te.getReason() != null
							&& !te.getReason().trim().equals("")) {
						MessageUtil.addErrorMessage(super.getServletRequest(),
								LabelUtil.getInstance().getText(textkey) + " ["
										+ te.getReason() + "]");
					} else {
						MessageUtil.addErrorMessage(super.getServletRequest(),
								LabelUtil.getInstance().getText(textkey));
					}
				} else {
					MessageUtil.addErrorMessage(super.getServletRequest(), te.getMessage() + " [" + te.getErrorcode() + "]");
					//super.setTechnicalMessage();
				}
			} else {
				log.error(e);
				super.setTechnicalMessage();
			}
		}

		return SUCCESS;
	}

	/**
	 * Displays transaction details
	 */
	public String displayTransactions() throws Exception {

		try {

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setTechnicalMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			prepareOrderDetails();

			PaymentService service = (PaymentService) ServiceFactory
					.getService(ServiceFactory.PaymentService);

			List smt = service.getTransactions(order);

			MerchantPaymentGatewayTrx lasttransaction = null;
			if (smt != null) {

				Iterator i = smt.iterator();

				while (i.hasNext()) {
					SalesManagerTransactionVO trx = (SalesManagerTransactionVO) i
							.next();
					if (trx instanceof GatewayTransactionVO) {
						this.creditcardtransaction = true;
						// downcast to the appropriate object
						GatewayTransactionVO gtx = (GatewayTransactionVO) trx;
						// Determines allowed actions
						MerchantPaymentGatewayTrx transaction = gtx
								.getTransactionDetails();
						if (lasttransaction != null) {
							if (transaction.getDateAdded().after(
									lasttransaction.getDateAdded())) {
								lasttransaction = transaction;
							}
						} else {
							lasttransaction = transaction;
						}
						gatewaytransactions.add(gtx);
					}
				}

				nextaction = Integer.parseInt(lasttransaction
						.getMerchantPaymentGwAuthtype()) + 1;
				int trtype = Integer.parseInt(lasttransaction
						.getMerchantPaymentGwAuthtype());

				if (trtype == PaymentConstants.PREAUTH) {
					nextaction = PaymentConstants.CAPTURE;
				} else if (trtype == PaymentConstants.SALE) {
					nextaction = PaymentConstants.REFUND;
				} else if (trtype == PaymentConstants.CAPTURE) {
					nextaction = PaymentConstants.REFUND;
				} else if (trtype == PaymentConstants.REFUND) {
					if (lasttransaction.getAmount().doubleValue() > new BigDecimal(
							"0").doubleValue()) {
						nextaction = PaymentConstants.REFUND;
					} else {
						nextaction = -1;
					}
				} else {
					nextaction = -1;
				}
			}

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public List getTransactions() {
		return gatewaytransactions;
	}

	public boolean isCreditcardtransaction() {
		return creditcardtransaction;
	}

	public int getNextaction() {
		return nextaction;
	}

	public String getProcess() {
		return process;
	}

	public void setProcess(String process) {
		this.process = process;
	}

	public Order getOrder() {
		return order;
	}

	public void setOrder(Order order) {
		this.order = order;
	}

	public String getOrderTotal() {
		return orderTotal;
	}

	public void setOrderTotal(String orderTotal) {
		this.orderTotal = orderTotal;
	}

	public String getRefundAmount() {
		return refundAmount;
	}

	public void setRefundAmount(String refundAmount) {
		this.refundAmount = refundAmount;
	}

}



```
