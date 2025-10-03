# PsigateTransactionImpl.java

## Review

## 1. Summary

**Purpose & Functionality**  
`PsigateTransactionImpl` implements the integration with the Psigate credit‑card payment gateway.  
The class exposes the usual payment operations that a gateway adapter must provide:

| Operation | Method |
|-----------|--------|
| Authorize only | `authorizeTransaction` |
| Authorize + Capture | `authorizeAndCapture` |
| Capture after pre‑auth | `captureTransaction` |
| Refund | `refundTransaction` |
| Retrieve past transactions | `retreiveTransactions` (typo) |
| Configuration handling | `getConfiguration`, `storeConfiguration` |
| Utility helpers | `makeTransaction`, `parseResponse`, `stripCredentials`, `stripProperties` |

**Key Components**

- **`PostMethod` & `HttpClient`** – from *Apache Commons HttpClient* (v3.x) to issue HTTP POST requests to the Psigate endpoint.
- **`Digester`** – used to parse the XML response returned by Psigate.
- **`EncryptionUtil`** – encrypts the raw XML request that is persisted for audit purposes.
- **`MerchantPaymentGatewayTrx`** – domain object that represents a payment transaction record in the local database.
- **`PaymentService`** – service layer responsible for persisting `MerchantPaymentGatewayTrx`.

**Design Patterns & Frameworks**

- *Adapter* – the class implements the gateway contract defined by its superclass `CreditCardGatewayTransactionImpl`.
- *Factory* – `ServiceFactory` is used to obtain `PaymentService`.
- *Utility* – several helper classes (`CreditCardUtil`, `CurrencyUtil`, `EncryptionUtil`, etc.) provide stateless operations.
- *Apache Commons Digester* – a lightweight SAX‑based parser for quick XML extraction.

---

## 2. Detailed Description

### 2.1 Flow of Execution

1. **Initialization**  
   `initTransaction` and `postTransaction` are stubs (`return null`); they are not used in the current implementation.

2. **Authorization / Capture**  
   - `authorizeTransaction` and `authorizeAndCapture` delegate to `makeTransaction`, passing the card action code (`"1"` for pre‑auth, `"0"` for capture).
   - `makeTransaction` builds an XML payload containing:
     - Store ID, passphrase, payment type (`CC`), order total, card action, card number/expiry, IP address (optional).
   - The XML is logged in masked form (`xmllogbuffer`).
   - A `PostMethod` is created pointing to the host/protocol/port determined by the environment flag in `props`.
   - The request is executed; the raw XML response is captured.

3. **Response Parsing**  
   - `Digester` populates a `PsigateParsedElements` instance with fields such as `OrderID`, `Approved`, `ReturnCode`, etc.
   - If the response contains an error message, a `TransactionException` is thrown.
   - Otherwise, `parseResponse` creates a `MerchantPaymentGatewayTrx`, encrypts the request payload, persists the record, and returns a `GatewayTransactionVO` describing the transaction.

4. **Capture & Refund**  
   - `captureTransaction` and `refundTransaction` are similar to `makeTransaction`, but send a different card action (`"2"` for post‑auth capture, `"3"` for refund) and include the original order ID.  
   - Refunds additionally supply a `SubTotal` field calculated from the supplied amount.

5. **Transaction Retrieval**  
   - `retreiveTransactions` queries the database for all sent payloads (`getSentData`), parses each payload with `Digester`, and constructs a list of `SalesManagerTransactionVO` objects for display in the UI.

5. **Configuration Handling**  
   - `getConfiguration` decrypts stored configuration values, extracts credentials (`stripCredentials`) and properties (`stripProperties`), and packages them into a `ConfigurationResponse`.
   - `storeConfiguration` is intentionally left empty because the surrounding action (`Central PaymentpsigateAction`) handles the persistence.

### 2.2 Assumptions & Constraints

| Assumption | Effect |
|------------|--------|
| `props` contains at least three semicolon‑delimited tokens | If fewer tokens exist, `stripProperties` will silently return empty strings, leading to malformed requests. |
| The XML response always follows the exact schema (`Result/OrderID`, etc.) | Any deviation will trigger a generic parsing exception. |
| The gateway endpoint always returns HTTP 200 | Non‑200 responses raise a `TransactionException`, but time‑outs, redirects or 4xx/5xx from the server are not distinguished. |
| The same XML header (`<AddressValidationRequest>`) is irrelevant to the payment payload | This header is included in every request but never used; it can confuse the server or developers. |
| `EncryptionUtil.generatekey` returns a deterministic key based on merchant ID | If the merchant ID changes or is not numeric, encryption will fail. |
| The gateway contract expects a `String` key, `String` password, and `String` transactionKey | The `stripCredentials` logic does not decrypt or validate these values – it merely splits on `;`. |

---

## 3. Functions / Methods

