# CreditCardGatewayTransactionImpl.java

## Review

## 1. Summary

`CreditCardGatewayTransactionImpl` is an **abstract** helper that implements the `PaymentModule` and `CreditCardPaymentModule` interfaces.  
It provides the high‑level orchestration for three core operations:

| Method | Purpose |
|--------|---------|
| `processCapture(...)` | Finalises a pre‑authorised payment by capturing the transaction. |
| `processRefund(...)` | Refunds a previously authorised / captured transaction. |
| `processTransaction(...)` | Performs a *new* transaction (authorisation or authorisation‑and‑capture). |

Concrete payment gateway drivers must implement four abstract methods that perform the actual gateway calls:

```
authorizeAndCapture(...)
authorizeTransaction(...)
captureTransaction(...)
refundTransaction(...)
```

The class relies on several service objects (`PaymentService`, `MerchantService`, `CoreModuleService`) and configuration objects (`ConfigurationResponse`, `IntegrationKeys`, `IntegrationProperties`) to obtain gateway credentials, locale, and country‑specific integration services.  

Overall, the design follows a *template‑method* style: the abstract class defines the common flow, delegating the gateway‑specific details to the concrete subclasses.

---

## 2. Detailed Description

### 2.1 Core Flow

1. **Retrieve Transaction**  
   * For capture / refund – fetch the relevant transaction (capturable / refundable) from `PaymentService`.  
   * For a new transaction – nothing is retrieved; a fresh `Order` is used.

2. **Validate Module**  
   Checks that the transaction’s original payment module matches the requested one.

3. **Load Gateway Credentials**  
   * Build a configuration key (`MODULE_PAYMENT_GATEWAY + paymentModule`) – with a special case for PayPal.  
   * Ask `MerchantService` for a `ConfigurationResponse`.  
   * Extract `IntegrationKeys` and `IntegrationProperties`.

4. **Obtain Central Integration Service**  
   Uses `ModuleManagerImpl.getModuleServiceByCode(countryCode, paymentModule)` to acquire a `CoreModuleService` that knows how to talk to the gateway in that country.

5. **Delegate to Subclass**  
   Calls one of the four abstract methods, passing in all gathered context.

6. **Update Order Status**  
   - Capture: `STATUSDELIVERED`.  
   - Refund: `STATUSREFUND`.  
   - Transaction: sets status before/after authorisation, and finally `STATUSPROCESSING`.  

7. **Return the gateway response** (`GatewayTransactionVO` or `SalesManagerTransactionVO`).

### 2.2 Assumptions & Constraints

- **Single‑threaded order processing** – the class is stateless; thread‑safety is not an issue.  
- **Configuration must exist** – a missing configuration causes a `TransactionException`.  
- **Country code resolution** – relies on `CountryUtil.getCountryIsoCodeById()` to map a country ID to ISO‑3166 alpha‑2.  
- **Error propagation** – any exception is wrapped in a `TransactionException`.  
- **Credit card data** – temporarily stored on the `Order` for the transaction, then masked afterwards.

### 2.3 Architectural Observations

