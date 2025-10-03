# PaymentService.java

## Review

## 1. Summary  

**Purpose**  
`PaymentService` is the central façade for all payment‑related operations in the SalesManager platform. It orchestrates the interaction between orders, customers, merchant configurations and the various payment modules (credit‑card gateways, PayPal, etc.) and persists transaction data.

**Key components**

| Class/Interface | Role |
|-----------------|------|
| `PaymentService` | High‑level API exposed to other modules (orders, checkout, admin). Delegates to specific payment modules. |
| `PaymentModule` / `CreditCardPaymentModule` | Strategy pattern – each concrete module implements the same interface and is looked up by name via Spring. |
| DAO classes (`IMerchantPaymentGatewayTrxDao`, `IOffsystemPendingOrderDao`, …) | Persist data to the database. |
| `ServiceFactory` | Simple static service locator used to obtain non‑Spring services (e.g. `ReferenceService`). |
| `CoreModuleService` | Encapsulates locale‑specific logic (country/ISO handling). |
| `TransactionImpl` | Helper that queries the transaction database and filters by type. |

**Frameworks / Libraries**

* Spring (dependency injection, transaction demarcation, bean lookup).  
* Apache Commons Lang (`StringUtils`).  
* JPA/Hibernate (through DAO interfaces, though not shown directly).  
* Standard Java (`java.util`, `java.math`, `java.lang`).  

The class heavily relies on **Spring’s `@Service`** annotation and **`@Transactional`** boundaries to manage persistence and rollbacks.

---

## 2. Detailed Description  

### Flow of execution

1. **Payment configuration**  
   - `getConfiguredPaymentMethods()` queries `MerchantConfiguration` by merchant ID and the payment indicator key.

2. **Order processing**  
   - `processPaymentTransaction()` is the main entry point.  
     * Determines the payment module code (`pm`) from the order or the supplied `PaymentMethod`.  
     * If still blank, it looks up the merchant configuration for a credit‑card gateway and extracts the module name.  
     * Retrieves the corresponding `CoreModuleService` (country‑specific).  
     * Instantiates the payment module via `SpringUtil.getBean(pm)`.  
     * Delegates the actual transaction to the module’s `processTransaction()` method.  
     * Marks the order channel as `ONLINE`.  

3. **Pre‑initialisation / Express Checkout**  
   - `preInitializePayment()` calls the module’s `initTransaction()` (e.g., PayPal Express).

4. **Refund / Capture / Pre‑auth**  
   - Methods such as `refundTransaction()`, `captureTransaction()`, `getRefundableTransaction()`, and `getCapturableTransaction()` locate the correct module, verify configuration, fetch the customer, and delegate to the module’s specific method (`processRefund()`, `processCapture()`, etc.).

5. **Persisting transaction data**  
   - `saveMerchantPaymentGatewayTrx()` and `saveOrUpdateOffsystemPendingOrder()` persist transaction entities using DAOs.

6. **Utility / lookup helpers**  
   - `getTransactions()`, `getPaymentMethodsList()`, `getPaymentMethods()` provide read‑only helpers that wrap DAO or service calls.

### Assumptions & Constraints

* Payment modules are defined as Spring beans whose names match the module code (`PaymentConstants.PAYMENT_*`).  
* All modules implement either `PaymentModule` or `CreditCardPaymentModule`.  
* Merchant configuration must contain a `configurationValue` (gateway key) and optionally a `configurationValue1` for credit‑card modules.  
* The code expects the `order` to already have a `paymentModuleCode` or a `PaymentMethod` object with a non‑blank module name.  
* Transaction persistence is handled via DAO; the service does not expose transaction IDs to callers except through returned VO objects.

### Architecture & Design Choices