| Method | Description |
|--------|-------------|
| **`initTransaction(CorePaymentTransaction)`** | Stub – currently returns `null`. Should initialise gateway session. |
| **`postTransaction()`** | Stub – currently returns `null`. Should finalise a transaction. |
| **`authorizeTransaction(...)`** | Calls `makeTransaction` with card action `"1"` (pre‑auth). Returns a `GatewayTransactionVO`. |
| **`authorizeAndCapture(...)`** | Calls `makeTransaction` with card action `"0"` (sale/capture). |
| **`makeTransaction(...)`** | Core routine that builds the XML request, sends it to Psigate, parses the XML response, and delegates to `parseResponse`. Contains duplicated environment‑switching logic. |
| **`captureTransaction(...)`** | Sends a card action `"2"` (post‑auth capture) with the original order ID. |
| **`refundTransaction(...)`** | Sends a card action `"3"` (refund) with the original order ID and a subtotal. |
| **`parseResponse(int, String, String, PsigateParsedElements, Order, BigDecimal)`** | Persists the transaction (`MerchantPaymentGatewayTrx`) after checking for errors, encrypts the request, and creates a `GatewayTransactionVO`. Throws `TransactionException` on any persistence error. |
| **`retreiveTransactions(int, Order)`** | (typo: *retreive*) Queries the local DB for all sent Psigate payloads, parses each response with `Digester`, and builds a list of `SalesManagerTransactionVO` objects. |
| **`getConfiguration(MerchantConfiguration, ConfigurationResponse)`** | Decrypts the stored configuration, splits credentials and properties, and attaches them to the `ConfigurationResponse`. |
| **`storeConfiguration(int, ConfigurationResponse, HttpServletRequest)`** | Empty – the surrounding action is responsible. |
| **`stripCredentials(String)`** | Splits a semicolon‑delimited string into an `IntegrationKeys` instance. Supports up to 6 tokens (`key1..key3`). |
| **`stripProperties(String)`** | Splits a semicolon‑delimited string into an `IntegrationProperties` instance. Supports up to 4 tokens. |
| **`PsigateParsedElements` (inner class)** | Simple POJO used as the Digester target for parsing the XML response. |

---

## 3.1 Notable Implementation Details

- **XML Construction** – Uses `StringBuffer` for concatenation; no XML validation or escaping of special characters.
- **Card Action Codes** – Hard‑coded string literals (`"0"`, `"1"`, `"2"`, `"3"`). A constant enum would make the intent clearer.
- **Environment Switching** – The same block of code is repeated in `makeTransaction`, `captureTransaction`, and `refundTransaction` to determine host/protocol/port. Extracting this into a helper method would reduce duplication.
- **Logging** – The request is masked (`xmllogbuffer`) before being written to the log, which is good practice. However the raw XML request is still logged in `xmllogbuffer` as plain text before encryption.
- **Error Handling** – If `response.getErrMsg()` is non‑empty, a `TransactionException` is thrown. Any parsing exception is wrapped in a generic `Exception`. No distinction is made between network errors, time‑outs, or malformed XML.
- **Persistence** – The raw request XML is encrypted using a merchant‑specific key and stored in `MerchantPaymentGatewayTrx`. The encryption key is generated by `EncryptionUtil.generatekey(merchantId)`. If the key generation algorithm changes, all existing records become unreadable.
- **Configuration** – Credentials and properties are stored encrypted and later stripped. The parsing logic assumes a strict ordering of tokens, but does not verify the presence of all required tokens.

---

## 3. Functions / Methods – In‑Depth

| Method | Signature | Responsibility | Notes |
|--------|-----------|----------------|-------|
| `authorizeTransaction(...)` | `public GatewayTransactionVO authorizeTransaction(...)` | Calls `makeTransaction` with pre‑auth card action (`"1"`). | Straightforward wrapper. |
| `authorizeAndCapture(...)` | `public GatewayTransactionVO authorizeAndCapture(...)` | Calls `makeTransaction` with sale card action (`"0"`). | Might be confusing: sale = capture. |
| `makeTransaction(...)` | `private GatewayTransactionVO makeTransaction(...)` | Builds XML, sends request, parses response, delegates to `parseResponse`. | Contains duplicated host determination logic. |
| `captureTransaction(...)` | `public GatewayTransactionVO captureTransaction(...)` | Sends post‑auth capture (`"2"`) using original order ID. | Reuses most of `makeTransaction` code. |
| `refundTransaction(...)` | `public GatewayTransactionVO refundTransaction(...)` | Sends refund (`"3"`) with original order ID and subtotal. | Similar duplication. |
| `parseResponse(int, String, String, PsigateParsedElements, Order, BigDecimal)` | `private GatewayTransactionVO parseResponse(...)` | Creates `MerchantPaymentGatewayTrx`, encrypts the raw request, persists via `PaymentService`, returns a `GatewayTransactionVO`. | Throws generic `Exception`; could be more specific. |
| `retreiveTransactions(int, Order)` | `public List<SalesManagerTransactionVO> retreiveTransactions(...)` | Retrieves all sent payloads for a merchant/order, parses each XML, builds a list of `GatewayTransactionVO`. | Typo in method name; returns `List<SalesManagerTransactionVO>` but actually builds `List<GatewayTransactionVO>`? (In code it returns `List<SalesManagerTransactionVO>` but the list is populated with `GatewayTransactionVO`. This is a type mismatch that will compile with a warning but is logically wrong.) |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | `public ConfigurationResponse getConfiguration(...)` | Decrypts stored config, strips credentials/properties, adds them to the response. | Relies on `stripCredentials` and `stripProperties`. |
| `stripCredentials(String)` | `private IntegrationKeys stripCredentials(...)` | Parses a `;`‑delimited string into `IntegrationKeys`. | No validation; will throw `StringTokenizer` errors if tokens are missing. |
| `stripProperties(String)` | `private IntegrationProperties stripProperties(...)` | Parses a `;`‑delimited string into `IntegrationProperties`. | Similar lack of validation. |
| `storeConfiguration(...)` | `public void storeConfiguration(...)` | Empty – implementation left to the surrounding action. | Might cause confusion if developers expect persistence here. |
| `PsigateParsedElements` | Inner class with getters/setters for XML fields. | POJO used by `Digester`. | Simple and fine. |