- **Template Method Pattern** – the abstract class encapsulates the algorithmic steps; concrete classes provide gateway specifics.  
- **Dependency Inversion** – `MerchantService` and `PaymentService` are retrieved via a `ServiceFactory`, but `PaymentService` is instantiated directly, breaking inversion for that dependency.  
- **Configuration‑Centric** – credentials and properties are stored centrally and accessed through a generic `ConfigurationResponse`.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `processCapture(...)` | Finalise a pre‑authorised payment | `Order`, `MerchantStore`, `Customer`, `paymentModule` | `GatewayTransactionVO` | Sets order status, logs errors |
| `processRefund(...)` | Issue a refund | `Order`, `MerchantStore`, `Customer`, `amount`, `paymentModule` | `GatewayTransactionVO` | Sets order status |
| `processTransaction(...)` | Execute a new transaction (auth or auth+capture) | `CoreModuleService`, `PaymentMethod`, `Order`, `Customer` | `SalesManagerTransactionVO` | Masks CC, sets order status, updates order with CC info |
| `authorizeAndCapture(...)` *(abstract)* | Authorise and immediately capture a transaction | `IntegrationKeys`, `IntegrationProperties`, `MerchantStore`, `Order`, `Customer`, `CoreModuleService` | `GatewayTransactionVO` | – |
| `authorizeTransaction(...)` *(abstract)* | Authorise a transaction (capture deferred) | Same as above | `GatewayTransactionVO` | – |
| `captureTransaction(...)` *(abstract)* | Capture a previously authorised transaction | `IntegrationKeys`, `IntegrationProperties`, `MerchantStore`, `Order`, `GatewayTransactionVO`, `Customer`, `CoreModuleService` | `GatewayTransactionVO` | – |
| `refundTransaction(...)` *(abstract)* | Refund a transaction | `IntegrationKeys`, `IntegrationProperties`, `MerchantStore`, `Order`, `GatewayTransactionVO`, `Customer`, `CoreModuleService`, `amount` | `GatewayTransactionVO` | – |

All methods are public except the abstract ones, which must be implemented by subclasses.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Simple string utilities |
| `org.apache.log4j.Logger` | Third‑party | Logging |
| `com.salesmanager.core.*` | Internal | Core entity, constants, services, utilities, configuration objects |
| `ServiceFactory` | Internal | Factory to obtain service implementations |
| `ModuleManagerImpl` | Internal | Resolves country‑specific `CoreModuleService` |
| `CountryUtil`, `CreditCardUtil`, `LabelUtil`, `LocaleUtil` | Internal | Utility helpers |

No external frameworks (e.g., Spring) are used; all services are obtained via a manual factory.

---

## 5. Additional Notes & Recommendations

### 5.1 Critical Bug – Null‑Pointer in Error Construction

```java
if (vo == null) {
    TransactionException te = new TransactionException(
        "Payment Gateway not configured for " + vo.getConfiguration("paymentmethod") + ... );
    throw te;
}
```

`vo` is null inside the `if` block, so calling `vo.getConfiguration(...)` throws an `NPE`. The same pattern appears in `processRefund` and `processTransaction`.  
**Fix**: Construct the message *after* verifying that `vo` is non‑null, or use a placeholder value.

```java
if (vo == null) {
    throw new TransactionException(
        "Payment Gateway not configured for merchantid " + order.getMerchantId());
}
```

### 5.2 Duplicate Logic

`processCapture`, `processRefund`, and `processTransaction` share almost identical code blocks for:
* Loading configuration
* Validating the merchant ID and store
* Obtaining the `CoreModuleService`

Extracting this into a private helper method would reduce duplication and improve maintainability.

### 5.3 Mixing Direct Instantiation with ServiceFactory

`PaymentService paymentservice = new PaymentService();` bypasses the factory, potentially breaking configuration or dependency injection. Consistently use `ServiceFactory.getService(ServiceFactory.PaymentService)`.

### 5.4 Property String Comparison

```java
if (props.getProperties1().equals(String.valueOf(PaymentConstants.PREAUTH))) {
    // ...
}
```

`PREAUTH` is an integer constant; converting it to a string for comparison is fragile. Prefer a dedicated enum or separate boolean flag, e.g., `props.isPreauth()`.

### 5.5 Order Status Logic

- In `processCapture`, status is set to `STATUSDELIVERED` *before* the transaction is actually captured.  
- In `processTransaction`, the status is changed multiple times (delivered → processing).  
Clarify the lifecycle semantics or move status updates to the concrete gateway implementation.

### 5.6 Security / Masking

The CC number is copied to the `Order` and later masked. The original number remains in memory until the method ends. Consider using a dedicated secure container or ensuring the data is cleared promptly.

### 5.7 Logging

Only errors for missing central services are logged. Other exceptional conditions (e.g., configuration missing, invalid credit card code) could benefit from logging for troubleshooting.