* **Strategy pattern** – payment modules are selected at runtime and injected by name.  
* **Service Locator** – `ServiceFactory.getService()` is used instead of autowiring for non‑Spring services (`ReferenceService`, `MerchantService`, etc.).  
* **Transactional boundaries** – each public method that mutates data is annotated with `@Transactional`.  
* **DTO/VO objects** – `SalesManagerTransactionVO` and `GatewayTransactionVO` are used to transfer data between layers.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side effects |
|--------|---------|--------|---------|--------------|
| `getConfiguredPaymentMethods(int)` | Retrieve payment configurations for a merchant | `merchantId` | Collection of `MerchantConfiguration` | None |
| `recordOffSystemPayment(String, Order)` | Placeholder for off‑system payment handling (unimplemented) | `paymentModuleName`, `order` | None | Throws `Exception` |
| `saveOrUpdateOffsystemPendingOrder(OffsystemPendingOrder, Order)` | Persist or update a pending off‑system order | `pending`, `order` | None | Persists `OffsystemPendingOrder` |
| `saveMerchantPaymentGatewayTrx(MerchantPaymentGatewayTrx)` | Persist a payment transaction record | `trx` | None | Persists `MerchantPaymentGatewayTrx` |
| `processPaymentTransaction(MerchantStore, Order, Customer, PaymentMethod)` | Main payment workflow: determine module, init, delegate | `store`, `order`, `customer`, `paymentMethod` | `SalesManagerTransactionVO` | May modify order (`paymentModuleCode`, `channel`) |
| `preInitializePayment(MerchantStore, Order)` | Initialise a payment (e.g., PayPal Express) | `store`, `order` | `Map<String,String>` | None |
| `getTransactions(Order)` | Retrieve all transactions for an order | `order` | `List<SalesManagerTransactionVO>` | None |
| `refundTransaction(MerchantStore, Order, BigDecimal)` | Refund a captured/sale transaction | `store`, `order`, `amount` | `SalesManagerTransactionVO` | Delegates to module |
| `findMerchantPaymentGatewayTrxByMerchantIdAndOrderId(int, long)` | Query transactions by merchant & order | `merchantId`, `orderId` | `Collection<MerchantPaymentGatewayTrx>` | None |
| `captureTransaction(MerchantStore, Order)` | Capture a pre‑authorized transaction | `store`, `order` | `GatewayTransactionVO` | Delegates to module |
| `getRefundableTransaction(Order)` | Find a transaction eligible for refund | `order` | `GatewayTransactionVO` | Delegates to `TransactionImpl` |
| `getCapturableTransaction(Order)` | Find a transaction eligible for capture | `order` | `GatewayTransactionVO` | Delegates to `TransactionImpl` |
| `getPaymentMethodsList(String)` | Retrieve country‑specific payment modules | `countryIsoCode` | `List<CoreModuleService>` | None |
| `getPaymentMethods()` | Retrieve all payment modules | None | `List<CoreModuleService>` | None |

### Reusable/Utility Methods

* `ServicesUtil.getPaymentMethodsList()` and `ServicesUtil.getPaymentMethods()` – static helpers that centralise module discovery.  
* `SpringUtil.getBean(String)` – used for dynamic bean lookup.  
* `ReferenceService.getCoreModuleService()` – obtains a country‑specific service that may contain locale‑specific logic.

---

## 4. Dependencies  

| External Library / API | Purpose |
|------------------------|---------|
| **Spring Framework** (`@Service`, `@Transactional`, `SpringUtil`, `ServiceFactory`) | Dependency injection, transaction management, bean lookup. |
| **Apache Commons Lang** (`StringUtils`) | String manipulation utilities. |
| **JPA / Hibernate** (implied by DAO interfaces) | ORM mapping for persistence. |
| **Java Standard Library** (`java.util`, `java.math`) | Collections, dates, numbers. |

*All dependencies are third‑party except the JDK.*  
*No platform‑specific code is visible; the service is intended to run in a typical Java EE / Spring container.*

---

## 5. Additional Notes  

### Strengths

* **Clear separation of concerns** – payment logic is isolated in modules; DAO handles persistence.  
* **Transactional safety** – each write method is wrapped in `@Transactional`.  
* **Extensibility** – new payment modules can be added by implementing `PaymentModule` and configuring a Spring bean.  

### Potential Issues / Edge Cases  

1. **Missing implementation**  
   * `recordOffSystemPayment()` throws `Exception("not implemented")`. If called, the system will fail silently.  