---

## 3. Dependencies

| Library | Version (implied) | Purpose |
|---------|------------------|---------|
| **Apache Commons HttpClient (v3.x)** | `org.apache.commons.httpclient.HttpClient`, `org.apache.commons.httpclient.methods.PostMethod` | Issue HTTP requests to Psigate. |
| **Apache Commons Digester** | `org.apache.commons.digester.Digester` | SAX‑based XML parsing of gateway responses. |
| **Service Layer** | `ServiceFactory`, `PaymentService` | Domain persistence via `MerchantPaymentGatewayTrx`. |
| **Utility classes** | `EncryptionUtil`, `CreditCardUtil`, `CurrencyUtil`, `EncryptionUtil` (again) | Stateless helpers for encryption, masking, currency formatting. |
| **Servlet API** | `javax.servlet.http.HttpServletRequest` | Request context for configuration storage (unused). |
| **Java SE** | `java.util.*`, `java.io.*`, `java.math.BigDecimal` | Core collections, I/O, and number handling. |

The code also imports several classes that are never used (`java.lang.reflect`, `java.text.SimpleDateFormat`, etc.). They can be removed to reduce noise.

---

## 4. Additional Notes

### 4.1 Strengths

| Strength | Reason |
|----------|--------|
| **Clear separation of concerns** – Request construction, network communication, response parsing, and persistence are handled in distinct methods. |
| **Audit trail** – persisting both the sent (encrypted) and received XML improves traceability. |
| **Use of existing utilities** – avoids reinventing logic for masking, currency formatting, and encryption. |

### 4.2 Weaknesses & Risks

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hard‑coded XML header** – the `<AddressValidationRequest>` tag is meaningless for payment processing and pollutes the request. | Confusing to maintain; may break Psigate’s XML parser if they ever validate the root element. | Remove it or replace with a meaningful root such as `<PaymentRequest>`. |
| **String concatenation for XML** – no escaping of `<`, `>`, `&`. | Malformed XML if card number or other fields contain these characters. | Use an XML builder (JAXB, DOM, or StAX). |
| **Magic numbers for card actions** – `"0"`, `"1"`, `"2"`, `"3"`. | Hard to read and maintain; easy to mix up. | Define an enum `CardAction { SALE, PRE_AUTH, POST_AUTH, REFUND }` and map to strings. |
| **Duplicated environment selection logic** – repeated blocks in `makeTransaction`, `captureTransaction`, and `refundTransaction`. | Violates DRY; easy to introduce inconsistencies. | Extract into a helper that returns a `UrlDetails` object. |
| **`StringTokenizer` parsing** – no validation, can silently ignore missing tokens. | Configuration can become corrupted without detection. | Use `String.split(";")`, validate length, throw explicit exception if incomplete. |
| **Error handling** – `parseResponse` throws generic `Exception`; callers don’t distinguish between transient network failures and business‑level errors. | Clients may receive a stack trace instead of a user‑friendly message. | Return a `TransactionStatus` enum or enrich `GatewayTransactionVO` with error codes. |
| **Empty `storeConfiguration`** – configuration persistence is delegated elsewhere, but the API contract suggests this class should handle it. | Can cause confusion during maintenance. | Either document that it’s intentionally empty or move the responsibility entirely to the action/controller. |
| **Typo in method name** – `retreiveTransactions` | Will compile but can break automated tests or reflection‑based invocations. | Rename to `retrieveTransactions`. |
| **No time‑out or retry strategy** – `HttpClient` uses default timeout (which is effectively infinite). | Gateway calls can block the thread for a long time. | Configure socket/connect/read timeouts; consider exponential back‑off on transient errors. |
| **Potential thread‑safety issue** – the `EncryptionUtil.generatekey` uses merchant ID; if this key changes, previously persisted encrypted requests become unreadable. | Loss of audit trail. | Store the key in a stable location or include it in the payload. |

### 4.3 Suggested Improvements

1. **Refactor Request/Response Handling**  
   - Create a `PsigateClient` component that encapsulates host selection, request building, and execution.  
   - Use *Apache HttpComponents* (v4.x or v5.x) instead of the deprecated HttpClient v3.

2. **XML Generation & Parsing**  
   - Replace string concatenation with a lightweight XML builder (`javax.xml.stream.XMLOutputFactory`) or JAXB.  
   - Define an XML schema (XSD) and validate responses against it.

3. **Configuration Abstraction**  
   - Store host/protocol/port in a properties file or database rather than hard‑coding the logic.  
   - Keep the environment flag (`props.getProperties1()`) in a dedicated enum.

