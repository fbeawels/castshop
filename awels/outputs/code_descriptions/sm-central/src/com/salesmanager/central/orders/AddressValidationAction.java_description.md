# AddressValidationAction.java

## Review

## 1. Summary  

The **`AddressValidationAction`** class is a Struts‑style action that validates a customer’s shipping address by integrating with external address‑validation services. It extends `EditCustomerAction`, thus inheriting common order‑handling logic. The primary responsibilities of this class are:

| Component | Role |
|-----------|------|
| `AddressValidationAction` | Orchestrates the request for address validation. |
| `AddressMatchVO` | Holds the validation result (currently unused). |
| `ShippingAddressVO` | Represents the shipping address in a format expected by the validation service. |
| `ReferenceService`, `CountryUtil`, `LanguageUtil` | Resolve country/zone codes and language contexts for address normalization. |

The class leverages **Apache Log4j 1.x** for logging and depends on the SalesManager core services for reference data and shipping utilities. Design patterns are minimal – mainly a procedural flow within an action method – but the code demonstrates a standard MVC pattern where the action prepares data for the view.

---

## 2. Detailed Description  

### Core Flow  

1. **Pre‑conditions** – Checks that an `Order` is present and has a valid ID. If not, it sets a technical error and returns an authorization exception result.  
2. **Preparation** – Calls `super.viewShippingCustomer()` to prepare the order and address‑match objects.  
3. **Context Retrieval** – Pulls the current `Context` from the HTTP session to obtain merchant and language information.  
4. **Zone Normalisation** – Uses `CountryUtil.getZoneCodeByName()` (with the language code from `LanguageUtil`) to translate the delivery state into a zone code.  
5. **Address Assembly** – Builds a `ShippingAddressVO` from the order’s delivery fields and the resolved zone code.  
6. **(Missing) Validation Call** – The actual interaction with the address‑validation service is **not** implemented; a placeholder comment marks where this logic should occur.  
7. **Result** – Returns a constant `SUCCESS` (assumed to be defined in a superclass).  

### Assumptions & Constraints  

- **Order Object**: Must be non‑null and possess a positive `orderId`.  
- **Session State**: The `Context` attribute must exist; otherwise a `NullPointerException` will occur.  
- **Zone Lookup**: The `CountryUtil.getZoneCodeByName()` must return a valid `Zone`; no null‑check is performed.  
- **Service Availability**: The external address‑validation service is assumed to be reachable, but this code currently does not invoke it.  
- **Logging**: Uses Log4j 1.x, which is deprecated; newer projects should migrate to Log4j 2.x or SLF4J.  

### Architecture & Design Choices  

- **Action‑Based Architecture**: Follows a classic Struts 1.x pattern – actions perform business logic and return result names.  
- **Separation of Concerns**: Validation logic is delegated to external services (though not yet wired).  
- **Utility Services**: Leverages central reference services for country/zone data rather than hard‑coding values.  

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `getAddressMatch()` | `public String getAddressMatch() throws Exception` | Orchestrates the address validation process; returns a Struts result name. | N/A | `String` (result name) | Logs errors, sets technical/authorization messages, may modify internal state via inherited methods. |
| `getAddressvalidation()` | `public AddressMatchVO getAddressvalidation()` | Getter for the `addressvalidation` field. | N/A | `AddressMatchVO` | None |
| `setAddressvalidation(AddressMatchVO)` | `public void setAddressvalidation(AddressMatchVO)` | Setter for the `addressvalidation` field. | `AddressMatchVO` | None |

### Notable Observations  

- **`addressvalidation` Field**: Declared but never used. It is intended to hold validation results, but the current implementation does not populate it.  
- **`ReferenceService` Instance**: Created (`rservice`) but never used; redundant.  
- **Exception Handling**: Broad `catch (Exception e)` block that logs only the exception reference; stack trace is lost.  

---

## 4. Dependencies  

