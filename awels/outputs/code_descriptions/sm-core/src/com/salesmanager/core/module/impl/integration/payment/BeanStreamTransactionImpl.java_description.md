# BeanStreamTransactionImpl.java

## Review

## 1. Summary  

**Purpose**  
`BeanStreamTransactionImpl` is a Spring‑managed component that implements the **BeanStream** credit‑card gateway.  
It translates order‑level data (`Order`, `Customer`, `MerchantStore`) into the URL‑encoded request format expected by BeanStream, sends the request via `HttpURLConnection`, parses the response, persists the raw sent/received payloads, and returns a `GatewayTransactionVO` that can be used by the rest of the sales‑flow.  

**Key collaborators**  

| Class / Interface | Responsibility |
|-------------------|----------------|
| `GatewayTransactionVO` | Holds the result of a gateway call. |
| `IntegrationKeys` / `IntegrationProperties` | Store the merchant credentials and configuration flags. |
| `PaymentService` | Persist and retrieve `MerchantPaymentGatewayTrx` objects (DAO). |
| `CoreModuleService` | Provides service‑specific URLs, environment handling, etc. |
| `MerchantConfiguration` | Store encrypted credentials and properties in the `MERCHANT_CONFIGURATION` table. |
| Utility classes (`CurrencyUtil`, `StringUtil`, `EncryptionUtil`, `CreditCardUtil`, `DateUtil`) | Formatting, encryption, and masking. |

The class is a concrete implementation of `CreditCardGatewayTransactionImpl`, itself a subclass of an abstract gateway transaction interface. It is wired into the framework by `@Component`, so Spring can inject it where needed.  

---

## 2. Detailed Description  

### 2.1 Transaction life‑cycle  

| Method | What it does | How it works |
|--------|--------------|--------------|
| `authorizeAndCapture(Order, ...)` | Performs a single‑step purchase (`P`). | Delegates to `makeTransaction("P")`. |
| `authorizeTransaction(Order, ...)` | Authorises a payment (`PA`). | Delegates to `makeTransaction("PA")`. |
| `captureTransaction(...)` | Completes a pre‑auth (`PAC`). | Builds a request string with the `trnId` of the original transaction and sends it. |
| `refundTransaction(...)` | Issues a refund (`R`). | Builds a request string that includes `adjId` (original `trnId`) and `trnOrderNumber`. |
| `sendTransaction(...)` | Low‑level HTTP helper: opens the connection, POSTs the NVP string, reads the response, parses it into a map, checks the approval flag, and finally delegates to `parseResponse(...)`. |
| `parseResponse(...)` | Persists a `MerchantPaymentGatewayTrx` via `PaymentService`, encrypts the original request, and builds a `GatewayTransactionVO`. |
| `retreiveTransactions(...)` | Pulls all gateway transaction records for a given order from the database using `TransactionHelper`. |
| `getConfiguration(...)` | Reads a merchant’s encrypted configuration, decrypts it, and populates a `ConfigurationResponse`. |
| `storeConfiguration(...)` | Accepts the values from the UI, encrypts the credentials, builds the configuration string, and persists it via `MerchantService`. |

### 2.2 Request construction  

`makeTransaction` concatenates all key/value pairs into a single URL‑encoded string (`transaction`). The string is written directly to the connection’s output stream. The implementation logs a *masked* version of the request for debugging.  

The request format follows BeanStream’s “name=value” style, but several of the keys are mistyped (`meerchant_id` instead of `merchant_id`, `trnID` vs `trnId`, etc.).  This will cause failed calls unless BeanStream is tolerant of the typo.

### 2.3 Response handling  

`sendTransaction`:

1. Sets `doInput`/`doOutput`, `Content-Type`, `User‑Agent`, and `Content‑Length`.
2. Writes the request string, reads the response stream, and concatenates all lines.
3. Uses `StringUtil.deformatUrlResponse` to turn the raw “name=value&…” string into a `Map`.
4. Validates the mandatory field `TRNAPPROVED`.  
5. If the transaction was rejected (`TRNAPPROVED=0`) it throws a `TransactionException` with code `02`.  
6. On success, delegates to `parseResponse` for persistence.

`parseResponse` creates a `MerchantPaymentGatewayTrx`, encrypts the raw request, sets all the fields expected by the DB schema, and calls `PaymentService.saveMerchantPaymentGatewayTrx`. It then creates a `GatewayTransactionVO` that contains the amount, card type, expiry, and the raw gateway ID from the response.

### 2.4 Configuration

`getConfiguration` decrypts the stored configuration value, extracts the integration keys, properties, and returns them in the `ConfigurationResponse`.  
`storeConfiguration` does the inverse: it reads values from the request, encrypts them with the merchant‑specific key, creates/updates a `MerchantConfiguration` entity, and persists it with `MerchantService`.

---

## 3. Functions / Methods  