4. **Error Handling & Reporting**  
   - Distinguish between *gateway* errors and *system* errors via a dedicated `TransactionStatus` enum.  
   - Return meaningful error codes and messages to the UI layer.

5. **Unit / Integration Tests**  
   - Mock the `HttpClient` to simulate success and failure responses.  
   - Verify that `MerchantPaymentGatewayTrx` is persisted correctly and that encryption/decryption round‑trips.

6. **Code Clean‑up**  
   - Remove unused imports.  
   - Rename `retreiveTransactions` to `retrieveTransactions`.  
   - Add JavaDoc for each public method.  
   - Replace magic numbers with constants or enums.

---

### 4.4 Final Verdict

`PsigateTransactionImpl` demonstrates a workable, if somewhat rough, integration with the Psigate gateway. The core flow – request construction, HTTP POST, XML parsing, transaction persistence – is correctly laid out. However, the implementation is fragile, repetitive, and does not fully adhere to modern Java best practices.  

With targeted refactoring (XML handling, configuration abstraction, error handling, and documentation) this class can evolve into a robust, maintainable adapter suitable for production use.

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

import java.io.Reader;
import java.io.StringReader;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.StringTokenizer;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.digester.Digester;
import org.apache.commons.httpclient.HttpClient;
import org.apache.commons.httpclient.methods.PostMethod;
import org.apache.commons.httpclient.methods.RequestEntity;
import org.apache.commons.httpclient.methods.StringRequestEntity;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.payment.MerchantPaymentGatewayTrx;
import com.salesmanager.core.entity.reference.CoreModuleService;
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
import com.salesmanager.core.service.payment.impl.TransactionHelper;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CreditCardUtil;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.EncryptionUtil;
import com.salesmanager.core.util.LogMerchantUtil;

public class PsigateTransactionImpl extends CreditCardGatewayTransactionImpl {

	private Logger log = Logger.getLogger(PsigateTransactionImpl.class);

	public Map<String, String> initTransaction(
			CoreModuleService serviceDefinition, Order order)
			throws TransactionException {
		return null;
	}

	public Order postTransaction(Order order) throws TransactionException {
		return null;
	}

	public GatewayTransactionVO authorizeTransaction(IntegrationKeys keys,
			IntegrationProperties properties, MerchantStore store, Order order, Customer customer, CoreModuleService cms)
			throws TransactionException {
		return makeTransaction("1", keys, properties, store, order, customer, cms);
	}

	public GatewayTransactionVO authorizeAndCapture(IntegrationKeys keys,
			IntegrationProperties properties, MerchantStore store, Order order, Customer customer, CoreModuleService cms)
			throws TransactionException {
		return makeTransaction("0", keys, properties, store, order, customer, cms);
	}