### 5.8 Error Messages

Messages sometimes include concatenated variables without spaces (e.g., `"Payment Gateway not configured for " + vo.getConfiguration("paymentmethod") + ", cannot retreive credentials for merchantid " + order.getMerchantId()`).  
Improve readability and internationalisation by using message templates or a localisation utility.

### 5.9 Code Style & Readability

* Method names follow Java conventions; however, comments could be expanded to explain complex blocks (e.g., key selection logic).  
* Use of `StringUtils.isBlank` is appropriate but sometimes redundant when the variable is known non‑null.  
* The class is heavily coupled to many internal services; consider injecting dependencies to make unit‑testing easier.

### 5.10 Future Enhancements

1. **Unit Testing** – Inject mock services via constructor or setters to test each flow independently.  
2. **Transaction Rollback** – In case of failure after partial updates (e.g., status set to delivered but capture fails), rollback the order state.  
3. **Metrics & Monitoring** – Add counters or timers to capture success/failure rates per gateway.  
4. **Extensibility** – Support new transaction types (e.g., recurring payments) by extending the abstract methods.

---

### Bottom Line

`CreditCardGatewayTransactionImpl` provides a solid, template‑method foundation for credit‑card payment integration.  
To reach production‑grade quality:

- Fix the NPE bug in configuration error handling.  
- Remove duplication, standardise service acquisition, and improve status handling.  
- Strengthen security handling of sensitive data and adopt type‑safe configuration flags.  
With these adjustments, the class becomes more robust, maintainable, and easier to test.

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
package com.salesmanager.core.module.impl.integration.payment;

import java.math.BigDecimal;
import java.util.Locale;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.OrderConstants;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.payment.CreditCard;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.module.model.integration.CreditCardPaymentModule;
import com.salesmanager.core.module.model.integration.PaymentModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.impl.ModuleManagerImpl;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.payment.GatewayTransactionVO;
import com.salesmanager.core.service.payment.PaymentService;
import com.salesmanager.core.service.payment.SalesManagerTransactionVO;
import com.salesmanager.core.service.payment.TransactionException;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CreditCardUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;

public abstract class CreditCardGatewayTransactionImpl implements PaymentModule, CreditCardPaymentModule {