| Method | Purpose | Comments |
|--------|---------|----------|
| `authorizeAndCapture(Order, ...)` | Authorises and captures a payment in a single step. | Delegates to `makeTransaction("P")`. |
| `authorizeTransaction(Order, ...)` | Authorises a payment but leaves it pending for later capture. | Delegates to `makeTransaction("PA")`. |
| `captureTransaction(...)` | Captures a pre‑authorised transaction. | Builds its own request string (type `PAC`). |
| `refundTransaction(...)` | Refunds a completed transaction. | Builds request string (type `R`). |
| `retreiveTransactions(int, Order)` | Returns all past transactions for an order. | Uses `TransactionHelper`. |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | Populates configuration metadata for the UI. | Decrypts stored values. |
| `storeConfiguration(int, ConfigurationResponse, HttpServletRequest)` | Persists updated credentials & properties. | Expects attributes on `HttpServletRequest`. |
| `sendTransaction(String, String, Order, HttpURLConnection)` | Low‑level HTTP POST and response parsing. | Handles I/O and throws `TransactionException`. |
| `parseResponse(String, String, String, Map, Order)` | Persists `MerchantPaymentGatewayTrx` and builds VO. | Calls `EncryptionUtil` to encrypt request. |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | Overloaded to fetch config? (duplicate signature) | Might be a mistake – appears twice. |
| `storeConfiguration(int, ConfigurationResponse, HttpServletRequest)` | Overloaded to store config? (duplicate signature) | Appears twice – likely copy‑paste error. |

> **Note**: Several methods contain `TODO` comments and lack actual implementation (e.g. `storeConfiguration` originally had incomplete code). Also, only some overridden methods carry `@Override`, which could hide interface mismatches.

---

## 4. Dependencies  

| Package / Class | Usage |
|-----------------|-------|
| `org.springframework.stereotype.Component` | Marks the class for DI. |
| `ServiceFactory` | Retrieves various services (`PaymentService`, `MerchantService`). |
| `HttpURLConnection`, `URL`, `DataOutputStream`, `DataInputStream`, `BufferedReader` | Native Java networking. |
| `org.apache.commons.lang3.StringUtils` | Null/blank checks. |
| `EncryptionUtil` | Encrypt/decrypt request/response payloads and credentials. |
| `StringUtil`, `CreditCardUtil`, `CurrencyUtil`, `DateUtil` | Helper utilities for formatting, masking, and currency conversion. |
| `PaymentService`, `MerchantService` | Persist `MerchantPaymentGatewayTrx` and `MerchantConfiguration`. |
| `TransactionHelper` | Retrieve existing gateway transactions. |
| `ConfigurationResponse`, `IntegrationKeys`, `IntegrationProperties` | Pass configuration data. |
| `PaymentConstants` | Numeric constants for transaction types. |

---

## 5. Additional Notes & Recommendations  

### 5.1 Code‑Quality Issues  

1. **Hard‑coded request keys**  
   * The request string uses `"meerchant_id"` instead of `"merchant_id"` in many places. This typo will cause BeanStream to reject the request.  
   * `request` strings are built manually; using a builder or a library that handles URL‑encoding would reduce bugs.

2. **Inconsistent logging**  
   * The debug log prints the raw, unmasked card number (`request`), whereas the masked string is only used in the debug‑only block. This leaks sensitive data in production logs.  
   * Masking should be applied *before* logging or omitted entirely.

3. **Exception handling**  
   * `sendTransaction` checks `conn.getResponseCode() != -1` but ignores non‑200 codes. It should treat any non‑2xx code as a failure.  
   * The map key lookup uses uppercase (`"TRNAPPROVED"`) but `StringUtil.deformatUrlResponse` may return lowercase keys, leading to `null` values.  
   * Many methods (`storeConfiguration`, `getConfiguration`) swallow generic `Exception` and log minimal information.

4. **Resource management**  
   * Streams are closed in a finally block, but `conn.disconnect()` is called only if `conn != null`.  
   * Using *try‑with‑resources* (`try (DataOutputStream out = …)`) would simplify cleanup and avoid manual close logic.

5. **Magic numbers & strings**  
   * Transaction types (`P`, `PA`, `PAC`, `R`) are hard‑coded. A dedicated enum or constants would improve readability.  
   * Property indexes (`properties1`, `properties2`, etc.) are string‑based attributes in the request; this is brittle.

6. **Duplicate methods**  
   * `getConfiguration` and `storeConfiguration` appear twice (two overloaded signatures) – likely a copy‑paste error. Only one pair should exist.

7. **Missing null checks**  
   * The code assumes all objects (`order`, `customer`, `keys`, `props`) are non‑null. A `NullPointerException` could surface if any is missing.

8. **Typo in `retreiveTransactions`**  
   * The method name should be `retrieveTransactions`. Also the method does not filter by transaction type or status.

9. **Hard‑coded agent string**  
   * `"Mozilla/4.0"` is a legacy user‑agent. Modern APIs may require a different value or may ignore it.

10. **Encryption key handling**  
    * The key is generated from the merchant ID. If the merchant ID changes (unlikely) or the key generation algorithm changes, decryption will fail.  

### 5.2 Design & Architecture  

* **Single Responsibility Violation** – The class handles request construction, HTTP communication, response parsing, persistence, and configuration management all in one.  
* **Mix of Concerns** – Business logic (transaction flow) is intertwined with infrastructure concerns (HTTP, persistence). Splitting into a *gateway client*, a *transaction service*, and a *configuration service* would improve testability and maintainability.  
* **Error Handling** – Domain‑specific errors (`TransactionException`) are thrown but often with hard‑coded error codes (`02`, `01`). Using a richer error hierarchy or an enum of error codes would help downstream handling.  
* **Configuration** – Credentials and flags are stored in `configurationValue`/`configurationValue2` as a semicolon‑separated string. A schema‑aware entity (e.g., dedicated fields for `merchantId`, `username`, `password`, `environment`, `useCvv`) would make the code clearer and safer.  