	private GatewayTransactionVO makeTransaction(String type,
			IntegrationKeys ik, IntegrationProperties props,
			MerchantStore store, Order order, Customer customer, CoreModuleService cis) throws TransactionException {

		PostMethod httppost = null;

		try {



			// determine production - test environment
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

			HttpClient client = new HttpClient();

			String xml = "<?xml version=\"1.0\"?><AddressValidationRequest xml:lang=\"en-US\"><Request><TransactionReference><CustomerContext>SalesManager Data</CustomerContext><XpciVersion>1.0001</XpciVersion></TransactionReference><RequestAction>AV</RequestAction></Request>";
			StringBuffer xmldatabuffer = new StringBuffer();
			xmldatabuffer.append("<Order>");
			xmldatabuffer.append("<StoreID>").append(ik.getUserid()).append(
					"</StoreID>");
			xmldatabuffer.append("<Passphrase>").append(ik.getTransactionKey())
					.append("</Passphrase>");
			xmldatabuffer.append("<PaymentType>").append("CC").append(
					"</PaymentType>");
			xmldatabuffer.append("<Subtotal>").append(
					order.getTotal().toString()).append("</Subtotal>");

			// 0=Sale, 1=PreAuth, 2=PostAuth, 3=Credit, 4=Forced PostAuth

			xmldatabuffer.append("<CardAction>").append(type).append(
					"</CardAction>");

			xmldatabuffer.append("<CardNumber>").append(order.getCcNumber())
					.append("</CardNumber>");
			xmldatabuffer.append("<CardExpMonth>").append(
					order.getCcExpires().substring(0, 2)).append(
					"</CardExpMonth>");
			xmldatabuffer.append("<CardExpYear>").append(
					order.getCcExpires().substring(2,
							order.getCcExpires().length())).append(
					"</CardExpYear>");
			// CVV
			if (props.getProperties3().equals("2")) {
				xmldatabuffer.append("<CustomerIP>").append(
						order.getIpAddress()).append("</CustomerIP>");
				xmldatabuffer.append("<CardIDNumber>").append("").append(
						"</CardIDNumber>");
			}
			xmldatabuffer.append("</Order>");
			
			
			
			/** log data **/
			StringBuffer xmllogbuffer = new StringBuffer();
			xmllogbuffer.append("<Order>");
			xmllogbuffer.append("<StoreID>").append(ik.getUserid()).append(
					"</StoreID>");
			xmllogbuffer.append("<Passphrase>").append(ik.getTransactionKey())
					.append("</Passphrase>");
			xmllogbuffer.append("<PaymentType>").append("CC").append(
					"</PaymentType>");
			xmllogbuffer.append("<Subtotal>").append(
					order.getTotal().toString()).append("</Subtotal>");
			xmllogbuffer.append("<CardAction>").append(type).append(
					"</CardAction>");
			xmllogbuffer.append("<CardNumber>").append(CreditCardUtil.maskCardNumber(order.getCcNumber()))
					.append("</CardNumber>");
			xmllogbuffer.append("<CardExpMonth>").append(
					order.getCcExpires().substring(0, 2)).append(
					"</CardExpMonth>");
			xmllogbuffer.append("<CardExpYear>").append(
					order.getCcExpires().substring(2,
							order.getCcExpires().length())).append(
					"</CardExpYear>");
			if (props.getProperties3().equals("2")) {
				xmllogbuffer.append("<CustomerIP>").append(
						order.getIpAddress()).append("</CustomerIP>");
				xmllogbuffer.append("<CardIDNumber>").append("").append(
						"</CardIDNumber>");
			}
			xmllogbuffer.append("</Order>");
			
			log.debug("Psigate request " + xmllogbuffer.toString());
			
			/** log **/

			

			httppost = new PostMethod(protocol + "://" + host + ":" + port
					+ url);
			RequestEntity entity = new StringRequestEntity(xmldatabuffer
					.toString(), "text/plain", "UTF-8");
			httppost.setRequestEntity(entity);

			PsigateParsedElements pe = null;
			String stringresult = null;

			int result = client.executeMethod(httppost);
			if (result != 200) {
				log.error("Communication Error with psigate " + protocol
						+ "://" + host + ":" + port + url);
				throw new Exception("Communication Error with psigate "
						+ protocol + "://" + host + ":" + port + url);
			}
			stringresult = httppost.getResponseBodyAsString();
			log.debug("Psigate response " + stringresult);

			pe = new PsigateParsedElements();
			Digester digester = new Digester();
			digester.push(pe);

			digester.addCallMethod("Result/OrderID", "setOrderID", 0);
			digester.addCallMethod("Result/Approved", "setApproved", 0);
			digester.addCallMethod("Result/ErrMsg", "setErrMsg", 0);
			digester.addCallMethod("Result/ReturnCode", "setReturnCode", 0);
			digester.addCallMethod("Result/TransRefNumber",
					"setTransRefNumber", 0);
			digester.addCallMethod("Result/CardType", "setCardType", 0);

			Reader reader = new StringReader(stringresult);

			digester.parse(reader);

			if (type.equals("0")) {

				return this.parseResponse(PaymentConstants.CAPTURE,
						xmldatabuffer.toString(), stringresult, pe, order,
						order.getTotal());

			} else {

				return this.parseResponse(PaymentConstants.PREAUTH,
						xmldatabuffer.toString(), stringresult, pe, order,
						order.getTotal());

			}

		} catch (Exception e) {
			if (e instanceof TransactionException) {
				throw (TransactionException) e;
			}
			log.error(e);
			TransactionException te = new TransactionException(
					"Psigate Gateway error ", e);
			te.setErrorcode("01");
			throw te;
		} finally {
			if (httppost != null)
				httppost.releaseConnection();
		}

	}

