# PayPalExpressCheckoutAction.java

## Review

## 1. Summary  
**Purpose & Functionality**  
`PayPalExpressCheckoutAction` is a Struts‑type action (it extends `CheckoutBaseAction`) that orchestrates the PayPal Express Checkout flow for the SalesManager e‑commerce platform. It performs three distinct responsibilities:

1. **`preparePaypalRequest`** – Initiates the Express Checkout request, obtains a PayPal token, and stores it in the HTTP session.  
2. **`preparePaypalResponse`** – Handles the return from PayPal after the user authorises the payment. It retrieves shipping and payer details, populates a `Customer` object (if the visitor is not logged in), and forwards the user to the next checkout step.  
3. **`payPalNotification`** – A placeholder for handling PayPal IPN (Instant Payment Notification) callbacks; currently only logs the incoming attributes.

**Key Components**  
- **`PaymentService`** – Used to pre‑initialise the payment and obtain the token.  
- **`ReferenceService` & `CoreModuleService`** – Provide localisation data and module configurations.  
- **`PaypalTransactionImpl`** – Encapsulates PayPal NVP API interactions.  
- **`SessionUtil`** – Utility for reading/writing session data such as `MerchantStore`, `Order`, `PaymentMethod`, and `Customer`.  

**Design Patterns & Libraries**  
- *Service Locator* (`ServiceFactory.getService`) for obtaining core services.  
- *Spring Bean Retrieval* (`SpringUtil.getBean`) for the PayPal module.  
- *Utility/Helper* classes (`CountryUtil`, `SessionUtil`) are heavily used.  
- Logging via **log4j**.  
- String utilities from **Apache Commons Lang**.

---

## 2. Detailed Description  

### Flow of Execution  

| Method | Entry Point | Core Actions | Success Path | Failure Path |
|--------|-------------|--------------|--------------|--------------|
| `preparePaypalRequest` | User selects PayPal at checkout | - Retrieve `MerchantStore`, `Order`, `PaymentMethod` from session.<br>- Set payment module code on the order.<br>- Call `PaymentService.preInitializePayment` to get PayPal NVP map.<br>- Validate presence of `TOKEN`.<br>- Store token in session and set request attributes. | Returns `SUCCESS` (forward to PayPal URL). | Logs error, adds user‑facing error message, returns `"PAYMENTERROR"`. |
| `preparePaypalResponse` | PayPal redirects back after authorisation | - Retrieve `store`, `order`, `paymentMethod` from session.<br>- Get the stored token.<br>- Resolve the module bean (`PaypalTransactionImpl`).<br>- Call `getShippingDetails` to fetch payer & shipping NVP.<br>- Populate `PaymentMethod` with `TOKEN` & `PAYERID`.<br>- If the customer is anonymous and the order is not an invoice channel, create a new `Customer`, map PayPal fields to the customer entity, resolve country IDs via `CountryUtil`, and store the customer in the session.<br>- Set the next checkout step (`STEP=1`). | Returns `SUCCESS` (forward to summary page). | Handles `TransactionException` (maps to `"PAYMENTERROR"`). Handles generic exceptions (logs, sets technical message, returns `"GENERICERROR"`). |
| `payPalNotification` | PayPal IPN endpoint | - Iterate over request attributes and build a query‑string representation.<br>- Log the resulting string. | Returns `SUCCESS` (no further action). | Catches all exceptions, logs them. |

### Initialization & Cleanup  
- No explicit resource initialization beyond service lookups.  
- No cleanup is required; the action relies on the container’s lifecycle.  

### Assumptions & Constraints  
- **Session Integrity**: Assumes `MerchantStore`, `Order`, `PaymentMethod`, and optionally `Customer` are already present in the session; otherwise, NPEs may occur.  
- **Locale**: Uses `getLocale()` from the action to resolve country descriptions.  
- **Payment Channel**: Skips creating a `Customer` when the channel is `INVOICE_CHANNEL`.  
- **Error Handling**: Uses generic strings (`"error.payment.paymenterror"`) for user feedback; no localization beyond that.  
- **PayPal Integration**: Relies on a custom `PaypalTransactionImpl` bean; the API contract is not shown.  

### Architecture & Design Choices  
- **Service Layer**: Payment and reference services are obtained via a `ServiceFactory`, decoupling the action from concrete implementations.  
- **Session‑centric**: All state is stored in the HTTP session; this is typical for checkout flows but may become problematic in a clustered environment without sticky sessions.  
- **Hard‑coded Response Names**: The action returns literal strings (`SUCCESS`, `"PAYMENTERROR"`, `"GENERICERROR"`). A more robust design would use constants or an enumeration.  
- **Data Mapping**: A lot of manual field mapping between PayPal NVP names and `Customer` properties; could be refactored into a dedicated mapper.