### 5.3 Suggested Improvements  

| Area | Suggested Change | Benefit |
|------|------------------|---------|
| **Request building** | Use a dedicated request‑builder (e.g., Apache `URIBuilder` or a custom `BeanStreamRequest` class) that handles URL‑encoding and masking. | Reduces typos, centralises key names. |
| **HTTP client** | Replace `HttpURLConnection` with Apache HttpClient or Spring’s `RestTemplate`/`WebClient`. | Better timeout handling, automatic resource cleanup, easier debugging. |
| **Logging** | Mask all PII before logging; store masked card number in logs only. | Avoid accidental data leaks. |
| **Error handling** | Use try‑with‑resources; propagate `IOException` as a domain error with a clear code. | Cleaner code, fewer resource leaks. |
| **Constants** | Externalise all request parameter names (`MERCHANT_ID`, `TRNTYPE`, etc.) in a `BeanStreamConstants` class. | Prevents typos and makes changes easier. |
| **Configuration** | Store credentials in separate fields in the `MERCHANT_CONFIGURATION` table; avoid a semicolon‑separated string. | Improves readability, reduces parsing errors. |
| **Unit tests** | Add unit tests for request construction, response parsing, and persistence. | Ensures future changes don’t break behaviour. |
| **Typo fixes** | Replace all `"meerchant_id"` with `"merchant_id"`; correct other misspellings. | Prevents silent API failures. |
| **Method signatures** | Ensure all overridden methods are annotated with `@Override` to catch signature mismatches. | Improves compiler safety. |
| **Naming** | Consistently use camelCase for variables; use `TransactionType` enum instead of string literals. | Readability. |

---

### 6. Final Verdict  

The class achieves the core goal of wiring BeanStream into the payment flow, but it suffers from several **maintenance‑risk** issues:

* Hard‑coded strings and typos that could silently break the integration.
* Mixed responsibilities that make the class hard to test and evolve.
* Inconsistent error handling and missing null checks.
* Potential data‑leakage through logs and improper masking.

Addressing these concerns—especially the typo in the request key, the masking of sensitive data, and refactoring into smaller, focused components—would greatly increase the reliability, security, and testability of the integration.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.module.impl.integration.payment;

import java.io.BufferedReader;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.InputStreamReader;
import java.io.Reader;
import java.io.StringReader;
import java.math.BigDecimal;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.ArrayList;
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;


import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.springframework.stereotype.Component;


import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.payment.MerchantPaymentGatewayTrx;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.payment.GatewayTransactionVO;
import com.salesmanager.core.service.payment.PaymentService;
import com.salesmanager.core.service.payment.SalesManagerTransactionVO;
import com.salesmanager.core.service.payment.TransactionException;
import com.salesmanager.core.service.payment.impl.TransactionHelper;
import com.salesmanager.core.util.CreditCardUtil;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.EncryptionUtil;
import com.salesmanager.core.util.LogMerchantUtil;
import com.salesmanager.core.util.MerchantConfigurationUtil;
import com.salesmanager.core.util.StringUtil;

/**
 * Manages credit card transactions with BeanStream API
 * @author Carl Samson
 *
 */
@Component("beanstream")
public class BeanStreamTransactionImpl extends CreditCardGatewayTransactionImpl {

	
	private static Logger log = Logger.getLogger(BeanStreamTransactionImpl.class);
	
	@Override
	public GatewayTransactionVO authorizeAndCapture(IntegrationKeys keys,
			IntegrationProperties properties, MerchantStore store, Order order, Customer customer, CoreModuleService cms)
			throws TransactionException {
		// TODO Auto-generated method stub
		return makeTransaction("P", keys, properties, store, order, customer, cms);
	}

	@Override
	public GatewayTransactionVO authorizeTransaction(IntegrationKeys keys,
			IntegrationProperties properties, MerchantStore store, Order order, Customer customer, CoreModuleService cms)
			throws TransactionException {
		// TODO Auto-generated method stub
		return makeTransaction("PA", keys, properties, store, order, customer, cms);
	}