	private Logger log = Logger.getLogger(CreditCardGatewayTransactionImpl.class);
	
	
	public GatewayTransactionVO processCapture(Order order, MerchantStore store, Customer customer, String paymentModule)
			throws TransactionException {
		
			//do common stuff
				
			try {
				
				//get transactions capturable for that order
				PaymentService paymentservice = new PaymentService();
				GatewayTransactionVO trx = paymentservice
						.getCapturableTransaction(order);

				if (trx == null) {
					TransactionException pe = new TransactionException(
							"No capturable transaction for orderid "
									+ order.getOrderId());
					pe.setErrorcode("01");
					throw pe;
				}

				if (trx.getTransactionDetails() != null
						&& !trx.getTransactionDetails()
								.getMerchantPaymentGwMethod().equals(
										paymentModule)) {
					TransactionException pe = new TransactionException(
							"Cannot use this payment module -> " + paymentModule + " to process transaction originaly created  with " + order.getPaymentModuleCode() + " for order id " + order.getOrderId());
					pe.setErrorcode("03");
					throw pe;
				}

				// Get credentials
				ConfigurationResponse vo = null;
				
				String key = PaymentConstants.MODULE_PAYMENT_GATEWAY + paymentModule;
				
				if(paymentModule.equals(PaymentConstants.PAYMENT_PAYPALNAME)) {
					key = PaymentConstants.MODULE_PAYMENT + paymentModule;
				}

				// Retrieve gateway configuration (merchantId and key)
				ConfigurationRequest configrequest = new ConfigurationRequest(order
						.getMerchantId(), key);
				MerchantService service = (MerchantService) ServiceFactory
						.getService(ServiceFactory.MerchantService);
				vo = service.getConfiguration(configrequest);

				if (vo == null) {
					TransactionException te = new TransactionException(
							"Payment Gateway not configured for "
									+ vo.getConfiguration("paymentmethod")
									+ ", cannot retreive credentials for merchantid "
									+ order.getMerchantId());
					throw te;
				}

				IntegrationKeys ik = (IntegrationKeys) vo.getConfiguration("keys");
				IntegrationProperties props = (IntegrationProperties) vo
						.getConfiguration("properties");

				if (order.getMerchantId() == 0) {
					throw new TransactionException("merchantId is not set in Order");
				}


				if (store == null) {
					throw new TransactionException(
							"MerchantStore is null for merchantId "
									+ order.getMerchantId());
				}
				
				
				String countrycode = CountryUtil.getCountryIsoCodeById(store
						.getCountry());
				CoreModuleService cis = ModuleManagerImpl.getModuleServiceByCode(
						countrycode, paymentModule);
				
				if (cis == null) {
					log.error("Central integration services not configured for "
							+ paymentModule
							+ " and country id " + store.getCountry());
					TransactionException pe = new TransactionException(
							"Central integration services not configured for "
									+ paymentModule
									+ " and country id " + store.getCountry());
					pe.setErrorcode("01");
					throw pe;
				}
				
				
				GatewayTransactionVO rvo= captureTransaction(
						ik, props, store, order, trx, customer, cis);
				
				
				order.setOrderStatus(OrderConstants.STATUSDELIVERED);
				
				return rvo;
				
			} catch (Exception e) {
				if (e instanceof TransactionException) {
					throw (TransactionException) e;
				} else {
					throw new TransactionException(e);
				}
			}
		
	}
	