---

## 3. Functions/Methods  

### `preparePaypalRequest()`  
- **Purpose**: Initiates the Express Checkout flow.  
- **Inputs**: None directly; pulls data from the HTTP session.  
- **Outputs**: Sets `PAYPALTOKEN` in the session and `TRANSACTIONTOKEN` in the request, then returns a navigation string.  
- **Side‑effects**: Logs errors, updates the session, populates request attributes.  

### `preparePaypalResponse()`  
- **Purpose**: Processes PayPal’s return after payment authorisation.  
- **Inputs**: None directly; pulls token from session and uses services to fetch shipping details.  
- **Outputs**: Populates the `PaymentMethod`, potentially creates a `Customer` and stores it in the session, sets a checkout step attribute, and returns a navigation string.  
- **Side‑effects**: Extensive mapping of NVP fields to a `Customer` instance, logging, error handling.  

### `payPalNotification()`  
- **Purpose**: Stub for handling IPN messages.  
- **Inputs**: None directly; iterates over request attributes.  
- **Outputs**: Logs the request payload and returns `SUCCESS`.  
- **Side‑effects**: Only logging; no state changes.  

---

## 4. Dependencies  

| Library / Component | Type | Notes |
|---------------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Used for null/blank checks. |
| `org.apache.log4j.Logger` | Third‑party | Standard log4j logger. |
| `com.salesmanager.checkout.CheckoutBaseAction` | Local | Base action providing request handling, locale, and error messaging. |
| `com.salesmanager.core.constants.OrderConstants` | Local | Contains channel constants. |
| `com.salesmanager.core.entity.*` (Customer, MerchantStore, Order, PaymentMethod, ShippingInformation, etc.) | Local | JPA/Hibernate entities. |
| `com.salesmanager.core.entity.reference.*` (CoreModuleService, CountryDescription) | Local | Reference data entities. |
| `com.salesmanager.core.module.impl.integration.payment.PaypalTransactionImpl` | Local | PayPal NVP integration implementation. |
| `com.salesmanager.core.service.*` (PaymentService, ServiceFactory, ReferenceService) | Local | Service layer interfaces. |
| `com.salesmanager.core.util.*` (CountryUtil, SpringUtil, SessionUtil) | Local | Utility classes for country lookups, Spring bean resolution, session handling. |
| `javax.servlet.http.HttpServletRequest` (implied) | Standard | Servlet API for request handling. |
| `java.util` (Enumeration, Map) | Standard | Java collections. |

No external REST or XML libraries are used; all PayPal interaction occurs via the `PaypalTransactionImpl` bean which likely encapsulates HTTP NVP calls.

---

## 5. Additional Notes  

### Strengths  
- **Clear separation** of request initiation, response handling, and IPN handling.  
- **Consistent use** of session utilities keeps code DRY for session access.  
- **Service abstraction** allows swapping payment implementations without changing the action.  

### Weaknesses & Risks  
1. **Hard‑coded strings** for navigation (`"PAYMENTERROR"`, `"GENERICERROR"`). Using constants would reduce typos and ease refactoring.  
2. **Exception handling** is overly broad; many catch blocks swallow specific details and return generic error messages.  
3. **Customer creation logic** is deeply nested and manually maps many fields. It would be cleaner to extract a `PayPalToCustomerMapper` component.  
4. **Null checks**: Methods assume non‑null session attributes; if any are missing, the code will throw `NullPointerException`.  
5. **Internationalisation**: Only `error.payment.paymenterror` is used; no dynamic localisation.  
6. **Session‑centric**: In a distributed environment, the session must be replicated or sticky‑sessioned; otherwise, the token might be lost.  
7. **Logging**: Sensitive PayPal data (tokens, PAYERID) is not masked; this could expose sensitive information in logs.  
8. **IPN handling** is incomplete – it only logs the incoming data and does not validate the IPN, which is a critical security step.  

### Future Enhancements  
- **Extract mapping logic** into a dedicated mapper or builder class.  
- **Improve error handling**: Use a custom exception hierarchy and a global exception mapper.  
- **Secure logging**: Mask or hash sensitive values before logging.  
- **Complete IPN flow**: Validate IPN signatures, update order status, and handle duplicate notifications.  
- **Use dependency injection** for services (e.g., via Spring) instead of `ServiceFactory` lookups.  
- **Introduce unit tests** for the mapping logic and error paths.  
- **Move navigation strings to constants** or an enum for maintainability.  
- **Add validation** for session data early in the action lifecycle.  

