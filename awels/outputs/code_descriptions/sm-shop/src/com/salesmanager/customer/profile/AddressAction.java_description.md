# AddressAction.java

## Review

## 1. Summary

The **`AddressAction`** class is a Struts‑style action that manages the display and update of a customer’s shipping address.  
Key responsibilities:

| Component | Role |
|-----------|------|
| `prepareZones(int)` | Loads the list of available countries and zones for a given country code from the cache. |
| `changeAddressForm()` | Pre‑populates the address form with the current customer data and the list of zones. |
| `changeAddress()` | Validates submitted address fields, persists the updated customer record, and updates the session. |
| Getters / Setters | Provide access to action properties that the JSP views consume. |

Design patterns / frameworks in use:

* **Action** (Struts) – the class extends `SalesManagerBaseAction` and returns `"SUCCESS"` / `"INPUT"`.
* **Factory** – `ServiceFactory` supplies a `CustomerService` instance.
* **Cache** – `RefCache` is used to pull country/zone data.
* **Session‑utility** – `SessionUtil` retrieves/updates `Customer` and `MerchantStore` objects from the HTTP session.

---

## 2. Detailed Description

### 2.1 Core Flow

| Phase | Method | Key Steps |
|-------|--------|-----------|
| **Initial Request (GET)** | `changeAddressForm()` | 1. Load `Customer` from session.<br>2. Determine the default zone (state or zone ID).<br>3. Call `prepareZones()` with the customer’s country.<br>4. Return `"SUCCESS"` (JSP renders the form). |
| **Form Submission (POST)** | `changeAddress()` | 1. Re‑load zones (redundant).<br>2. Determine default zone again.<br>3. Validate each field (email, name, street, city, postal, state if text, phone).<br>4. If any validation error → return `"INPUT"` (form re‑displayed with messages).<br>5. Retrieve `MerchantStore`, copy merchantId, nick, password from the session customer.<br>6. Persist the updated `Customer` via `CustomerService`.<br>7. Set success message, update session, and return `"SUCCESS"`. |

### 2.2 Assumptions & Constraints

* **Session‑bound data** – The action relies on `SessionUtil` to fetch and store the `Customer` and `MerchantStore`. It assumes a valid HTTP session exists.
* **Locale‑aware lookup** – Country and zone lists are filtered by the current locale’s language code via `LanguageUtil.getLanguageNumberCode(...)`.
* **Thread‑safety** – Struts actions are usually instantiated per request, so no shared mutable state is expected.
* **Validation** – Uses `StringUtils` to check for blanks and a custom `CustomerUtil.validateEmail`. No regex or length limits are applied beyond the required presence.

### 2.3 Architecture & Design Choices

