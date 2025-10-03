# CheckoutBaseAction.java

## Review

## 1. Summary  
**Purpose & Scope**  
`CheckoutBaseAction` is an abstract Struts‑2 action that powers the checkout flow for the SalesManager e‑commerce platform. It handles:  

* **Payment initialization** – loading payment methods and credit‑card definitions.  
* **Order total calculation** – delegating to `OrderService` to compute totals and shipping.  
* **Credit‑card validation** – validating card data against business rules and merchant configuration.  
* **Session state management** – persisting order, totals, and payment flags across the checkout wizard.  

**Key Components**  

| Component | Role |
|-----------|------|
| `preparePayments()` | Loads available payment methods for the current merchant and populates the request. |
| `prepareCreditCards()` | Loads supported credit card types from a cache. |
| `updateOrderTotal(...)` | Calculates subtotal, taxes, shipping, etc., and stores intermediate data in the session. |
| `validateCreditCard(...)` | Performs client‑side credit‑card checks and attaches validated card information to the selected payment method. |
| `getPrincipal()` | Exposes the authenticated user wrapped in a `SalesManagerPrincipalProxy`. |

**Design Patterns & Frameworks**  

* **Struts‑2** – Action extends `BaseAction` and uses `HttpServletRequest`/`HttpSession`.  
* **Factory** – `ServiceFactory` is used to obtain business services (`OrderService`, `MerchantService`).  
* **Cache** – `RefCache` holds a map of supported credit cards.  
* **Proxy** – `SalesManagerPrincipalProxy` wraps the `Principal`.  
* **Utility classes** – `PaymentUtil`, `CreditCardUtil`, `OrderUtil`, etc., provide reusable logic.  

## 2. Detailed Description  
1. **Initialization**  
   * When the checkout page is requested, `preparePayments()` is invoked. It retrieves the current `MerchantStore`, fetches all available payment methods for the store’s ID and locale via `PaymentUtil`, and sets the result in the request under `"PAYMENTS"`.  
   * The method also calls `prepareCreditCards()` which pulls a cached map of supported credit cards, turns it into a list, and assigns it to `creditCards`.  
   * The flag `hasPayment` is set based on the session attribute `"HAS_PAYMENT"`.

2. **Session/Request Interaction**  
   * Order totals and lists are stored in the session through `SessionUtil`.  
   * After totals are computed, the method stores the list of `OrderProduct` and the updated `Order` in the session.

3. **Order Total Calculation**  
   * `updateOrderTotal(...)` has three overloads; the most complete one receives an `Order`, the list of `OrderProduct`, an optional `Customer`, optional `Shipping`, and the `MerchantStore`.  
   * It obtains `OrderService`, delegates `calculateTotal()`, and then converts the resulting `OrderTotalSummary` into a list of `OrderTotal` objects.  
   * The method also flags whether a payment is needed: if the subtotal is zero, `hasPayment` is set to `false`.  
   * Locale and currency are applied to all entities via `LocaleUtil` and `OrderUtil`.

4. **Credit‑Card Validation**  
   * The method expects a `PaymentMethod` and the `merchantId`.  
   * Validation logic is split into two parts:  
     * Basic format checks (nulls, blank fields).  
     * `CreditCardUtil` performs card number and CVV checks.  
   * If the merchant’s payment module requires CVV (indicated by `properties.getProperties3().equals("2")`), `validateCvv()` is invoked.  
   * On success, the validated `CreditCard` is stored in the `PaymentMethod` config and the payment method’s name is set to a localized “Credit Card” label.

5. **Utility Methods**  
   * `getCreditCardYears()` returns a list of the next 10 years.  
   * `getCreditCardMonths()` returns month strings “01”‑“12”.  

6. **Getter/Setter**  
   * Standard accessors for credit cards, payment methods, and the `hasPayment` flag.  

### Assumptions & Constraints  