	public GatewayTransactionVO captureTransaction(IntegrationKeys ik, IntegrationProperties props,
			MerchantStore store, Order order, GatewayTransactionVO trx, Customer customer, CoreModuleService cis) throws TransactionException {

		// Get capturable transaction
		PostMethod httppost = null;

		try {


			// determine production - test environment

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

			// Protocol easyhttps = new Protocol("https", new
			// EasySSLProtocolSocketFactory(), 443);
			HttpClient client = new HttpClient();

			String xml = "<?xml version=\"1.0\"?><AddressValidationRequest xml:lang=\"en-US\"><Request><TransactionReference><CustomerContext>SalesManager Data</CustomerContext><XpciVersion>1.0001</XpciVersion></TransactionReference><RequestAction>AV</RequestAction></Request>";
			StringBuffer xmldatabuffer = new StringBuffer();
			xmldatabuffer.append("<Order>");
			xmldatabuffer.append("<StoreID>").append(ik.getUserid()).append(
					"</StoreID>");
			xmldatabuffer.append("<Passphrase>").append(ik.getTransactionKey())
					.append("</Passphrase>");
			xmldatabuffer.append("<PaymentType>").append("CC").append(
					"</PaymentType>");

			// 0=Sale, 1=PreAuth, 2=PostAuth, 3=Credit, 4=Forced PostAuth

			xmldatabuffer.append("<CardAction>").append("2").append(
					"</CardAction>");
			// For postauth only
			xmldatabuffer.append("<OrderID>").append(
					trx.getInternalGatewayOrderId()).append("</OrderID>");
			xmldatabuffer.append("</Order>");
			/**
			 * xmldatabuffer.append("<CardNumber>").append("").append(
			 * "</CardNumber>");
			 * xmldatabuffer.append("<CardExpMonth>").append(""
			 * ).append("</CardExpMonth>");
			 * xmldatabuffer.append("<CardExpYear>")
			 * .append("").append("</CardExpYear>"); //CVV
			 * xmldatabuffer.append("<CustomerIP>"
			 * ).append("").append("</CustomerIP>");
			 * xmldatabuffer.append("<CardIDNumber>"
			 * ).append("").append("</CardIDNumber>");
			 * xmldatabuffer.append("</Order>");
			 **/

			log.debug("Psigate request " + xmldatabuffer.toString());

			httppost = new PostMethod(protocol + "://" + host + ":" + port
					+ url);
			RequestEntity entity = new StringRequestEntity(xmldatabuffer
					.toString(), "text/plain", "UTF-8");
			httppost.setRequestEntity(entity);

			PsigateParsedElements pe = null;
			String stringresult = null;

			int result = client.executeMethod(httppost);
			if (result != 200) {
				log.error("Communication Error with psigate " + protocol
						+ "://" + host + ":" + port + url);
				throw new Exception("Communication Error with psigate "
						+ protocol + "://" + host + ":" + port + url);
			}
			stringresult = httppost.getResponseBodyAsString();
			log.debug("Psigate response " + stringresult);

			pe = new PsigateParsedElements();
			Digester digester = new Digester();
			digester.push(pe);

			digester.addCallMethod("Result/OrderID", "setOrderID", 0);
			digester.addCallMethod("Result/Approved", "setApproved", 0);
			digester.addCallMethod("Result/ErrMsg", "setErrMsg", 0);
			digester.addCallMethod("Result/ReturnCode", "setReturnCode", 0);
			digester.addCallMethod("Result/TransRefNumber",
					"setTransRefNumber", 0);
			digester.addCallMethod("Result/CardType", "setCardType", 0);

			Reader reader = new StringReader(stringresult);

			digester.parse(reader);

			return this.parseResponse(PaymentConstants.CAPTURE, xmldatabuffer
					.toString(), stringresult, pe, order, order.getTotal());

		} catch (Exception e) {
			if (e instanceof TransactionException) {
				throw (TransactionException) e;
			}
			log.error(e);
			TransactionException te = new TransactionException(
					"Psigate Gateway error ", e);
			te.setErrorcode("01");
			throw te;
		} finally {
			if (httppost != null)
				httppost.releaseConnection();
		}

	}

	public GatewayTransactionVO refundTransaction(IntegrationKeys keys, IntegrationProperties props,
			MerchantStore store, Order order, GatewayTransactionVO trx, Customer customer, CoreModuleService cis, BigDecimal amount) throws TransactionException {

		// Get refundable transaction

		PostMethod httppost = null;

		try {

			
			
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

			// String total = CurrencyUtil.getAmount(order.getTotal(),
			// order.getCurrency());
			String total = CurrencyUtil.getAmount(amount, order.getCurrency());

			HttpClient client = new HttpClient();


			String xml = "<?xml version=\"1.0\"?><AddressValidationRequest xml:lang=\"en-US\"><Request><TransactionReference><CustomerContext>SalesManager Data</CustomerContext><XpciVersion>1.0001</XpciVersion></TransactionReference><RequestAction>AV</RequestAction></Request>";
			StringBuffer xmldatabuffer = new StringBuffer();
			xmldatabuffer.append("<Order>");
			xmldatabuffer.append("<StoreID>").append(keys.getUserid()).append(
					"</StoreID>");
			xmldatabuffer.append("<Passphrase>").append(keys.getTransactionKey())
					.append("</Passphrase>");
			xmldatabuffer.append("<PaymentType>").append("CC").append(
					"</PaymentType>");

			// 0=Sale, 1=PreAuth, 2=PostAuth, 3=Credit, 4=Forced PostAuth

			xmldatabuffer.append("<CardAction>").append("3").append(
					"</CardAction>");
			// For postauth only
			xmldatabuffer.append("<OrderID>").append(
					trx.getInternalGatewayOrderId()).append("</OrderID>");
			xmldatabuffer.append("<SubTotal>").append(total).append(
					"</SubTotal>");
			xmldatabuffer.append("</Order>");
			/**
			 * xmldatabuffer.append("<CardNumber>").append("").append(
			 * "</CardNumber>");
			 * xmldatabuffer.append("<CardExpMonth>").append(""
			 * ).append("</CardExpMonth>");
			 * xmldatabuffer.append("<CardExpYear>")
			 * .append("").append("</CardExpYear>"); //CVV
			 * xmldatabuffer.append("<CustomerIP>"
			 * ).append("").append("</CustomerIP>");
			 * xmldatabuffer.append("<CardIDNumber>"
			 * ).append("").append("</CardIDNumber>");
			 * xmldatabuffer.append("</Order>");
			 **/

			log.debug("Psigate request " + xmldatabuffer.toString());

			httppost = new PostMethod(protocol + "://" + host + ":" + port
					+ url);
			RequestEntity entity = new StringRequestEntity(xmldatabuffer
					.toString(), "text/plain", "UTF-8");
			httppost.setRequestEntity(entity);

			PsigateParsedElements pe = null;
			String stringresult = null;

			int result = client.executeMethod(httppost);
			if (result != 200) {
				log.error("Communication Error with psigate " + protocol
						+ "://" + host + ":" + port + url);
				// throw new Exception("Psigate Gateway error ");
				TransactionException te = new TransactionException(
						"Communication Error with psigate " + protocol + "://"
								+ host + ":" + port + url);
				te.setErrorcode("01");
				throw te;
			}
			stringresult = httppost.getResponseBodyAsString();
			log.debug("Psigate response " + stringresult);

			pe = new PsigateParsedElements();
			Digester digester = new Digester();
			digester.push(pe);

			digester.addCallMethod("Result/OrderID", "setOrderID", 0);
			digester.addCallMethod("Result/Approved", "setApproved", 0);
			digester.addCallMethod("Result/ErrMsg", "setErrMsg", 0);
			digester.addCallMethod("Result/ReturnCode", "setReturnCode", 0);
			digester.addCallMethod("Result/TransRefNumber",
					"setTransRefNumber", 0);
			digester.addCallMethod("Result/CardType", "setCardType", 0);

			Reader reader = new StringReader(stringresult);

			digester.parse(reader);

			return this.parseResponse(PaymentConstants.REFUND, xmldatabuffer
					.toString(), stringresult, pe, order, amount);

		} catch (Exception e) {
			if (e instanceof TransactionException) {
				throw (TransactionException) e;
			}
			log.error(e);
			// throw new
			// Exception("Exception occured while calling Psigate gateway " +
			// e);
			TransactionException te = new TransactionException(
					"Communication Error with psigate ", e);
			te.setErrorcode("01");
			throw te;
		} finally {
			if (httppost != null)
				httppost.releaseConnection();
		}

	}