* **Coupling to Cache** – `RefCache` is a static cache, which simplifies lookups but couples the action tightly to that caching layer.
* **Service Layer** – Business logic for persisting customers is delegated to `CustomerService`, adhering to separation of concerns.
* **Form State Handling** – The `formState` field controls whether the state field is treated as a text input or a drop‑down, a pragmatic but somewhat opaque solution.
* **Error Messaging** – Uses Struts field messages (`addFieldMessage`) with message keys, enabling internationalisation.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `prepareZones(int shippingCountryId)` | Populates `countries` and `zones` for the UI drop‑downs. | `shippingCountryId` – country id to filter zones. | Sets `this.countries` and `this.zones`. | Reads from static cache. |
| `changeAddressForm()` | Initialises the address editing form. | None (uses session). | Returns `"SUCCESS"` or `"SUCCESS"` even on error. | Reads `Customer` from session; sets default zone; calls `prepareZones`. |
| `changeAddress()` | Handles POST of address changes. | Form‑bound `customer` object (populated by Struts). | Returns `"INPUT"` on validation error, otherwise `"SUCCESS"`. | Validates fields, persists via `CustomerService`, updates session, sets messages. |
| Getters / Setters | Provide access to action properties. | N/A | Return or set property value. | None. |

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String helper utilities. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.common.SalesManagerBaseAction` | Local | Base Struts action providing i18n and message helpers. |
| `com.salesmanager.core.entity.customer.Customer` | Local | Domain model. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Local | Domain model. |
| `com.salesmanager.core.entity.reference.Country` | Local | Domain model. |
| `com.salesmanager.core.entity.reference.Zone` | Local | Domain model. |
| `com.salesmanager.core.service.ServiceFactory` | Local | Factory for service objects. |
| `com.salesmanager.core.service.cache.RefCache` | Local | Static cache for reference data. |
| `com.salesmanager.core.service.customer.CustomerService` | Local | Business logic for customers. |
| `com.salesmanager.core.util.CustomerUtil` | Local | Utility for email validation. |
| `com.salesmanager.core.util.LanguageUtil` | Local | Locale helper. |
| `com.salesmanager.core.util.www.SessionUtil` | Local | Session helper for Customer/MerchantStore. |

All dependencies are internal to the *salesmanager* codebase or common open‑source libraries. No external web services or platform‑specific APIs are invoked.

---

## 5. Additional Notes

### 5.1 Strengths
* **Clear separation** between UI handling (action) and persistence (service layer).
* **Internationalisation** support via locale‑based cache lookups and message keys.
* **Simple validation logic** that is easy to understand and modify.

### 5.2 Potential Issues & Edge Cases

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Redundant `prepareZones` call** in `changeAddress()` | Minor performance hit; possible mismatch if customer country changes during submission. | Remove the second call or move the zone selection logic to a single place. |
| **Missing null checks** (e.g., `customer.getCustomerState()`) | Could throw `NullPointerException` if session is lost or form data incomplete. | Add guard clauses; validate presence before accessing. |
| **Hard‑coded field‑message keys** | If messages change, the code must be updated manually. | Use a central validation utility or annotations to map fields to messages. |
| **Zone/state handling** | The `formState` field is loosely documented; its value determines required validation. | Refactor into an enum or separate form objects for text vs. dropdown. |
| **Thread‑safety** | Although Struts typically creates a new action per request, any static cache modifications are not thread‑safe. | Ensure `RefCache` operations are synchronized or immutable. |
| **Email validation** | `CustomerUtil.validateEmail` may accept invalid patterns or reject valid ones. | Replace with a robust regex or use Apache Commons Validator. |
| **Password/nick preservation** | Copies nick and password from the session; if the user changes password elsewhere, it may be overwritten. | Explicitly handle password changes or document the behaviour. |
| **Logging** | Errors are logged but not rethrown or propagated to the user. | Consider setting an error message for the user or rethrowing to a global error handler. |

### 5.3 Future Enhancements

1. **Form Validation Layer** – Replace inline validation with a dedicated validator (e.g., Struts `Validator` framework or JSR‑380 Bean Validation).  
2. **DTO Usage** – Map form data to a `CustomerDTO` to decouple the UI model from the persistence entity.  
3. **Error Handling** – Centralise error handling via exception mappings and user‑friendly messages.  
4. **Unit Tests** – Add unit tests for `AddressAction` using a mock `SessionUtil` and `CustomerService`.  
5. **Cache Invalidation** – Implement listeners to refresh country/zone lists when reference data changes.  
6. **Security** – Add CSRF protection and audit logging for address changes.

--- 

**Overall Assessment:**  
The class is straightforward and fulfills its purpose within the larger SalesManager application. Refactoring for cleaner validation, eliminating redundant code, and tightening null safety would improve maintainability and robustness.

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
package com.salesmanager.customer.profile;

import java.util.ArrayList;
import java.util.Collection;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.common.SalesManagerBaseAction;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.SystemUrlEntryType;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.CustomerUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class AddressAction extends SalesManagerBaseAction {

	private Logger log = Logger.getLogger(AddressAction.class);

	private Customer customer;
	private Collection<Zone> zones = new ArrayList();// collection for drop down
														// list
	private Collection<Country> countries;// collection for drop down list

	private String formState = null;
	private String defaultZone = "";

	private void prepareZones(int shippingCountryId) throws Exception {

		Collection lcountries = RefCache.getAllcountriesmap(
				LanguageUtil.getLanguageNumberCode(super.getLocale()
						.getLanguage())).values();
		this.setCountries(lcountries);

		Collection lzones = RefCache.getFilterdByCountryZones(
				shippingCountryId, LanguageUtil.getLanguageNumberCode(super
						.getLocale().getLanguage()));
		this.setZones(lzones);
	}

	/**
	 * Displays customer Address
	 * 
	 * @return
	 */
	public String changeAddressForm() {

		try {

			customer = SessionUtil.getCustomer(super.getServletRequest());

			if (!StringUtils.isBlank(customer.getCustomerState())) {
				this.setDefaultZone(customer.getCustomerState());
			} else {
				this.setDefaultZone(String
						.valueOf(customer.getCustomerZoneId()));
			}

			prepareZones(customer.getCustomerCountryId());

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	/*
	 * Changes customer Address
	 */
	public String changeAddress() {

		try {

			prepareZones(customer.getCustomerCountryId());

			if (!StringUtils.isBlank(customer.getCustomerState())) {
				this.setDefaultZone(customer.getCustomerState());
			} else {
				this.setDefaultZone(String
						.valueOf(customer.getCustomerZoneId()));
			}

			// validate submited fields
			boolean hasError = false;
			if (StringUtils.isBlank(customer.getCustomerEmailAddress())) {
				super.addFieldMessage("customer.customerEmailAddress",
						"messages.required.email");
				hasError = true;
			} else {
				if (!CustomerUtil.validateEmail(customer
						.getCustomerEmailAddress())) {
					super.addFieldMessage("customer.customerEmailAddress",
							"messages.invalid.email");
					hasError = true;
				}
			}
			if (StringUtils.isBlank(customer.getCustomerFirstname())) {
				super.addFieldMessage("customer.customerFirstname",
						"messages.required.firstname");
				hasError = true;
			}
			if (StringUtils.isBlank(customer.getCustomerLastname())) {
				super.addFieldMessage("customer.customerLastname",
						"messages.required.lastname");
				hasError = true;
			}
			if (StringUtils.isBlank(customer.getCustomerStreetAddress())) {
				super.addFieldMessage("customer.customerStreetAddress",
						"messages.required.setreetaddress");
				hasError = true;
			}
			if (StringUtils.isBlank(customer.getCustomerCity())) {
				super.addFieldMessage("customer.customerCity",
						"messages.required.city");
				hasError = true;
			}
			if (StringUtils.isBlank(customer.getCustomerPostalCode())) {
				super.addFieldMessage("customer.customerPostalCode",
						"messages.required.postalcode");
				hasError = true;
			}
			if (!StringUtils.isBlank(this.getFormState())
					&& this.getFormState().equals("text")) {
				if (StringUtils.isBlank(customer.getCustomerState())) {
					super.addFieldMessage("customer.customerState",
							"messages.required.state");
					hasError = true;
				}
			}
			if (StringUtils.isBlank(customer.getCustomerTelephone())) {
				super.addFieldMessage("customer.customerTelephone",
						"messages.required.telephone");
				hasError = true;
			}
			if (hasError) {
				return INPUT;
			}

			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());

			customer.setMerchantId(store.getMerchantId());
			Customer tmpCustomer = SessionUtil.getCustomer(super
					.getServletRequest());
			if (tmpCustomer != null) {
				customer.setCustomerNick(tmpCustomer.getCustomerNick());
				customer.setCustomerPassword(tmpCustomer.getCustomerPassword());
			}
			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			cservice.saveOrUpdateCustomer(customer, SystemUrlEntryType.WEB,
					super.getLocale());
			super.setMessage("messages.customeraddress.changed");
			SessionUtil.setCustomer(customer, super.getServletRequest());

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String getFormState() {
		return formState;
	}

	public void setFormState(String formState) {
		this.formState = formState;
	}

	public Collection<Zone> getZones() {
		return zones;
	}

	public void setZones(Collection<Zone> zones) {
		this.zones = zones;
	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

	public Collection<Country> getCountries() {
		return countries;
	}

	public void setCountries(Collection<Country> countries) {
		this.countries = countries;
	}

	public String getDefaultZone() {
		return defaultZone;
	}

	public void setDefaultZone(String defaultZone) {
		this.defaultZone = defaultZone;
	}

}



```