| Assumption | Impact |
|------------|--------|
| `MerchantStore` is always present in the session | The code will throw NPE if missing. |
| `PaymentUtil.getPaymentMethods()` returns a map keyed by `String` | No error handling if map is null. |
| `RefCache` contains all supported credit cards | Cache misses are silently ignored. |
| `OrderService.calculateTotal()` can throw checked exceptions | The code propagates these, but the calling action may need to handle them. |
| The session always has `"PRINCIPAL"` attribute for authenticated users | If not, `getPrincipal()` returns null. |
| `IntegrationProperties.getProperties3()` is a string “2” for CVV requirement | No null‑check, risk of NPE. |
| The `Locale` is provided by `BaseAction` | Not explicitly validated. |

### Architecture & Design Choices  

* **Service Layer** – Business logic is delegated to `OrderService` and `MerchantService`.  
* **Separation of Concerns** – UI logic (request/session handling) is distinct from payment validation.  
* **Reusability** – The class is abstract; concrete checkout actions extend it and only provide specifics (e.g., shipping options).  
* **Extensibility** – Adding a new payment module requires only a new `PaymentMethod` and possibly a new `ConfigurationProperties` entry.

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `preparePayments()` | Load payment methods & credit cards, set request attributes. | None | None | Sets `"PAYMENTS"` request attribute, updates `hasPayment`. |
| `getPrincipal()` | Wrap the `Principal` from session. | None | `PrincipalProxy` or `null` | Reads session. |
| `prepareCreditCards()` | Load supported credit cards into `creditCards`. | None | None | Sets `creditCards`. |
| `updateOrderTotal(Order, List, MerchantStore)` | Wrapper that calls full overload with nulls. | `Order`, `List`, `MerchantStore` | `OrderTotalSummary` | Updates order totals, session. |
| `updateOrderTotal(Order, List, Customer, MerchantStore)` | Wrapper adding optional customer. | `Order`, `List`, `Customer`, `MerchantStore` | `OrderTotalSummary` | Same as above. |
| `updateOrderTotal(Order, List, Customer, Shipping, MerchantStore)` | Core calculation logic. | `Order`, `List`, `Customer`, `Shipping`, `MerchantStore` | `OrderTotalSummary` | Persists order, totals, product list; sets locale/currency. |
| `validateCreditCard(PaymentMethod, int)` | Validate credit card info and attach to payment method. | `PaymentMethod`, `merchantId` | None | Adds field errors, populates payment method config. |
| `getCreditCardYears()` | Return list of next 10 years. | None | `Collection<Integer>` | None. |
| `getCreditCardMonths()` | Return list of months “01”‑“12”. | None | `Collection<String>` | None. |
| `getCreditCard()` | Getter. | None | `CreditCard` | None. |
| `setCreditCard(CreditCard)` | Setter. | `CreditCard` | None | Sets `creditCard`. |
| `getCreditCards()` | Getter. | None | `Collection` | None. |
| `setCreditCards(Collection)` | Setter. | `Collection` | None | Sets `creditCards`. |
| `isHasPayment()` | Getter. | None | `boolean` | None. |
| `setHasPayment(boolean)` | Setter. | `boolean` | None | Sets flag. |
| `getPaymentMethods()` | Getter. | None | `Map` | None. |
| `setPaymentMethods(Map)` | Setter. | `Map` | None | Sets map. |

### Reusable / Utility Methods  
* `validateCreditCard` is a good candidate for extraction into a separate validator service.  
* `getCreditCardYears/Months` could be shared in a common UI util class.  

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For blank checks. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `org.apache.struts2.interceptor.PrincipalProxy` | Third‑party | Struts2 security. |
| `com.salesmanager.*` | Internal | Core entities, services, utilities, and cache. |
| `javax.servlet.http.HttpSession` | Standard | Web session handling. |
| `java.util` collections, `Calendar`, `BigDecimal` | Standard | Core Java. |

**Platform Assumptions**  
* Servlet container with HTTP session support.  
* Struts‑2 action lifecycle is in effect.  
* `ServiceFactory` is correctly configured to supply services.  

## 5. Additional Notes  

### Strengths  