	private GatewayTransactionVO parseResponse(int transactiontype,
			String request, String xmlresponse, PsigateParsedElements response,
			Order order, BigDecimal amount) throws Exception {

		MerchantPaymentGatewayTrx gtrx = null;
		// check if error
		if (response.getErrMsg() != null
				&& !response.getErrMsg().trim().equals("")) {
			LogMerchantUtil.log(order.getMerchantId(),
					"Can't process Psigate message " + response.getErrMsg());
			log.debug("Can't process Psigate message " + response.getErrMsg());
			// throw new Exception("Can't process Psigate message " +
			// response.getErrMsg());
			TransactionException te = new TransactionException(
					"Can't process Psigate message " + response.getErrMsg());
			te.setErrorcode("02");
			te.setReason(response.getErrMsg());
			throw te;
			// End user should see the message
		}


		try {


			PaymentService pservice = (PaymentService) ServiceFactory
					.getService(ServiceFactory.PaymentService);

			gtrx = new MerchantPaymentGatewayTrx();
			gtrx.setMerchantId(order.getMerchantId());
			gtrx.setCustomerid(order.getCustomerId());
			gtrx.setOrderId(order.getOrderId());
			gtrx.setAmount(amount);
			gtrx.setMerchantPaymentGwMethod(order.getPaymentModuleCode());
			gtrx.setMerchantPaymentGwRespcode(response.getReturnCode());
			gtrx.setMerchantPaymentGwOrderid(response.getOrderID());
			gtrx.setMerchantPaymentGwTrxid(response.getTransRefNumber());
			gtrx.setMerchantPaymentGwAuthtype(String.valueOf(transactiontype));
			gtrx.setMerchantPaymentGwSession("");

			String cryptedvalue = EncryptionUtil.encrypt(EncryptionUtil
					.generatekey(String.valueOf(order.getMerchantId())),
					request);
			gtrx.setMerchantPaymentGwSent(cryptedvalue);
			gtrx.setMerchantPaymentGwReceived(xmlresponse);
			gtrx.setDateAdded(new Date(new Date().getTime()));
			gtrx.setAmount(amount);

			pservice.saveMerchantPaymentGatewayTrx(gtrx);


		} catch (Exception e) {

			TransactionException te = new TransactionException(
					"Can't persist MerchantPaymentGatewayTrx internal id (orderid)"
							+ response.getOrderID(), e);
			te.setErrorcode("01");
			throw te;
		}

		GatewayTransactionVO vo = new GatewayTransactionVO();
		vo.setAmount(order.getTotal());
		vo.setCreditcard(response.getCardType());
		vo.setCreditcardtransaction(true);
		vo.setExpirydate(order.getCcExpires());
		vo.setInternalGatewayOrderId(response.getOrderID());
		vo.setTransactionDetails(gtrx);
		return vo;
	}

