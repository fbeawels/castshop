# CreditCardPaymentModule.java

## Review

## 1. Summary  
The file defines **`CreditCardPaymentModule`**, a pure interface that represents the contract for handling credit‑card related transactions in the Sales Manager core module.  
- **Purpose**: Expose two core operations – *capture* an authorized payment and *refund* a completed sale – for any concrete payment gateway implementation.  
- **Key Components**:
  - `processCapture(...)` – completes an authorization that was previously held.  
  - `processRefund(...)` – reverses a previously settled sale.  
- **Design patterns / frameworks**:  
  - *Strategy/Template Method*: Concrete gateway classes implement this interface to plug different payment providers.  
  - *Domain‑Driven Design*: The interface lives in the `integration` package and relies on domain entities (`Order`, `Customer`, `MerchantStore`).  

The interface is lightweight, allowing any payment gateway implementation to be injected wherever a `CreditCardPaymentModule` is required.

---

## 2. Detailed Description  
1. **Initialization**  
   - No state or constructors are involved because this is an interface.  
   - The actual implementation will be instantiated by a dependency‑injection container (Spring, CDI, etc.) and injected into services that need to perform capture/refund operations.  

2. **Runtime Behavior**  
   - **`processCapture`** is called after an order has been authorized.  
     - Parameters:  
       - `Order order` – the order whose payment is to be captured.  
       - `MerchantStore store` – the merchant context (currency, locale, etc.).  
       - `Customer customer` – the payer’s profile (needed for AVS, 3DS, etc.).  
       - `String paymentModule` – the identifier of the gateway module (e.g., “PayPal”, “AuthorizeNet”).  
     - Returns a `GatewayTransactionVO` that contains transaction status, ID, and any gateway‑specific data.  
   - **`processRefund`** is invoked after a sale has settled.  
     - Same parameters as capture, plus a `BigDecimal amount` for partial refunds.  

   Both methods can throw `TransactionException`, allowing the calling code to decide how to roll back or retry.

3. **Cleanup**  
   - Since the interface has no resources, cleanup is handled by the concrete implementation if necessary (e.g., closing HTTP clients, clearing caches).

4. **Assumptions & Constraints**  
   - The caller is responsible for ensuring that the `order`, `store`, and `customer` objects are fully populated.  
   - No null‑checking or validation is performed at this level – it is expected to be handled by the implementation or by upstream services.  
   - The interface assumes that a string identifier (`paymentModule`) is sufficient to select the concrete gateway logic; the implementation must map this string to the right gateway instance.  

5. **Overall Architecture**  
   - The core domain layer defines this contract so that higher‑level services (e.g., `PaymentService`) can remain agnostic of specific gateway APIs.  
   - Implementations may use adapters (e.g., to REST or SOAP endpoints) and populate the `GatewayTransactionVO` accordingly.

---

## 3. Functions/Methods  
| Method | Purpose | Inputs | Output | Side‑Effects / Exceptions |
|--------|---------|--------|--------|----------------------------|
| `processCapture(Order, MerchantStore, Customer, String)` | Finalize an authorized credit‑card payment. | - `Order order` – order to capture.<br>- `MerchantStore store` – merchant context.<br>- `Customer customer` – payer profile.<br>- `String paymentModule` – gateway identifier. | `GatewayTransactionVO` – contains transaction ID, status, response codes, and any additional data from the gateway. | May throw `TransactionException` if the gateway rejects the capture or a network error occurs. |
| `processRefund(Order, MerchantStore, Customer, BigDecimal, String)` | Reverse a settled sale. | Same as `processCapture` plus:<br>- `BigDecimal amount` – amount to refund (supports partial refunds). | `GatewayTransactionVO` – contains refund transaction details. | May throw `TransactionException` on failure. |

### Reusable / Utility Methods  
- None defined directly in the interface.  
- Implementations typically provide helper methods for HTTP communication, XML/JSON parsing, and data transformation.