2. **Type safety / Raw types**  
   * DAO queries return `Collection<MerchantConfiguration>` but `TransactionImpl` uses raw `List` and `int[]` arrays.  
   * Use generics everywhere to avoid `ClassCastException`.  
3. **Error handling**  
   * Most methods catch `Exception` and wrap it in a custom `TransactionException`. This hides the original cause for some callers.  
   * `ServiceFactory` is a static lookup; if a service is not registered it will return `null` and cause `NullPointerException` later.  
4. **Configuration lookup duplication**  
   * Several methods (refund, capture) repeat the same block of code to fetch merchant configuration. Extract to a helper.  
5. **Concurrency / State leakage**  
   * The service holds no mutable state, so it is thread‑safe. However, `order.setChannel(OrderConstants.ONLINE_CHANNEL);` modifies the order passed in, which may be unexpected for callers.  
6. **Hard‑coded strings**  
   * Strings such as `"Payment module " + paymentModule + " is not implemented in the module list"` can be externalised for localisation.  
7. **Order channel hard‑coding**  
   * Setting the channel to `ONLINE_CHANNEL` inside the service couples business logic with the payment flow. Consider moving this to the controller or a dedicated order service.  

### Suggested Enhancements

| Area | Recommendation |
|------|----------------|
| **Module Discovery** | Replace `SpringUtil.getBean(pm)` with autowired `Map<String, PaymentModule>` to let Spring inject all modules at startup. |
| **Configuration Service** | Abstract configuration lookup into a dedicated `PaymentConfigurationService` to avoid duplicated code. |
| **Error Handling** | Define specific exception types (`PaymentModuleNotFoundException`, `MerchantConfigurationMissingException`) instead of generic `TransactionException`. |
| **Unit Tests** | Add mocks for DAOs and modules to unit‑test each method’s logic without hitting the database. |
| **DTO Validation** | Validate `PaymentMethod` and `Order` before proceeding (e.g., non‑null, mandatory fields). |
| **Logging** | Add structured logging (SLF4J) for tracing payment flows and errors. |
| **Transactional Granularity** | Review transaction boundaries; e.g., `processPaymentTransaction` may need to span multiple DAO calls. |
| **Performance** | Cache `CoreModuleService` lookups per country to avoid repeated `ReferenceService` calls. |
| **Internationalisation** | Externalise all hard‑coded messages. |

---

### Final Verdict  

`PaymentService` is a well‑structured façade that encapsulates complex payment logic behind a clean interface. It leverages Spring for dependency injection and transaction management, and adheres to several good design patterns (Strategy, DAO). However, the code suffers from duplicated configuration logic, some missing implementations, and limited type safety. Addressing the issues above will improve maintainability, testability, and robustness of the payment subsystem.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-3 Sep, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.service.payment;

import java.math.BigDecimal;
import java.util.Collection;
import java.util.Date;
import java.util.List;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.salesmanager.core.constants.OrderConstants;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.payment.MerchantPaymentGatewayTrx;
import com.salesmanager.core.entity.payment.OffsystemPendingOrder;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.module.model.integration.CreditCardPaymentModule;
import com.salesmanager.core.module.model.integration.PaymentModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.impl.ServicesUtil;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.merchant.impl.dao.IMerchantConfigurationDao;
import com.salesmanager.core.service.payment.impl.TransactionImpl;
import com.salesmanager.core.service.payment.impl.dao.IMerchantPaymentGatewayTrxDao;
import com.salesmanager.core.service.payment.impl.dao.IOffsystemNotificationOrderDao;
import com.salesmanager.core.service.payment.impl.dao.IOffsystemPendingOrderDao;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.PaymentUtil;
import com.salesmanager.core.util.SpringUtil;



@Service
public class PaymentService {



	@Autowired
	private IMerchantPaymentGatewayTrxDao MerchantPaymentGatewayTrxDao;


	@Autowired
	private IOffsystemNotificationOrderDao offsystemNotificationOrderDao;

	@Autowired
	private IOffsystemPendingOrderDao offsystemPendingOrderDao;

	@Autowired
	private IMerchantConfigurationDao merchantConfigurationDao;