	public List<SalesManagerTransactionVO> retreiveTransactions(int merchantid,
			Order order) throws Exception {
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

			PsigateParsedElements pe = new PsigateParsedElements();
			Digester digester = new Digester();
			digester.push(pe);

			digester.addCallMethod("Result/OrderID", "setOrderID", 0);
			// digester.addCallMethod("Result/Approved", "setApproved",0);
			// digester.addCallMethod("Result/ErrMsg", "setErrMsg",0);
			digester.addCallMethod("Result/TransRefNumber",
					"setTransRefNumber", 0);
			// digester.addCallMethod("Result/CardType", "setCardType",0);
			digester.addCallMethod("Result/FullTotal", "setFullTotal", 0);

			Reader reader = new StringReader(trx.getMerchantPaymentGwReceived());

			digester.parse(reader);

			GatewayTransactionVO mtrx = new GatewayTransactionVO();

			if (pe.getOrderID() == null) {
				log.error("Can't parse transaction for orderid "
						+ order.getOrderId());
				throw new Exception(
						"Psigate (retreiveTransaction) can't parse XML");
			}

			// GatewayTransaction gt = new GatewayTransaction();
			mtrx.setOrderID(String.valueOf(order.getOrderId()));
			mtrx.setInternalGatewayOrderId(pe.getOrderID());
			mtrx.setTransactionID(pe.getTransRefNumber());
			mtrx.setTransactionDetails(trx);
			mtrx.setType(Integer.parseInt(trx.getMerchantPaymentGwAuthtype()));
			mtrx.setAmount(new BigDecimal(pe.getFullTotal()));
			returnlist.add(mtrx);

		}
		return returnlist;
	}

	public ConfigurationResponse getConfiguration(
			MerchantConfiguration configurations, ConfigurationResponse vo) {

		try {
			// vo.addConfiguration("paymentmethod",
			// configurations.getConfigurationValue());

			String decryptedvalue = EncryptionUtil.decrypt(
					EncryptionUtil.generatekey(String.valueOf(configurations
							.getMerchantId())), configurations
							.getConfigurationValue());
			IntegrationKeys ik = this.stripCredentials(decryptedvalue);
			vo.addConfiguration("keys", ik);

			IntegrationProperties props = this.stripProperties(configurations
					.getConfigurationValue2());
			vo.addConfiguration("properties", props);

			vo.addConfiguration(PaymentConstants.PAYMENT_PSIGATENAME,
					configurations);
		} catch (Exception e) {
			log.error("Can't understand MerchantConfiguration"
					+ configurations.getConfigurationId());
		}

		vo.addMerchantConfiguration(configurations);
		return vo;
	}

	/**
	 * key1 = storeid, transactionkey = password
	 * 
	 * @param configvalue
	 * @return
	 * @throws Exception
	 */
	private IntegrationKeys stripCredentials(String configvalue)
			throws Exception {
		if (configvalue == null)
			return new IntegrationKeys();
		StringTokenizer st = new StringTokenizer(configvalue, ";");
		int i = 1;
		int j = 1;
		IntegrationKeys keys = new IntegrationKeys();
		while (st.hasMoreTokens()) {
			String value = st.nextToken();
	
			if (i == 1) {
				// decrypt
				keys.setUserid(value);
			} else if (i == 2) {
				// decrypt
				keys.setPassword(value);
			} else if (i == 3) {
				// decrypt
				keys.setTransactionKey(value);
			} else {
				if (j == 1) {
					keys.setKey1(value);
				} else if (j == 2) {
					keys.setKey2(value);
				} else if (j == 3) {
					keys.setKey3(value);
				}
				j++;
			}
			i++;
		}
		return keys;
	}

	public void storeConfiguration(int merchantid, ConfigurationResponse vo, HttpServletRequest request)
			throws Exception {

		// managed in Central PaymentpsigateAction

	}

	/**
	 * Properties are 1) Production(1) - Test(2) 2) Pre-Auth(1) - Capture (2) -
	 * Sale (0) 3) No CCV (1) - With CCV (2)
	 * 
	 * @param configvalue
	 * @return
	 */
	private IntegrationProperties stripProperties(String configvalue) {
		if (configvalue == null)
			return new IntegrationProperties();
		StringTokenizer st = new StringTokenizer(configvalue, ";");
		int i = 1;
		IntegrationProperties keys = new IntegrationProperties();
		while (st.hasMoreTokens()) {
			String value = st.nextToken();
			if (i == 1) {
				keys.setProperties1(value);
			} else if (i == 2) {
				keys.setProperties2(value);
			} else if (i == 3) {
				keys.setProperties3(value);
			} else {
				keys.setProperties4(value);
			}
			i++;
		}
		return keys;
	}

}

class PsigateParsedElements {

	private String approved;
	private String returnCode;
	private String errMsg;
	private String orderID;
	private String transRefNumber;
	private String cardType;
	private String fullTotal;

	public String getApproved() {
		return approved;
	}

	public void setApproved(String approved) {
		this.approved = approved;
	}

	public String getErrMsg() {
		return errMsg;
	}

	public void setErrMsg(String errMsg) {
		this.errMsg = errMsg;
	}

	public String getOrderID() {
		return orderID;
	}

	public void setOrderID(String orderID) {
		this.orderID = orderID;
	}

	public String getReturnCode() {
		return returnCode;
	}

	public void setReturnCode(String returnCode) {
		this.returnCode = returnCode;
	}

	public String getTransRefNumber() {
		return transRefNumber;
	}

	public void setTransRefNumber(String transRefNumber) {
		this.transRefNumber = transRefNumber;
	}

	public String getCardType() {
		return cardType;
	}

	public void setCardType(String cardType) {
		this.cardType = cardType;
	}

	public String getFullTotal() {
		return fullTotal;
	}

	public void setFullTotal(String fullTotal) {
		this.fullTotal = fullTotal;
	}

}



```