	public GatewayTransactionVO processRefund(Order order, MerchantStore store, Customer customer, BigDecimal amount, String paymentModule)
			throws TransactionException {
		
			//do common stuff
		try {
			
			//get transactions refundable for that order
			PaymentService paymentservice = new PaymentService();
			GatewayTransactionVO trx = paymentservice
					.getRefundableTransaction(order);

			if (trx == null) {
				TransactionException pe = new TransactionException(
						"No capturable transaction for orderid "
								+ order.getOrderId());
				pe.setErrorcode("01");
				throw pe;
			}

			if (trx.getTransactionDetails() != null
					&& !trx.getTransactionDetails()
							.getMerchantPaymentGwMethod().equals(
									paymentModule)) {
				TransactionException pe = new TransactionException(
						"Cannot use this payment module -> " + paymentModule + " to process transaction originaly created  with " + order.getPaymentModuleCode() + " for order id " + order.getOrderId());
				pe.setErrorcode("03");
				throw pe;
			}

			// Get credentials
			ConfigurationResponse vo = null;
			
			String key = PaymentConstants.MODULE_PAYMENT_GATEWAY + paymentModule;
			
			if(paymentModule.equals(PaymentConstants.PAYMENT_PAYPALNAME)) {
				key = PaymentConstants.MODULE_PAYMENT + paymentModule;
			}

			// Retrieve gateway configuration (merchantId and key)
			ConfigurationRequest configrequest = new ConfigurationRequest(order
					.getMerchantId(), key);
			MerchantService service = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			vo = service.getConfiguration(configrequest);

			if (vo == null) {
				TransactionException te = new TransactionException(
						"Payment Gateway not configured for "
								+ vo.getConfiguration("paymentmethod")
								+ ", cannot retreive credentials for merchantid "
								+ order.getMerchantId());
				throw te;
			}

			IntegrationKeys ik = (IntegrationKeys) vo.getConfiguration("keys");
			IntegrationProperties props = (IntegrationProperties) vo
					.getConfiguration("properties");

			if (order.getMerchantId() == 0) {
				throw new TransactionException("merchantId is not set in Order");
			}



			if (store == null) {
				throw new TransactionException(
						"MerchantStore is null for merchantId "
								+ order.getMerchantId());
			}
			
			
			String countrycode = CountryUtil.getCountryIsoCodeById(store
					.getCountry());
			CoreModuleService cis = ModuleManagerImpl.getModuleServiceByCode(
					countrycode, paymentModule);

			if (cis == null) {
				log.error("Central integration services not configured for "
						+ paymentModule
						+ " and country id " + store.getCountry());
				TransactionException pe = new TransactionException(
						"Central integration services not configured for "
								+ paymentModule
								+ " and country id " + store.getCountry());
				pe.setErrorcode("01");
				throw pe;
			}
			
				
			GatewayTransactionVO rvo = refundTransaction(
					ik, props, store, order, trx, customer, cis, amount);
			
			
			order.setOrderStatus(OrderConstants.STATUSREFUND);
			
			return rvo;

			
		} catch (Exception e) {
			if (e instanceof TransactionException) {
				throw (TransactionException) e;
			} else {
				throw new TransactionException(e);
			}
		}
	}
	
	
	public SalesManagerTransactionVO processTransaction(CoreModuleService serviceDefinition,
			PaymentMethod paymentMethod, Order order, Customer customer)
			throws TransactionException {

		try {

			ConfigurationResponse vo = null;

			// Retreive gateway configuration (merchantId and key)
			ConfigurationRequest configrequest = new ConfigurationRequest(order
					.getMerchantId(), PaymentConstants.MODULE_PAYMENT_GATEWAY
					+ paymentMethod.getPaymentModuleName());
			MerchantService service = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			vo = service.getConfiguration(configrequest);

			if (vo == null) {
				TransactionException te = new TransactionException(
						"Payment Gateway not configured for "
								+ vo.getConfiguration("paymentmethod")
								+ ", cannot retreive credentials for merchantid "
								+ order.getMerchantId());
				throw te;
			}

			IntegrationKeys ik = (IntegrationKeys) vo.getConfiguration("keys");
			IntegrationProperties props = (IntegrationProperties) vo
					.getConfiguration("properties");

			if (order.getMerchantId() == 0) {
				throw new TransactionException("merchantId is not set in Order");
			}

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice.getMerchantStore(order
					.getMerchantId());

			if (store == null) {
				throw new TransactionException(
						"MerchantStore is null for merchantId "
								+ order.getMerchantId());
			}
			
			
			/*String countrycode = CountryUtil.getCountryIsoCodeById(store
					.getCountry());*/
	
			if (serviceDefinition == null) {
				log.error("Central integration services not configured for "
						+ paymentMethod.getPaymentModuleName()
						+ " and country id " + store.getCountry());
				TransactionException pe = new TransactionException(
						"Central integration services not configured for "
								+ paymentMethod.getPaymentModuleName()
								+ " and country id " + store.getCountry());
				pe.setErrorcode("01");
				throw pe;
			}
			

			// determine which kind of transaction needs to be processed
			// if transaction type = 1 = pre-authorization
			// if transaction type = 2 = authorizeAndCapture
			// properties2 =1-> prod 2-> dev
			
			if(!StringUtils.isBlank(paymentMethod.getPaymentModuleName())) {
				order.setPaymentModuleCode(paymentMethod.getPaymentModuleName());
			}
			
			//set payment information to order
			CreditCard creditCard = paymentMethod.getCreditCard();
			if(creditCard!=null) {
				
				
				if(creditCard.getCreditCardCode()==-1) {
					throw new TransactionException("Invalid credit card code");
				}
				
				Locale locale = LocaleUtil.getLocale(customer.getCustomerLang());
				
				creditCard.setLocale(locale);
				
				order.setCardType(creditCard.getCreditCardName());
				
				
				if(StringUtils.isBlank(paymentMethod.getPaymentMethodName())) {
				
					
					LabelUtil label = LabelUtil.getInstance();
					label.setLocale(locale);
					String cc = label.getText("label.creditcard");
					
					paymentMethod.setPaymentMethodName(cc);
	
				}
				
				if(!StringUtils.isBlank(paymentMethod.getPaymentMethodName()) && StringUtils.isBlank(order.getPaymentMethod())) {
					order.setPaymentMethod(paymentMethod.getPaymentMethodName());
				}
				

				if(!StringUtils.isBlank(creditCard.getCardNumber()) && StringUtils.isBlank(order.getCcNumber())) {
					order.setCcNumber(creditCard.getCardNumber());
				}
				if(!StringUtils.isBlank(creditCard.getExpirationMonth()) && !StringUtils.isBlank(creditCard.getExpirationYear())
						&& StringUtils.isBlank(order.getCcExpires())) {
					order.setCcExpires(creditCard.getExpirationMonth() + creditCard.getExpirationYear());
				}
				if(!StringUtils.isBlank(creditCard.getCvv()) && StringUtils.isBlank(order.getCcCvv())) {
					order.setCcCvv(creditCard.getCvv());
				}
				if(!StringUtils.isBlank(creditCard.getCardOwner()) && StringUtils.isBlank(order.getCcOwner())) {
					order.setCcOwner(creditCard.getCardOwner());
				}
			}
			
			SalesManagerTransactionVO rvo = null;
			if (props != null && !StringUtils.isBlank(props.getProperties1())) {
				if (props.getProperties1().equals(
						String.valueOf(PaymentConstants.PREAUTH))) {					
					rvo =  this.authorizeTransaction(ik, props, store, order, customer, serviceDefinition);
					order.setOrderStatus(OrderConstants.STATUSPROCESSING);
					

				} else {// preAuthAndCapture
					order.setOrderStatus(OrderConstants.STATUSDELIVERED);
					rvo = this.authorizeAndCapture(ik, props, store, order, customer, serviceDefinition);
					order.setOrderStatus(OrderConstants.STATUSPROCESSING);

				}
			} else {
				order.setOrderStatus(OrderConstants.STATUSDELIVERED);
				rvo =  this.authorizeAndCapture(ik, props, store, order, customer, serviceDefinition);
				order.setOrderStatus(OrderConstants.STATUSPROCESSING);
			}
			
			if(creditCard!=null) {
				//hash cc number
				String cardNumber = CreditCardUtil.maskCardNumber(creditCard.getCardNumber());
				order.setCcNumber(cardNumber);
			}
			
			return rvo;

		} catch (Exception e) {
			if (e instanceof TransactionException) {
				throw (TransactionException) e;
			} else {
				throw new TransactionException(e);
			}
		}

	}