	/**
	 * Invoked from admin panel to capture after an authorization 
	 */
	public GatewayTransactionVO captureTransaction(IntegrationKeys ik, IntegrationProperties props,
			MerchantStore store, Order order, GatewayTransactionVO trx, Customer customer, CoreModuleService cis) throws TransactionException {
		// TODO Auto-generated method stub
		

		String host = cis.getCoreModuleServiceProdDomain();
		String protocol = cis.getCoreModuleServiceProdProtocol();
		String port = cis.getCoreModuleServiceProdPort();
		String url = cis.getCoreModuleServiceProdEnv();
		if (props.getProperties2().equals(
				String.valueOf(PaymentConstants.TEST_ENVIRONMENT))) {
			host = cis.getCoreModuleServiceDevDomain();
			protocol = cis.getCoreModuleServiceDevProtocol();
			port = cis.getCoreModuleServiceDevPort();
			url = cis.getCoreModuleServiceDevEnv();
		}
		
		StringBuffer server = new StringBuffer();
		if(!StringUtils.isBlank(protocol)) {
			server.append(protocol);
			server.append("://");
		}
		if(!StringUtils.isBlank(host)) {
			server.append(host);
		}
		if(!StringUtils.isBlank(port)) {
			server.append(":");
			server.append(port);
		}
		if(!StringUtils.isBlank(url)) {
			server.append(url);
		}

		String trnID = trx.getTransactionDetails().getMerchantPaymentGwTrxid();
		
		String amount = CurrencyUtil.displayFormatedAmountNoCurrency(order.getTotal(), order.getCurrency());
		
		/**
		merchant_id=123456789&requestType=BACKEND
		&trnType=PAC&username=user1234&password=pass1234&trnID=1000
		2115 --> requires also adjId [not documented]
		**/
		
		StringBuffer messageString = new StringBuffer();
		messageString.append("requestType=BACKEND&");
		messageString.append("meerchant_id=").append(ik.getTransactionKey()).append("&");
		messageString.append("trnType=").append("PAC").append("&");
		messageString.append("username=").append(ik.getUserid()).append("&");
		messageString.append("password=").append(ik.getPassword()).append("&");
		messageString.append("trnAmount=").append(amount).append("&");
		messageString.append("adjId=").append(trnID).append("&");
		messageString.append("trnID=").append(trnID);
		
		log.debug("REQUEST SENT TO BEANSTREAM -> " + messageString.toString());

		
		HttpURLConnection conn = null;

		try {
			
			URL postURL = new URL(server.toString());
			conn = (HttpURLConnection) postURL.openConnection();
			

			GatewayTransactionVO response = this.sendTransaction(messageString.toString(), "PAC", order, conn);
			
			return response;
			
		} catch(Exception e) {
			
			if(e instanceof TransactionException)
				throw (TransactionException)e;
			throw new TransactionException("Error while processing BeanStream transaction",e);

		} finally {
			
			
			if (conn != null) {
				try {
					conn.disconnect();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}
			
			
		}
		
	}

	
	/**
	 * no need to initialize a transaction for this gateway
	 */
	public Map<String, String> initTransaction(
			CoreModuleService serviceDefinition, Order order)
			throws TransactionException {
		// TODO Auto-generated method stub
		return null;
	}

	/**
	 * no need to invoke any 'post transaction' url once completed
	 */
	public Order postTransaction(Order order) throws TransactionException {
		// TODO Auto-generated method stub
		return null;
	}

	/**
	 * Invoked from admin panel to refund after a capture 
	 */
	public 	GatewayTransactionVO refundTransaction(IntegrationKeys keys, IntegrationProperties props,
			MerchantStore store, Order order, GatewayTransactionVO trx, Customer customer, CoreModuleService cis, BigDecimal amount) throws TransactionException {
		// TODO Auto-generated method stub
		
		String host = cis.getCoreModuleServiceProdDomain();
		String protocol = cis.getCoreModuleServiceProdProtocol();
		String port = cis.getCoreModuleServiceProdPort();
		String url = cis.getCoreModuleServiceProdEnv();
		if (props.getProperties2().equals(
				String.valueOf(PaymentConstants.TEST_ENVIRONMENT))) {
			host = cis.getCoreModuleServiceDevDomain();
			protocol = cis.getCoreModuleServiceDevProtocol();
			port = cis.getCoreModuleServiceDevPort();
			url = cis.getCoreModuleServiceDevEnv();
		}
		
		StringBuffer server = new StringBuffer();
		if(!StringUtils.isBlank(protocol)) {
			server.append(protocol);
			server.append("://");
		}
		if(!StringUtils.isBlank(host)) {
			server.append(host);
		}
		if(!StringUtils.isBlank(port)) {
			server.append(":");
			server.append(port);
		}
		if(!StringUtils.isBlank(url)) {
			server.append(url);
		}
		
		
		String orderTansactionNumber = trx.getInternalGatewayOrderId();
		
		/**
			merchant_id=123456789&requestType=BACKEND
			&trnType=R&username=user1234&password=pass1234
			&trnOrderNumber=1234&trnAmount=1.00&adjId=1000
			2115
		**/
		
		String amnt = CurrencyUtil.displayFormatedAmountNoCurrency(amount, order.getCurrency());
		String trn = trx.getTransactionDetails().getMerchantPaymentGwTrxid();
		
		StringBuffer messageString = new StringBuffer();
		messageString.append("requestType=BACKEND&");
		messageString.append("meerchant_id=").append(keys.getTransactionKey()).append("&");
		messageString.append("trnType=").append("R").append("&");
		messageString.append("username=").append(keys.getUserid()).append("&");
		messageString.append("password=").append(keys.getPassword()).append("&");
		messageString.append("trnOrderNumber=").append(orderTansactionNumber).append("&");
		messageString.append("trnAmount=").append(amnt).append("&");
		messageString.append("adjId=").append(trn);
		
		
		log.debug("REQUEST SENT TO BEANSTREAM -> " + messageString.toString());

		
		HttpURLConnection conn = null;

		try {
			
			URL postURL = new URL(server.toString());
			conn = (HttpURLConnection) postURL.openConnection();
			

			GatewayTransactionVO response = this.sendTransaction(messageString.toString(), "R", order, conn);
			
			return response;
			
		} catch(Exception e) {
			if(e instanceof TransactionException)
				throw (TransactionException)e;
			
			throw new TransactionException("Error while processing BeanStream transaction",e);

		} finally {
			
			
			if (conn != null) {
				try {
					conn.disconnect();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}
			
			
		}
		
	}

	/**
	 * Retrieve transaction history
	 */
	public List<SalesManagerTransactionVO> retreiveTransactions(int merchantid,
			Order order) throws Exception {
		// TODO Auto-generated method stub
		
		
		TransactionHelper trxhelper = new TransactionHelper();
		List trxs = trxhelper.getSentData(merchantid, order.getOrderId());

		if (trxs == null) {
			return null;
		}

		Iterator i = trxs.iterator();

		List returnlist = new ArrayList();

		while (i.hasNext()) {

			MerchantPaymentGatewayTrx trx = (MerchantPaymentGatewayTrx) i
					.next();

			GatewayTransactionVO mtrx = new GatewayTransactionVO();


			mtrx.setOrderID(String.valueOf(order.getOrderId()));
			
			mtrx.setInternalGatewayOrderId(trx.getMerchantPaymentGwOrderid());
			mtrx.setTransactionID(trx.getMerchantPaymentGwTrxid());
			
			mtrx.setTransactionDetails(trx);
			
			mtrx.setType(Integer.parseInt(trx.getMerchantPaymentGwAuthtype()));
			
			mtrx.setAmount(trx.getAmount());
			returnlist.add(mtrx);

		}
		return returnlist;
		
	}
	
	
	private GatewayTransactionVO makeTransaction(String type,
			IntegrationKeys ik, IntegrationProperties props,
			MerchantStore store, Order order, Customer customer, CoreModuleService cms) throws TransactionException {
		

		

		//determine environment
		// determine production - test environment
		String host = cms.getCoreModuleServiceProdDomain();
		String protocol = cms.getCoreModuleServiceProdProtocol();
		String port = cms.getCoreModuleServiceProdPort();
		String url = cms.getCoreModuleServiceProdEnv();
		if (props.getProperties2().equals(
				String.valueOf(PaymentConstants.TEST_ENVIRONMENT))) {
			host = cms.getCoreModuleServiceDevDomain();
			protocol = cms.getCoreModuleServiceDevProtocol();
			port = cms.getCoreModuleServiceDevPort();
			url = cms.getCoreModuleServiceDevEnv();
		}
		
		StringBuffer server = new StringBuffer();
		if(!StringUtils.isBlank(protocol)) {
			server.append(protocol);
			server.append("://");
		}
		if(!StringUtils.isBlank(host)) {
			server.append(host);
		}
		if(!StringUtils.isBlank(port)) {
			server.append(":");
			server.append(port);
		}
		if(!StringUtils.isBlank(url)) {
			server.append(url);
		}

		
		String orderNumber = order.getOrderId()+DateUtil.generateTimeStamp();
		
		StringBuffer messageString = new StringBuffer();
		
		messageString.append("requestType=BACKEND&");
		messageString.append("meerchant_id=").append(ik.getTransactionKey()).append("&");
		messageString.append("trnType=").append(type).append("&");
		messageString.append("orderNumber=").append(orderNumber).append("&");
		messageString.append("trnCardOwner=").append(order.getCcOwner()).append("&");
		messageString.append("trnCardNumber=").append(order.getCcNumber()).append("&");
		messageString.append("trnExpMonth=").append(order.getCcExpires().substring(0, 2)).append("&");
		messageString.append("trnExpYear=").append(order.getCcExpires().substring(2,
				order.getCcExpires().length())).append("&");
		if (props.getProperties3().equals("2")) {
			messageString.append("trnCardCvd=").append(order.getCcCvv()).append("&");
		}
		messageString.append("trnAmount=").append(CurrencyUtil.displayFormatedAmountNoCurrency(order.getTotal(), order.getCurrency())).append("&");
		messageString.append("ordName=").append(customer.getCustomerBillingFirstName() + " " + customer.getCustomerBillingLastName()).append("&");
		messageString.append("ordAddress1=").append(customer.getCustomerBillingStreetAddress()).append("&");
		messageString.append("ordCity=").append(customer.getCustomerBillingCity()).append("&");
		
		Map zones = RefCache.getAllZonesmap(1);
		String zone = "--";
		if(zones!=null) {
			Zone z = (Zone)zones.get(customer.getCustomerBillingZoneId());
			if(z!=null) {
				zone = z.getZoneCode();
			}
		}
		
		Map countries = RefCache.getAllcountriesmap(1);
		Country c = (Country)countries.get(customer.getCustomerBillingCountryId());
		
		if(c==null) {
			log.error("Country is null for c " + customer.getCustomerCountryId());
			throw new TransactionException("Invalid country id " + customer.getCustomerBillingCountryId());
		}
		
		messageString.append("ordProvince=").append(zone).append("&");
		messageString.append("ordPostalCode=").append(customer.getCustomerBillingPostalCode()).append("&");
		messageString.append("ordCountry=").append(c.getCountryIsoCode2()).append("&");
		messageString.append("ordPhoneNumber=").append(customer.getCustomerTelephone()).append("&");
		messageString.append("ordEmailAddress=").append(customer.getCustomerEmailAddress());
		
		
		
		
		/**
		 * 	purchase (P)
		 *  -----------
				REQUEST -> merchant_id=123456789&requestType=BACKEND&trnType=P&trnOrderNumber=1234TEST&trnAmount=5.00&trnCardOwner=Joe+Test&trnCardNumber=4030000010001234&trnExpMonth=10&trnExpYear=10&ordName=Joe+Test&ordAddress1=123+Test+Street&ordCity=Victoria&ordProvince=BC&ordCountry=CA&ordPostalCode=V8T2E7&ordPhoneNumber=5555555555&ordEmailAddress=joe%40testemail.com
				RESPONSE-> trnApproved=1&trnId=10003067&messageId=1&messageText=Approved&trnOrderNumber=E40089&authCode=TEST&errorType=N&errorFields=&responseType=T&trnAmount=10%2E00&trnDate=1%2F17%2F2008+11%3A36%3A34+AM&avsProcessed=0&avsId=0&avsResult=0&avsAddrMatch=0&avsPostalMatch=0&avsMessage=Address+Verification+not+performed+for+this+transaction%2E&rspCodeCav=0&rspCavResult=0&rspCodeCredit1=0&rspCodeCredit2=0&rspCodeCredit3=0&rspCodeCredit4=0&rspCodeAddr1=0&rspCodeAddr2=0&rspCodeAddr3=0&rspCodeAddr4=0&rspCodeDob=0&rspCustomerDec=&trnType=P&paymentMethod=CC&ref1=&ref2=&ref3=&ref4=&ref5=
		
			pre authorization (PA)
			----------------------

			Prior to processing a pre-authorization through the API, you must modify the transaction settings in your Beanstream merchant member area to allow for this transaction type.
			- Log in to the Beanstream online member area at www.beanstream.com/admin/sDefault.asp.
			- Navigate to administration - account admin - order settings in the left menu.
			Under the heading �Restrict Internet Transaction Processing Types,� select either of the last two options. The �Purchases or Pre-Authorization Only� option will allow you to process both types of transaction through your web interface. De-selecting the �Restrict Internet Transaction Processing Types� checkbox will allow you to process all types of transactions including returns, voids and pre-auth completions.
		
			capture (PAC) -> requires trnId
			-------------
		
			refund (R)
			-------------
				REQUEST -> merchant_id=123456789&requestType=BACKEND&trnType=R&username=user1234&password=pass1234&trnOrderNumber=1234&trnAmount=1.00&adjId=10002115
				RESPONSE-> trnApproved=1&trnId=10002118&messageId=1&messageText=Approved&trnOrderNumber=1234R&authCode=TEST&errorType=N&errorFields=&responseType=T&trnAmount=1%2E00&trnDate=8%2F17%2F2009+1%3A44%3A56+PM&avsProcessed=0&avsId=0&avsResult=0&avsAddrMatch=0&avsPostalMatch=0&avsMessage=Address+Verification+not+performed+for+this+transaction%2E&cardType=VI&trnType=R&paymentMethod=CC&ref1=&ref2=&ref3=&ref4=&ref5=
		

			//notes
			//On receipt of the transaction response, the merchant must display order amount, transaction ID number, bank authorization code (authCode), currency, date and �messageText� to the customer on a confirmation page.
		*/
		

		//String agent = "Mozilla/4.0";
		//String respText = "";
		//Map nvp = null;
		
		
		/** debug **/
		
		try {
			

		StringBuffer messageLogString = new StringBuffer();
		
		messageLogString.append("requestType=BACKEND&");
		messageLogString.append("meerchant_id=").append(ik.getTransactionKey()).append("&");
		messageLogString.append("trnType=").append(type).append("&");
		messageLogString.append("orderNumber=").append(orderNumber).append("&");
		messageLogString.append("trnCardOwner=").append(order.getCcOwner()).append("&");
		messageLogString.append("trnCardNumber=").append(CreditCardUtil.maskCardNumber(order.getCcNumber())).append("&");
		messageLogString.append("trnExpMonth=").append(order.getCcExpires().substring(0, 2)).append("&");
		messageLogString.append("trnExpYear=").append(order.getCcExpires().substring(2,
				order.getCcExpires().length())).append("&");
		if (props.getProperties3().equals("2")) {
			messageLogString.append("trnCardCvd=").append(order.getCcCvv()).append("&");
		}
		messageLogString.append("trnAmount=").append(CurrencyUtil.displayFormatedAmountNoCurrency(order.getTotal(), order.getCurrency())).append("&");
		messageLogString.append("ordName=").append(customer.getCustomerBillingFirstName() + " " + customer.getCustomerBillingLastName()).append("&");
		messageLogString.append("ordAddress1=").append(customer.getCustomerBillingStreetAddress()).append("&");
		messageLogString.append("ordCity=").append(customer.getCustomerBillingCity()).append("&");


		messageLogString.append("ordProvince=").append(zone).append("&");
		messageLogString.append("ordPostalCode=").append(customer.getCustomerBillingPostalCode()).append("&");
		messageLogString.append("ordCountry=").append(c.getCountryIsoCode2()).append("&");
		messageLogString.append("ordPhoneNumber=").append(customer.getCustomerTelephone()).append("&");
		messageLogString.append("ordEmailAddress=").append(customer.getCustomerEmailAddress());

		/** debug **/


		log.debug("REQUEST SENT TO BEANSTREAM -> " + messageLogString.toString());

		
		} catch (Exception e) {
			log.error("cannot log debug transaction");
		}
		
		HttpURLConnection conn = null;
		//DataOutputStream output = null;
		//DataInputStream in = null;
		//BufferedReader is = null;
		try {
			
			URL postURL = new URL(server.toString());
			conn = (HttpURLConnection) postURL.openConnection();
			
			GatewayTransactionVO response = this.sendTransaction(messageString.toString(), type, order, conn);
			
			return response;


			
		} catch(Exception e) {
			
			if(e instanceof TransactionException) {
				throw (TransactionException)e;
			}
			
			throw new TransactionException("Error while processing BeanStream transaction",e);

		} finally {


			if (conn != null) {
				try {
					conn.disconnect();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}
		}

	}
	
	
	private GatewayTransactionVO sendTransaction(String transaction, String type, Order order, HttpURLConnection conn) throws TransactionException {
		
		String agent = "Mozilla/4.0";
		String respText = "";
		Map nvp = null;
		DataOutputStream output = null;
		DataInputStream in = null;
		BufferedReader is = null;
		try {
			

			// Set connection parameters. We need to perform input and output,
			// so set both as true.
			conn.setDoInput(true);
			conn.setDoOutput(true);

			// Set the content type we are POSTing. We impersonate it as
			// encoded form data
			conn.setRequestProperty("Content-Type",
					"application/x-www-form-urlencoded");
			conn.setRequestProperty("User-Agent", agent);

			conn.setRequestProperty("Content-Length", String
					.valueOf(transaction.length()));
			conn.setRequestMethod("POST");

			// get the output stream to POST to.
			output = new DataOutputStream(conn.getOutputStream());
			output.writeBytes(transaction);
			output.flush();


			// Read input from the input stream.
			in = new DataInputStream(conn.getInputStream());
			int rc = conn.getResponseCode();
			if (rc != -1) {
				is = new BufferedReader(new InputStreamReader(conn
						.getInputStream()));
				String _line = null;
				while (((_line = is.readLine()) != null)) {
					respText = respText + _line;
				}
				
				log.debug("BeanStream response -> " + respText.trim());
				
				nvp = StringUtil.deformatUrlResponse(respText.trim());
			} else {
				throw new TransactionException("Invalid response from BeanStream, return code is " + rc);
			}
			
			//check
			//trnApproved=1&trnId=10003067&messageId=1&messageText=Approved&trnOrderNumber=E40089&authCode=TEST&errorType=N&errorFields=

			String transactionApproved = (String)nvp.get("TRNAPPROVED");
			String transactionId = (String)nvp.get("TRNID");
			String messageId = (String)nvp.get("MESSAGEID");
			String messageText = (String)nvp.get("MESSAGETEXT");
			String orderId = (String)nvp.get("TRNORDERNUMBER");
			String authCode = (String)nvp.get("AUTHCODE");
			String errorType = (String)nvp.get("ERRORTYPE");
			String errorFields = (String)nvp.get("ERRORFIELDS");
			
			
			if(StringUtils.isBlank(transactionApproved)) {
				throw new TransactionException("Required field transactionApproved missing from BeanStream response");
			}
			
			//errors
			if(transactionApproved.equals("0")) {
				LogMerchantUtil.log(order.getMerchantId(),
						"Can't process BeanStream message " + messageText + " return code id " + messageId);
				log.debug("Can't process BeanStream message " + messageText);
	
				TransactionException te = new TransactionException(
						"Can't process BeanStream message " + messageText);
				te.setErrorcode("02");
				te.setReason(messageText);
				throw te;
			}
			
			//create transaction object

			return parseResponse(type,transaction,respText,nvp,order);
			
			
		} catch(Exception e) {
			if(e instanceof TransactionException) {
				throw (TransactionException)e;
			}
			
			throw new TransactionException("Error while processing BeanStream transaction",e);

		} finally {
			if (is != null) {
				try {
					is.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (in != null) {
				try {
					in.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

			if (output != null) {
				try {
					output.close();
				} catch (Exception ignore) {
					// TODO: handle exception
				}
			}

		}

		
	}
	
	private GatewayTransactionVO parseResponse(String transactionType,
			String request, String response, Map nvp,
			Order order) throws Exception {
		
		MerchantPaymentGatewayTrx gtrx = null;
		
		try {


			PaymentService pservice = (PaymentService) ServiceFactory
					.getService(ServiceFactory.PaymentService);

			gtrx = new MerchantPaymentGatewayTrx();
			gtrx.setMerchantId(order.getMerchantId());
			gtrx.setCustomerid(order.getCustomerId());
			gtrx.setOrderId(order.getOrderId());
			gtrx.setAmount(order.getTotal());
			gtrx.setMerchantPaymentGwMethod(order.getPaymentModuleCode());
			gtrx.setMerchantPaymentGwRespcode((String)nvp.get("TRNAPPROVED"));//transactionApproved
			gtrx.setMerchantPaymentGwOrderid((String)nvp.get("TRNORDERNUMBER"));//trnOrderNumber [required for refund]
			gtrx.setMerchantPaymentGwTrxid((String)nvp.get("TRNID"));//transactionId
			if(transactionType.equals("PA")) {//pre-auth
				gtrx.setMerchantPaymentGwAuthtype(String.valueOf(PaymentConstants.PREAUTH));
			} else if(transactionType.equals("PAC")) {//capture
				gtrx.setMerchantPaymentGwAuthtype(String.valueOf(PaymentConstants.CAPTURE));
			} else if(transactionType.equals("P")) {//capture
				gtrx.setMerchantPaymentGwAuthtype(String.valueOf(PaymentConstants.CAPTURE));
			} else if(transactionType.equals("R")) {//refund
				gtrx.setMerchantPaymentGwAuthtype(String.valueOf(PaymentConstants.REFUND));
			}
			
			gtrx.setMerchantPaymentGwSession("");

			String cryptedvalue = EncryptionUtil.encrypt(EncryptionUtil
					.generatekey(String.valueOf(order.getMerchantId())),
					request);
			gtrx.setMerchantPaymentGwSent(cryptedvalue);
			gtrx.setMerchantPaymentGwReceived(response);
			gtrx.setDateAdded(new Date(new Date().getTime()));
			gtrx.setAmount(order.getTotal());

			pservice.saveMerchantPaymentGatewayTrx(gtrx);


		} catch (Exception e) {

			TransactionException te = new TransactionException(
					"Can't persist MerchantPaymentGatewayTrx for order id"
							+ order.getOrderId(), e);
			te.setErrorcode("01");
			throw te;
		}

		GatewayTransactionVO vo = new GatewayTransactionVO();
		vo.setAmount(order.getTotal());
		vo.setCreditcard(order.getCardType());
		vo.setCreditcardtransaction(true);
		vo.setExpirydate(order.getCcExpires());
		vo.setInternalGatewayOrderId((String)nvp.get("TRNORDERNUMBER"));
		vo.setTransactionDetails(gtrx);
		
		vo.setTransactionID((String)nvp.get("TRNORDERNUMBER"));
		vo.setTransactionMessage((String)nvp.get("MESSAGETEXT"));
		
		return vo;
		
	}
	

	public ConfigurationResponse getConfiguration(
			MerchantConfiguration configurations, ConfigurationResponse vo)
			throws Exception {
		//get payment gatemay configuration from MERCHANT_CONFIGURATION table
		
		//merchantId, userName, password
		
		try {
			// vo.addConfiguration("paymentmethod",
			// configurations.getConfigurationValue());

			String decryptedvalue = EncryptionUtil.decrypt(
					EncryptionUtil.generatekey(String.valueOf(configurations
							.getMerchantId())), configurations
							.getConfigurationValue());
			IntegrationKeys ik = MerchantConfigurationUtil.getIntegrationKeys(decryptedvalue,";");
			vo.addConfiguration("keys", ik);

			IntegrationProperties props = MerchantConfigurationUtil.getIntegrationProperties(configurations
					.getConfigurationValue2(),";");
			vo.addConfiguration("properties", props);

			vo.addConfiguration("beanstream",
					configurations);
			
			
		} catch (Exception e) {
			log.error("Can't understand MerchantConfiguration"
					+ configurations.getConfigurationId());
		}
		
		return vo;
	}

	public void storeConfiguration(int merchantid, ConfigurationResponse vo, HttpServletRequest request)
			throws Exception {
		//
		
		//key -> MD_PAY_GW_beanstream
		//module -> beanstream
		//configurationValue -> encrypt(merchantId;userName;password)
		//configurationValue2 -> transactionType(auth/capture);environnement(1=PROD/2=TEST);useCvv(1=NO/2=YES)

		String transactionType = (String)request.getAttribute("properties.properties1");
		String environment = (String)request.getAttribute("properties.properties2");
		String useCvv = (String)request.getAttribute("properties.properties3");
		
		
		String merchantId = (String)request.getAttribute("keys.transactionKey");
		String userName = (String)request.getAttribute("keys.userid");
		String password = (String)request.getAttribute("keys.password");
		
		
		
		//we assume here that everything has been validated in the action class
		//and that no object will be null !
		
		
		String key = EncryptionUtil.generatekey(String.valueOf(merchantid));
		// keep this order userid,password,transactionkey
		String credentials = new StringBuffer().append(userName)
				.append(";").append(password).append(";").append(
						merchantId).toString();

		String encrypted = EncryptionUtil.encrypt(key, credentials);

		String props = new StringBuffer().append(
				transactionType).append(";").append(
						environment).append(";").append(
								useCvv).toString();
		
		

		MerchantConfiguration conf = (MerchantConfiguration)vo.getConfiguration(PaymentConstants.MODULE_PAYMENT_GATEWAY + "beanstream");
		if(conf==null) {

				conf = new MerchantConfiguration();
				conf.setConfigurationKey(PaymentConstants.MODULE_PAYMENT_GATEWAY + "beanstream");
				conf.setConfigurationModule("beanstream");
				conf.setMerchantId(merchantid);

		}
		conf.setLastModified(new Date());
		conf.setConfigurationValue(encrypted);
		conf.setConfigurationValue2(props);
		
		
		MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
		mservice.saveOrUpdateMerchantConfiguration(conf);
		

		
		
	}

}



```