---

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `java.math.BigDecimal` | Standard Java | Used for monetary amounts. |
| `com.salesmanager.core.entity.customer.Customer` | Domain entity | Represents the payer. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Domain entity | Holds merchant configuration. |
| `com.salesmanager.core.entity.orders.Order` | Domain entity | Contains order details and payment information. |
| `com.salesmanager.core.service.payment.GatewayTransactionVO` | Value object | Encapsulates transaction result data. |
| `com.salesmanager.core.service.payment.TransactionException` | Custom exception | Signals payment‑related errors. |

All dependencies are part of the `salesmanager` core library, making the interface self‑contained for the application’s domain layer. No external frameworks (e.g., Spring) are referenced directly, keeping the interface lightweight.

---

## 5. Additional Notes  
### Strengths  
- **Separation of Concerns**: Keeps payment gateway logic out of business services.  
- **Extensibility**: New gateways can be added by simply implementing this interface.  
- **Clear Contract**: Method signatures and return types are well defined, promoting consistency across implementations.  

### Areas for Improvement  
1. **Parameter Documentation**  
   - Javadoc currently lists only the parameter names but not their semantics. Adding descriptions would aid developers implementing the interface.  

2. **Null Safety**  
   - The interface trusts callers to supply non‑null arguments. A defensive strategy (e.g., `Objects.requireNonNull`) could be added in implementations or in a wrapper to avoid `NullPointerException`s downstream.  

3. **Payment Module Identification**  
   - Using a `String` for `paymentModule` is fragile. Consider an enum (`PaymentModuleType`) or a dedicated key‑value lookup so that typos are caught at compile time.  

4. **Error Handling Granularity**  
   - `TransactionException` is a generic wrapper. Distinguishing between transient (network) and permanent (invalid credentials) errors via sub‑exceptions or error codes can improve retry logic.  

5. **Return Type Richness**  
   - `GatewayTransactionVO` should expose status codes, messages, and raw gateway responses to aid debugging. If not already present, adding a `Map<String, String>` for gateway‑specific fields is useful.  

### Edge Cases  
- **Partial vs Full Refunds**: The method accepts any `BigDecimal amount`. Implementation must validate that the amount does not exceed the settled transaction value.  
- **Concurrent Captures**: If multiple capture requests are made for the same order, the implementation must handle idempotency.  
- **Currency Mismatch**: The interface does not enforce that the `order` currency matches the `store` currency; implementations should verify and handle conversions.  

### Future Enhancements  
- **Support for 3DS / AVS**: Adding optional callback URLs or challenge responses could be required for some gateways.  
- **Asynchronous Processing**: Expose async variants (e.g., `CompletableFuture<GatewayTransactionVO>`) for non‑blocking integration.  
- **Audit Logging**: Provide a hook to log every transaction attempt, success, or failure.  

--- 

**Overall**, the interface is concise and well‑intentioned, fitting cleanly into a domain‑driven architecture. Strengthening documentation, tightening contract types, and anticipating edge cases will make it even more robust for production use.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Sep 17, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.module.model.integration;

import java.math.BigDecimal;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.service.payment.GatewayTransactionVO;
import com.salesmanager.core.service.payment.TransactionException;

/**
 * Specific transactions for credit card
 * @author Carl Samson
 *
 */
public interface CreditCardPaymentModule {
	
	/**
	 * Capture a transaction that has been authorized
	 * 
	 * @param origincountryid
	 * @param order
	 * @return
	 * @throws TransactionException
	 */

	
	public GatewayTransactionVO processCapture(Order order, MerchantStore store, Customer customer, String paymentModule)
			throws TransactionException;
	
	/**
	 * Can be invoked after a SALE transaction
	 * 
	 * @param origincountryid
	 * @param order
	 * @return
	 * @throws TransactionException
	 */
	
	public GatewayTransactionVO processRefund(Order order, MerchantStore store, Customer customer, BigDecimal amount, String paymentModule)
		throws TransactionException;
	


}



```