	/**
	 * Does the authorization and capture for a credit card payment
	 * 
	 * @param origincountryid
	 * @param order
	 * @return
	 * @throws TransactionException
	 */
	public abstract GatewayTransactionVO authorizeAndCapture(
			IntegrationKeys keys, IntegrationProperties properties,
			MerchantStore store, Order order, Customer customer, CoreModuleService cms) throws TransactionException;

	/**
	 * Authorize a transaction This will require to capture the transaction
	 * after so the transaction is completed
	 * 
	 * @param origincountryid
	 * @param order
	 * @return
	 * @throws TransactionException
	 */
	public abstract GatewayTransactionVO authorizeTransaction(
			IntegrationKeys keys, IntegrationProperties properties,
			MerchantStore store, Order order, Customer customer, CoreModuleService cms) throws TransactionException;
	
	public abstract GatewayTransactionVO captureTransaction(
			IntegrationKeys keys, IntegrationProperties properties,
			MerchantStore store, Order order, GatewayTransactionVO trx, Customer customer, CoreModuleService cms) throws TransactionException;
	
	public abstract GatewayTransactionVO refundTransaction(
			IntegrationKeys keys, IntegrationProperties properties,
			MerchantStore store, Order order, GatewayTransactionVO trx, Customer customer, CoreModuleService cms, BigDecimal amount) throws TransactionException;

}



```