* **Clear Separation** – UI‑specific logic is kept in this base action while heavy lifting is off‑loaded to services.  
* **Session Management** – Order, totals, and payment flags are consistently stored in the session, simplifying the wizard flow.  
* **Locale Handling** – Explicitly sets locale and currency on entities, ensuring consistent formatting.  

### Potential Issues & Edge Cases  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Null‑pointer risks** – e.g., `paymentMethod.getPaymentModuleName()` could be null; `properties.getProperties3()` may be null. | Runtime crashes during checkout. | Add defensive null checks or fail‑fast validation. |
| **Hardcoded “2” for CVV requirement** – magic string. | Maintenance difficulty. | Use constants or enum. |
| **No error handling for `PaymentUtil.getPaymentMethods()` returning null**. | Silent failures. | Log warning and fall back to empty map. |
| **`updateOrderTotal` mutates `order` and session state** – not thread‑safe if session accessed concurrently. | Concurrency bugs in high‑traffic scenarios. | Document session scope; consider immutable DTOs. |
| **Locale/Currency conversion** – relies on `LocaleUtil` and `OrderUtil` without visibility of their behavior. | Possible formatting inconsistencies. | Validate that these utilities handle all locales. |
| **Session attribute names are hard‑coded** – e.g., `"ORDER_PRODUCT_LIST"`. | Tight coupling to string constants. | Define constants in a central config. |
| **Credit‑card month list uses string literals** – not locale‑aware. | Non‑English month display may look odd. | Use `DateFormatSymbols` for localized month names. |
| **`validateCreditCard` throws generic `Exception`** – caller must catch. | Exception handling burden on actions. | Declare specific exception types or handle within the method. |
| **`getPrincipal()` returns null when no principal** – might lead to NPE in callers. | Hidden bugs in derived actions. | Return an empty `SalesManagerPrincipalProxy` or throw custom `UnauthorizedException`. |
| **`hasPayment` flag logic duplicated** – both set in `preparePayments` and `updateOrderTotal`. | Inconsistent state if methods are called out of order. | Centralize flag calculation in a single place. |

### Future Enhancements  

1. **Extract Validation to a Dedicated Service** – `CreditCardValidator` can encapsulate all rules and be unit‑tested separately.  
2. **Use Dependency Injection** – Inject `OrderService`, `MerchantService`, `PaymentUtil`, etc., instead of static factory calls.  
3. **Implement Logging Levels** – Add debug logs for each major step (e.g., payment method retrieval, total calculation).  
4. **Unit Tests** – Provide tests for `updateOrderTotal` and `validateCreditCard` covering success, CVV required, CVV missing, and invalid card numbers.  
5. **Error Handling API** – Return structured error codes or messages instead of field errors for API‑centric flows.  
6. **Internationalization** – Replace hard‑coded month strings and magic values with resource bundles.  
7. **Session Cleanup** – Provide a method to purge checkout session data after order completion.  

### Code Style & Readability  

* **Magic Strings** – `"PAYMENTS"`, `"ORDER_PRODUCT_LIST"`, `"HAS_PAYMENT"` should be constants.  
* **Exception Handling** – `validateCreditCard` catches `Exception` and re‑throws; better to catch only expected types.  
* **Variable Naming** – `ccmap`, `objects` are ambiguous; use descriptive names (`creditCardMap`, `orderProductArray`).  
* **Logging** – Only one `log.error(e)` in `preparePayments`; consider logging at debug level for normal operations.  
* **Commenting** – Some methods have brief comments; adding JavaDoc would improve maintainability.  

Overall, the class provides a solid foundation for checkout logic but would benefit from stronger null‑safety, better separation of concerns, and more robust error handling.

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

import java.math.BigDecimal;
import java.security.Principal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.apache.struts2.interceptor.PrincipalProxy;

import com.salesmanager.checkout.util.PaymentUtil;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderTotal;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.payment.CreditCard;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.reference.CentralCreditCard;
import com.salesmanager.core.entity.shipping.Shipping;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.CreditCardUtil;
import com.salesmanager.core.util.CreditCardUtilException;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.OrderUtil;
import com.salesmanager.core.util.www.BaseAction;
import com.salesmanager.core.util.www.SalesManagerPrincipalProxy;
import com.salesmanager.core.util.www.SessionUtil;