	/**
	 * Returns a list of MerchantConfuration (payment methods) configured for a given
	 * merchantId
	 * @param merchantId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Collection<MerchantConfiguration> getConfiguredPaymentMethods(int merchantId) throws Exception {
		
		return merchantConfigurationDao.findListByKey(PaymentConstants.MODULE_PAYMENT_INDICATOR_NAME, merchantId);
		
	}
	
	/**
	 * CC, MONEYORDER, CHEQUE
	 * @param paymentModuleName
	 * @param order
	 * @throws TransactionException
	 */
	@Transactional
	public void recordOffSystemPayment(String paymentModuleName, Order order) throws Exception {

		throw new Exception("not implemented");


	}

	/**
	 * Add a pending payment for a given order
	 * this is used for Paypal IPN
	 * @param pending
	 * @param order
	 * @throws Exception
	 */
	@Transactional
	public void saveOrUpdateOffsystemPendingOrder(OffsystemPendingOrder pending, Order order) throws Exception {
		pending.setOffsystemPendingOrderId(order.getOrderId());
		pending.setDateAdded(new Date());
		pending.setMerchantId(order.getMerchantId());
		pending.setOffsystemModule(order.getPaymentMethod());

		offsystemPendingOrderDao.saveOrUpdate(pending);
	}




	@Transactional
	public void saveMerchantPaymentGatewayTrx(MerchantPaymentGatewayTrx trx) throws Exception {
		MerchantPaymentGatewayTrxDao.persist(trx);
	}

	/**
	 * This is the main payment processing method
	 * @param order
	 * @param paymentMethod
	 * @throws Exception
	 */
	@Transactional
	public SalesManagerTransactionVO processPaymentTransaction(MerchantStore store, Order order, Customer customer, PaymentMethod paymentMethod) throws Exception {


		ReferenceService refservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		
		String pm = order.getPaymentModuleCode();
		if(StringUtils.isBlank(pm)) {
			pm  = paymentMethod.getPaymentModuleName();
			if(StringUtils.isBlank(pm)) {
				
				//check payment module configured if type is credit card
				if(paymentMethod.getType()==PaymentConstants.PAYMENT_TYPE_CREDIT_CARD_GATEWAY) {
					//determine if a gateway is configured
					
					ConfigurationRequest configrequest = new ConfigurationRequest(order
							.getMerchantId(), PaymentConstants.MODULE_PAYMENT_INDICATOR_NAME);
					MerchantService service = (MerchantService) ServiceFactory
							.getService(ServiceFactory.MerchantService);
					ConfigurationResponse vo = service.getConfiguration(configrequest);
					
					if(vo!=null) {
						List configs = vo.getMerchantConfigurationList();
						if(configs!=null && configs.size()>0) {
							for(Object o:configs) {
								MerchantConfiguration config = (MerchantConfiguration)o;
								if(!StringUtils.isBlank(config.getConfigurationValue()) && !StringUtils.isBlank(config.getConfigurationValue1())) {
									
									if(PaymentUtil.isPaymentModuleCreditCardType(config.getConfigurationValue1())) {
										pm = config.getConfigurationValue1();
										break;
									}
									
								}
							}
						}
					}
					
				}
			}
		}
		
		if(StringUtils.isBlank(pm)) {
			throw new Exception("Payment module is not defined for order id " + order.getOrderId());
		}
		
		CoreModuleService cms = refservice.getCoreModuleService(store.getCountry(), pm);


		//String moduleName = paymentMethod.getPaymentModuleName();

		PaymentModule paymentModule = (PaymentModule)SpringUtil.getBean(pm);
		paymentMethod.setPaymentModuleName(pm);
		order.setPaymentModuleCode(pm);

		if(paymentModule==null) {
			throw new Exception("Payment module " + paymentModule + " is not implemented in the module list");
		}

		SalesManagerTransactionVO vo = paymentModule.processTransaction(cms, paymentMethod, order, customer);
		
		order.setChannel(OrderConstants.ONLINE_CHANNEL);
		
		return vo;

	}