Overall, the action fulfills its purpose but would benefit from refactoring to improve maintainability, testability, and security.

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

import java.util.Enumeration;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.core.constants.OrderConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.shipping.ShippingInformation;
import com.salesmanager.core.module.impl.integration.payment.PaypalTransactionImpl;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.payment.PaymentService;
import com.salesmanager.core.service.payment.TransactionException;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class PayPalExpressCheckoutAction extends CheckoutBaseAction {

	private Logger log = Logger.getLogger(PayPalExpressCheckoutAction.class);

	public String preparePaypalRequest() {

		try {

			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());
			Order order = SessionUtil.getOrder(super.getServletRequest());
			PaymentMethod paymentMethod = SessionUtil.getPaymentMethod(super
					.getServletRequest());

			order.setPaymentModuleCode(paymentMethod.getPaymentModuleName());

			PaymentService pservice = (PaymentService) ServiceFactory
					.getService(ServiceFactory.PaymentService);
			Map tokens = pservice.preInitializePayment(store, order);

			// check if there is a registration inthe payment

			if (tokens == null || tokens.get("TOKEN") == null) {
				log.error("No token received from PayPal");
				super.addErrorMessage("error.payment.paymenterror");
				return "PAYMENTERROR";
			}

			String tk = (String) tokens.get("TOKEN");
			String paymentType = (String) tokens.get("PAYMENTTYPE");
			paymentMethod.addInfo("PAYMENTTYPE", paymentType);

			super.getServletRequest().getSession().setAttribute("PAYPALTOKEN",
					tk);

			super.getServletRequest().setAttribute("TRANSACTIONTOKEN", tokens);

			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			super.addErrorMessage("error.payment.paymenterror");
			return "PAYMENTERROR";
		}

	}

	public String preparePaypalResponse() {

		try {

			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());
			Order order = SessionUtil.getOrder(getServletRequest());
			PaymentMethod paymentMethod = SessionUtil
					.getPaymentMethod(getServletRequest());

			String token = (String) super.getServletRequest().getSession()
					.getAttribute("PAYPALTOKEN");

			ReferenceService refservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			CoreModuleService cms = refservice.getCoreModuleService(store
					.getCountry(), order.getPaymentModuleCode());

			PaypalTransactionImpl module = (PaypalTransactionImpl) SpringUtil
					.getBean(order.getPaymentModuleCode());

			Map nvp = module.getShippingDetails(token, order, cms);

			if (nvp == null) {
				log
						.warn("Did not received any customer information from Paypal");
				return SUCCESS;
			}

			String payerId = (String) nvp.get("PAYERID"); // ' Unique PayPal
			// customer account
			// identification
			// number.

			paymentMethod.addInfo("TOKEN", token);
			paymentMethod.addInfo("PAYERID", payerId);

			ShippingInformation shippingInformation = SessionUtil
					.getShippingInformation(getServletRequest());

			if (shippingInformation != null) {
				return "SUMMARY";
			}

			String email = (String) nvp.get("EMAIL"); // ' Email address of
			// payer.

			String payerStatus = (String) nvp.get("PAYERSTATUS"); // ' Status of
			// payer.
			// Character
			// length
			// and
			// limitations:
			// 10
			// single-byte
			// alphabetic
			// characters.

			/**
			 * If customer is logged in, do not use paypal customer information
			 */
			Customer tmpCustomer = SessionUtil.getCustomer(super
					.getServletRequest());

			if (order.getChannel() != OrderConstants.INVOICE_CHANNEL
					&& tmpCustomer == null) {

				Customer customer = new Customer();
				String salutation = (String) nvp.get("SALUTATION"); // ' Payer's
				// salutation.
				String firstName = (String) nvp.get("FIRSTNAME"); // ' Payer's
																	// first
				// name.
				String middleName = (String) nvp.get("MIDDLENAME"); // ' Payer's
				// middle name.
				String lastName = (String) nvp.get("LASTNAME"); // ' Payer's
																// last
				// name.
				String suffix = (String) nvp.get("SUFFIX"); // ' Payer's suffix.
				String cntryCode = (String) nvp.get("COUNTRYCODE"); // ' Payer's
				// country of
				// residence in
				// the form of
				// ISO standard
				// 3166
				// two-character
				// country
				// codes.
				String business = (String) nvp.get("BUSINESS"); // ' Payer's
				// business name.

				if (!StringUtils.isBlank(cntryCode)) {
					CountryDescription cDescription = CountryUtil
							.getCountryByIsoCode(cntryCode, super.getLocale());
					if (cDescription == null) {
						log
								.error("Cannot find CountryDescription from paypal country code "
										+ cntryCode);
					} else {
						customer.setCustomerBillingCountryId(cDescription
								.getId().getCountryId());
						customer.setCustomerBillingCountryName(cDescription
								.getCountryName());
					}
				}

				customer.setCustomerBillingCompany(business);
				String billingFirstName = firstName;
				if (!StringUtils.isBlank(middleName)) {
					billingFirstName = billingFirstName + " " + middleName;
				}
				customer.setCustomerBillingFirstName(billingFirstName);
				customer.setCustomerBillingLastName(lastName);

				customer.setCustomerEmailAddress(email);

				String shipToName = (String) nvp.get("SHIPTONAME"); // '
																	// Person's
				// name
				// associated
				// with this
				// address.
				String shipToStreet = (String) nvp.get("SHIPTOSTREET"); // '
																		// First
				// street
				// address.
				String shipToStreet2 = (String) nvp.get("SHIPTOSTREET2"); // '
				// Second
				// street
				// address.
				String shipToCity = (String) nvp.get("SHIPTOCITY"); // ' Name of
				// city.
				String shipToState = (String) nvp.get("SHIPTOSTATE"); // ' State
																		// or
				// province
				String shipToCntryCode = (String) nvp.get("SHIPTOCOUNTRYCODE"); // '
				// Country
				// code.
				String shipToZip = (String) nvp.get("SHIPTOZIP"); // ' U.S. Zip
																	// code
				// or other
				// country-specific
				// postal code.

				String addressStatus = (String) nvp.get("ADDRESSSTATUS"); // '
				// Status
				// of
				// street
				// address
				// on
				// file
				// with
				// PayPal
				String invoiceNumber = (String) nvp.get("INVNUM"); // ' Your own
				// invoice or
				// tracking
				// number, as
				// set by you in
				// the element
				// of the same
				// name in
				// SetExpressCheckout
				// request .
				String phonNumber = (String) nvp.get("PHONENUM"); // ' Payer's
				// contact
				// telephone
				// number. Note:
				// PayPal
				// returns a
				// contact
				// telephone
				// number only
				// if your
				// Merchant
				// account
				// profile
				// settings
				// require that
				// the buyer
				// enter one.

				if (!StringUtils.isBlank(shipToCntryCode)) {
					CountryDescription cDescription = CountryUtil
							.getCountryByIsoCode(shipToCntryCode, super
									.getLocale());
					if (cDescription == null) {
						log
								.error("Cannot find ShippingCountryDescription from paypal country code "
										+ shipToCntryCode);
					} else {
						customer.setCustomerCountryId(cDescription.getId()
								.getCountryId());
						customer.setCountryName(cDescription.getCountryName());
					}
				}

				customer.setCustomerBillingCountryName(shipToState);

				customer.setCustomerFirstname(shipToName);
				customer.setCustomerCity(shipToCity);
				customer.setCustomerPostalCode(shipToZip);
				customer.setCustomerTelephone(phonNumber);

				String shippingAddress = shipToStreet;
				if (!StringUtils.isBlank(shipToStreet2)) {
					shippingAddress = shippingAddress + " " + shipToStreet2;
				}
				customer.setCustomerStreetAddress(shippingAddress);
				customer.setLocale(getLocale());
				customer.setCustomerLang(getLocale().getLanguage());

				SessionUtil.setCustomer(customer, getServletRequest());

			}

			// prepare steps
			super.getServletRequest().setAttribute("STEP", 1);

		} catch (Exception e) {

			if (e instanceof TransactionException) {
				// if (((TransactionException) e).getErrorcode().equals("01")) {
				super.addErrorMessage("error.payment.paymenterror");
				return "PAYMENTERROR";
				// }
			} else {

				log.error(e);
				super.setTechnicalMessage();
				return "GENERICERROR";

			}
		}

		return SUCCESS;

	}

	public String payPalNotification() {

		try {

			Enumeration attributesName = this.getServletRequest()
					.getAttributeNames();
			StringBuffer postBack = new StringBuffer();
			int i = 0;
			while (attributesName != null && attributesName.hasMoreElements()) {
				i++;
				String attributeName = (String) attributesName.nextElement();
				String attributeValue = (String) this.getServletRequest()
						.getAttribute(attributeName);
				postBack.append(attributeName).append("=").append(
						attributeValue).append("&");
			}

			log.debug("Values received from IPN " + postBack.toString());

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

}



```