public abstract class CheckoutBaseAction extends BaseAction {

	private Collection creditCards;
	protected Map paymentMethods;

	private CreditCard creditCard;
	private Logger log = Logger.getLogger(CheckoutBaseAction.class);
	private java.util.Calendar cal = new java.util.GregorianCalendar();

	private boolean hasPayment = true;// flag indicating if the product has to
										// be paid

	protected void preparePayments() {

		try {
			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());
			paymentMethods = PaymentUtil.getPaymentMethods(store
					.getMerchantId(), super.getLocale());
			prepareCreditCards();
			super.getServletRequest().setAttribute("PAYMENTS", paymentMethods);
			hasPayment = SessionUtil.isHasPayment(getServletRequest());

		} catch (Exception e) {
			log.error(e);
		}
	}

	public PrincipalProxy getPrincipal() {
		HttpSession session = this.getServletRequest().getSession();
		Principal p = (Principal) session.getAttribute("PRINCIPAL");

		if (p != null) {

			SalesManagerPrincipalProxy proxy = new SalesManagerPrincipalProxy(p);
			return proxy;

		} else {
			return null;
		}
	}

	protected void prepareCreditCards() {

		Map ccmap = com.salesmanager.core.service.cache.RefCache
				.getSupportedCreditCards();
		if (ccmap != null) {
			creditCards = new ArrayList();
			Iterator i = ccmap.keySet().iterator();

			while (i.hasNext()) {
				int key = (Integer) i.next();
				CentralCreditCard ccc = (CentralCreditCard) ccmap.get(key);
				creditCards.add(ccc);
			}
		}

	}

	protected OrderTotalSummary updateOrderTotal(Order order, List products,
			MerchantStore store) throws Exception {
		return updateOrderTotal(order, products, null, null, store);
	}

	protected OrderTotalSummary updateOrderTotal(Order order, List products,
			Customer customer, MerchantStore store) throws Exception {
		return updateOrderTotal(order, products, customer, null, store);
	}

	protected OrderTotalSummary updateOrderTotal(Order order, List products,
			Customer customer, Shipping shipping, MerchantStore store)
			throws Exception {

		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);

		// for displaying the order summary, need to create an OrderSummary
		// entity
		OrderTotalSummary summary = oservice.calculateTotal(order, products,
				customer, shipping, store.getCurrency(), super.getLocale());

		SessionUtil.setHasPayment(true, super.getServletRequest());

		// if order is free, then no payment
		if (summary.getOneTimeSubTotal().toString().equals(
				new BigDecimal("0.00").toString())) {
			this.setHasPayment(false);
			SessionUtil.setHasPayment(false, super.getServletRequest());
		}


		order.setTotal(summary.getTotal());

		Map totals = OrderUtil.getOrderTotals(order.getOrderId(), summary,
				store.getCurrency(), super.getLocale());

		// @TODO change to ORDERPRODUCTS
		super.getServletRequest().getSession().setAttribute(
				"ORDER_PRODUCT_LIST", products);

		// transform totals to a list
		List totalsList = new ArrayList();
		if (totals != null) {
			Iterator totalsIterator = totals.keySet().iterator();
			while (totalsIterator.hasNext()) {
				String key = (String) totalsIterator.next();
				OrderTotal total = (OrderTotal) totals.get(key);
				totalsList.add(total);
			}
		}

		SessionUtil.setOrderTotals(totalsList, getServletRequest());

		OrderProduct[] opArray = new OrderProduct[products.size()];
		OrderProduct[] objects = (OrderProduct[]) products.toArray(opArray);
		summary.setOrderProducts(objects);

		order.setRecursiveAmount(summary.getRecursiveSubTotal());

		order.setLocale(super.getLocale());
		order.setCurrency(store.getCurrency());

		// Set orderProducts = order.getOrderProducts();
		LocaleUtil.setLocaleToEntityCollection(products, super.getLocale(),
				store.getCurrency());

		SessionUtil.setOrder(order, getServletRequest());

		return summary;

	}

	/**
	 * Utility method for doing credit card validation
	 * 
	 * @throws Exception
	 */
	protected void validateCreditCard(PaymentMethod paymentMethod,
			int merchantId) throws Exception {

		if (creditCard == null) {
			super.addFieldError("creditCard.cardNumber",
					getText("errors.creditcard.missinginformation"));
			return;
		}

		if (StringUtils.isBlank(creditCard.getCardNumber())) {
			super.addFieldError("creditCard.cardNumber",
					getText("errors.creditcard.missinginformation"));
			return;
		}

		try {

			CreditCardUtil ccUtil = new CreditCardUtil();
			ccUtil.validate(creditCard.getCardNumber(), creditCard
					.getCreditCardCode(), creditCard.getExpirationMonth(),
					creditCard.getExpirationYear());
			// if need to validate cvv

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationResponse vo = mservice.getConfigurationByModule(
					paymentMethod.getPaymentModuleName(), merchantId);

			if (vo == null) {
				throw new Exception("No configuration for payment module "
						+ paymentMethod.getPaymentModuleName()
						+ " for merchant ID " + merchantId);
			}

			IntegrationProperties properties = (IntegrationProperties) vo
					.getConfiguration("properties");
			if (properties != null && properties.getProperties3().equals("2")) {
				ccUtil.validateCvv(creditCard.getCvv(), creditCard
						.getCreditCardCode());
			}

			this.getCreditCard().setLocale(super.getLocale());
			paymentMethod.setType(1);
			paymentMethod.addConfig("CARD", getCreditCard());
			
			Locale locale = super.getLocale();
			
			LabelUtil label = LabelUtil.getInstance();
			label.setLocale(locale);
			String cc = label.getText("label.creditcard");
			
			paymentMethod.setPaymentMethodName(cc);

		} catch (Exception e) {
			if (e instanceof CreditCardUtilException) {
				CreditCardUtilException cce = (CreditCardUtilException) e;
				if (cce.getErrorType() == CreditCardUtilException.CVV) {
					super.addFieldError("creditCard.creditCardCode", cce
							.getMessage());
					addFieldMessage("creditCard.creditCardCode", cce
							.getMessage());
				} else if (cce.getErrorType() == CreditCardUtilException.DATE) {
					super.addFieldError(
							"creditCard.creditCard.expirationMonth", cce
									.getMessage());
					addFieldMessage("creditCard.creditCard.expirationMonth",
							cce.getMessage());
				} else {
					super.addFieldError("creditCard.cardNumber", cce
							.getMessage());
					addFieldMessage("creditCard.cardNumber", cce.getMessage());
				}
				return;
			}
			throw e;
		}

	}

	public Collection getCreditCardYears() {
		List l = new ArrayList();
		int yearNow = cal.get(java.util.Calendar.YEAR);
		for (int i = 0; i < 10; i++) {
			int y = yearNow + i;
			l.add(y);
		}
		return l;
	}

	public Collection getCreditCardMonths() {
		List l = new ArrayList();
		l.add("01");
		l.add("02");
		l.add("03");
		l.add("04");
		l.add("05");
		l.add("06");
		l.add("07");
		l.add("08");
		l.add("09");
		l.add("10");
		l.add("11");
		l.add("12");
		return l;
	}

	public CreditCard getCreditCard() {
		return creditCard;
	}

	public void setCreditCard(CreditCard creditCard) {
		this.creditCard = creditCard;
	}

	public Collection getCreditCards() {
		return creditCards;
	}

	public void setCreditCards(Collection creditCards) {
		this.creditCards = creditCards;
	}

	public boolean isHasPayment() {
		return hasPayment;
	}

	public void setHasPayment(boolean hasPayment) {
		this.hasPayment = hasPayment;
	}

	public Map getPaymentMethods() {
		return paymentMethods;
	}

	public void setPaymentMethods(Map paymentMethods) {
		this.paymentMethods = paymentMethods;
	}

}



```