	/**
	 * This method will invoke initTransaction on the payment module
	 * This is used for PayPal ExpressCheckout.
	 * Returns a Map conpaining the data returned by the transaction system
	 * @throws Exception
	 */
	@Transactional
	public Map<String,String> preInitializePayment(MerchantStore store, Order order) throws Exception {

		//get CoreModuleService
		ReferenceService refservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		CoreModuleService cms = refservice.getCoreModuleService(store.getCountry(), order.getPaymentModuleCode());

		PaymentModule module = (PaymentModule)SpringUtil.getBean(order.getPaymentModuleCode());
		return module.initTransaction(cms, order);

	}


	public List<com.salesmanager.core.service.payment.SalesManagerTransactionVO> getTransactions(Order order) throws TransactionException {
		try {
			TransactionImpl impl = new TransactionImpl();
			return impl.getTransactions(order);
		} catch(Exception e) {
			if(e instanceof TransactionException) throw (TransactionException)e;
			throw new TransactionException(e);
		}
	}

	/**
	 * Refunds an order processed as capture or sale
	 * @param origincountryid
	 * @param order
	 * @return
	 * @throws PaymentException
	 */
	public SalesManagerTransactionVO refundTransaction(MerchantStore store,Order order, BigDecimal amount) throws TransactionException {
		
		try {
			
		String moduleName = order.getPaymentModuleCode();
		
		String key = PaymentConstants.MODULE_PAYMENT_GATEWAY + moduleName;
		
		if(moduleName.equals(PaymentConstants.PAYMENT_PAYPALNAME)) {
			key = PaymentConstants.MODULE_PAYMENT + moduleName;
		}
		
		ConfigurationRequest configRequest = new ConfigurationRequest(order
				.getMerchantId(), key);//will get the line containing which gateway is configured

		MerchantService service = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);
		ConfigurationResponse resp = service.getConfiguration(configRequest);
		if(resp==null) {
			TransactionException te = new TransactionException(" This gateway [" + moduleName + "] is not configured configured for merchantId " + store.getMerchantId());
			te.setErrorcode("03");
			
			throw te;
			
			
		}

		MerchantConfiguration conf = (MerchantConfiguration)resp.getConfiguration(moduleName);
		
		if(conf==null || StringUtils.isBlank(conf.getConfigurationValue())) {
			
			TransactionException te = new TransactionException(" This gateway [" + moduleName + "] is not configured configured for merchantId " + store.getMerchantId());
			te.setErrorcode("03");
			
			throw te;
			
		}
		
		
		    CreditCardPaymentModule module = (CreditCardPaymentModule)SpringUtil.getBean(order.getPaymentModuleCode());
			if(module!=null) {
				
					CustomerService cservice = (CustomerService)ServiceFactory.getService(ServiceFactory.CustomerService);
					Customer customer = cservice.getCustomer(order.getCustomerId());
					
					
					GatewayTransactionVO vo = module.processRefund(order, store, customer, amount, moduleName);
					return vo;
	
			} else {
				throw new TransactionException("Module " + order.getPaymentModuleCode() + " not defined in sm-core-config.properties");
			}
		
		
		} catch(Exception e) {
			if(e instanceof TransactionException) throw (TransactionException)e;
			throw new TransactionException("Error while refunding transaction for " + order.getPaymentModuleCode() + " and orderid " + order.getOrderId(),e);
		}
	}
	
	@Transactional
	public Collection<MerchantPaymentGatewayTrx> findMerchantPaymentGatewayTrxByMerchantIdAndOrderId( int merchantId, long orderId) {

		return MerchantPaymentGatewayTrxDao.findByMerchantIdAndOrderId(merchantId, orderId);
	}

	/**
	 * Process an order processed as pre-autorize
	 * FOR CREDIT CARDS ONLY
	 * @param origincountryid
	 * @param order
	 * @return
	 * @throws Exception
	 */
	public GatewayTransactionVO captureTransaction(MerchantStore store,Order order) throws TransactionException {
		
		try {
		
			CreditCardPaymentModule module = (CreditCardPaymentModule) SpringUtil.getBean(order
					.getPaymentModuleCode());
			
			/**
			 * get the order module name
			 * check if that gateway is still configured
			 */
			
			String moduleName = order.getPaymentModuleCode();
			
			//check if configured
			
			String key = PaymentConstants.MODULE_PAYMENT_GATEWAY + moduleName;
			
			if(moduleName.equals(PaymentConstants.PAYMENT_PAYPALNAME)) {
				key = PaymentConstants.MODULE_PAYMENT + moduleName;
			}
			
			ConfigurationRequest configRequest = new ConfigurationRequest(order
					.getMerchantId(), key);//will get the line containing which gateway is configured

			MerchantService service = (MerchantService) ServiceFactory
			.getService(ServiceFactory.MerchantService);
			ConfigurationResponse resp = service.getConfiguration(configRequest);
			if(resp==null) {
				
				TransactionException te = new TransactionException(" No gateway configured for merchantId " + store.getMerchantId());
				te.setErrorcode("03");

				
				throw te;
				
				//throw new TransactionException(" No gateway configured for merchantId " + store.getMerchantId());	
			}
			
			if(resp.getMerchantConfigurationList()==null || resp.getMerchantConfigurationList().size()==0) {
				
				TransactionException te = new TransactionException(" This gateway [" + moduleName + "] is not configured configured for merchantId " + store.getMerchantId());
				te.setErrorcode("03");
				
				throw te;
				
				
			}
			
			MerchantConfiguration conf = (MerchantConfiguration)resp.getConfiguration(moduleName);
			
			if(conf==null || StringUtils.isBlank(conf.getConfigurationValue())) {
				
				TransactionException te = new TransactionException(" This gateway [" + moduleName + "] is not configured configured for merchantId " + store.getMerchantId());
				te.setErrorType(02);
				
				throw te;
				
				
			}
			
			CustomerService cservice = (CustomerService)ServiceFactory.getService(ServiceFactory.CustomerService);
			Customer customer = cservice.getCustomer(order.getCustomerId());
	
			if (module != null) {
				try {
					
					GatewayTransactionVO vo = module.processCapture(order,store,customer,moduleName);
	
					return vo;
				} catch (Exception e) {
					if (e instanceof TransactionException)
						throw (TransactionException) e;
					throw new TransactionException("Error while refunding transaction for "
							+ order.getPaymentModuleCode() + " and orderid "
							+ order.getOrderId(), e);
				}
			} else {
				throw new TransactionException("Module " + order.getPaymentModuleCode()
						+ " not defined in sm-core-config.properties");
			}
		
		} catch(Exception e) {
			if(e instanceof TransactionException) throw (TransactionException)e;
			throw new TransactionException(e);
		}
		
		

	}

	/**
	 * Finds for a given order one and only one refundable transaction, meaning
	 * a transaction that can be a CAPTURE or a SALE
	 * @param order
	 * @return
	 * @throws PaymentException
	 */
	public GatewayTransactionVO getRefundableTransaction(Order order) throws TransactionException {
		try {

			TransactionImpl impl = new TransactionImpl();
			int types[] = {PaymentConstants.CAPTURE,PaymentConstants.SALE};
			return impl.getTransactionType(order,types);
		} catch(Exception e) {
			if(e instanceof TransactionException) throw (TransactionException)e;
			throw new TransactionException(e);
		}
	}
	/**
	 * Finds for a given order one and only one capturable transaction, meaning
	 * a transaction that is set to CAPTURE
	 * @param order
	 * @return
	 * @throws PaymentException
	 */
	public GatewayTransactionVO getCapturableTransaction(Order order) throws TransactionException {
		try {

			TransactionImpl impl = new TransactionImpl();
			int types[] = {PaymentConstants.PREAUTH};
			return impl.getTransactionType(order,types);
		} catch(Exception e) {
			if(e instanceof TransactionException) throw (TransactionException)e;
			throw new TransactionException(e);
		}
	}

	/**
	 * Returns a list of CentralIntegrationService for payment method
	 * @param countryid
	 * @return
	 */
	public List<CoreModuleService> getPaymentMethodsList(String countryIsoCode) {



		return ServicesUtil.getPaymentMethodsList(countryIsoCode);

	}

	public List<CoreModuleService> getPaymentMethods() {
		return ServicesUtil.getPaymentMethods();
	}

}



```