| External Library | Type | Purpose |
|------------------|------|---------|
| `org.apache.log4j.Logger` | Third‑party (Log4j 1.x) | Logging |
| `com.salesmanager.central.*` | Core application | `AuthorizationException`, `Context`, `ProfileConstants` |
| `com.salesmanager.core.entity.reference.Zone` | Core | Zone data model |
| `com.salesmanager.core.service.reference.ReferenceService` | Core | Reference data retrieval |
| `com.salesmanager.core.service.shipping.*` | Core | Shipping address/value objects |
| `com.salesmanager.core.util.*` | Core | Utility classes for country/language handling |

*All dependencies are part of the SalesManager platform. No external web services are referenced directly in this file.*

---

## 5. Additional Notes  

### Edge Cases / Missing Handling  

1. **Null `Context`** – If the session does not contain `ProfileConstants.context`, a `NullPointerException` will be thrown.  
2. **Zone Lookup Failure** – `CountryUtil.getZoneCodeByName()` might return `null`; subsequent code would crash.  
3. **Order Validation** – Only checks `orderId == 0`; negative IDs or missing fields are not validated.  
4. **Exception Logging** – `log.error(e)` prints the exception’s `toString()` but not the stack trace; useful debugging information is omitted.  
5. **Missing Validation Logic** – The core purpose—interacting with the address‑validation service—is not implemented.  
6. **Unused Fields** – `addressvalidation` and `rservice` are declared but never used, indicating incomplete code.  

### Suggested Enhancements  

- **Implement Validation** – Wire the actual call to the external address‑validation API and populate `addressvalidation`.  
- **Populate the Result** – Return a meaningful result string (`SUCCESS`, `ERROR`, etc.) that reflects validation status.  
- **Clean Up Unused Code** – Remove or implement the `ReferenceService` instance and `addressvalidation` field.  
- **Null‑Safety** – Add explicit null checks for `Context`, `Zone`, and other critical objects.  
- **Logging Improvements** – Use `log.error("Message", e)` to capture stack traces.  
- **Modern Logging** – Migrate to Log4j 2.x or SLF4J for better performance and flexibility.  
- **Unit Tests** – Add tests covering normal validation, failure scenarios, and exception handling.  
- **Dependency Injection** – Use a DI framework (e.g., Spring) to inject services instead of creating them manually.  

By addressing these points the class will become robust, maintainable, and truly fulfil its intended role in address validation.

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

import org.apache.log4j.Logger;

import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.service.shipping.AddressMatchVO;
import com.salesmanager.core.service.shipping.ShippingAddressVO;
import com.salesmanager.core.service.shipping.ShippingService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.LanguageUtil;

/**
 * @description Uses address validation external systems for validating shipping
 *              address exactitude
 * @author Carl Samson
 * 
 */
public class AddressValidationAction extends EditCustomerAction {

	private Logger log = Logger.getLogger(AddressValidationAction.class);

	private AddressMatchVO addressvalidation;

	// AddressValidation Functionality
	public String getAddressMatch() throws Exception {

		try {

			if (this.getOrder() == null || this.getOrder().getOrderId() == 0) {
				super.setTechnicalMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			super.viewShippingCustomer();// prepare the order and address match
											// objects

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			// convert delivery to isocode 2
			ReferenceService rservice = new ReferenceService();
			Zone z = CountryUtil.getZoneCodeByName(super.getOrder()
					.getDeliveryState(), LanguageUtil.getLanguageNumberCode(ctx
					.getLang()));

			ShippingAddressVO svo = new ShippingAddressVO(super.getOrder()
					.getDeliveryName(), super.getOrder()
					.getDeliveryStreetAddress(), super.getOrder()
					.getDeliveryCity(), super.getOrder().getDeliveryPostcode(),
					z.getZoneCode(), super.getOrder().getDeliveryCountry());

			// Not implemented

			return SUCCESS;

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return ERROR;
		}

	}

	public AddressMatchVO getAddressvalidation() {
		return addressvalidation;
	}

	public void setAddressvalidation(AddressMatchVO addressvalidation) {
		this.addressvalidation = addressvalidation;
	}

}



```
