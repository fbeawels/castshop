# CustomerDetailsAction.java

## Review

## 1. Summary  

`CustomerDetailsAction` is a Struts‑style action that handles all CRUD‑related operations for a `Customer` entity in the **SalesManager** web application.  
* **Purpose** – Display customer details (create/edit), validate and persist customer data, reset passwords, and provide lists of companies, countries, and zones for form dropdowns.  
* **Key components**  
  * `CustomerService` – business service for CRUD operations.  
  * `RefCache` – in‑memory cache for reference data (countries, zones).  
  * `Context` – holds session‑level information such as the current merchant ID and country.  
  * `ProfileConstants`, `LabelUtil`, `MessageUtil`, `LanguageUtil` – utilities for i18n, messaging, and language handling.  
* **Patterns & libraries**  
  * **Factory** (`ServiceFactory`) for obtaining services.  
  * **Cache** (`RefCache`) for quick lookup of reference data.  
  * **Logging** – Apache Log4j.  
  * **Commons Lang** – `StringUtils` for null/blank checks.  

The class tightly couples UI logic (error messages, view titles) with business logic, which is common in older Struts applications but can be improved with more separation.

---

## 2. Detailed Description  

### Execution Flow  

1. **Action Entry Points**  
   * `displaySelectCompany()`: populates a list of distinct company names for the current merchant.  
   * `displayCustomerCreate()`: prepares the form for creating a new customer, setting defaults and internationalization.  
   * `displayCustomerDetails()`: loads an existing customer, authorises the user, and prepares the edit form.  
   * `resetPassword()`: triggers a password reset for the currently loaded customer.  
   * `createCustomer()`: handles both create and update flows – it validates, maps form fields to the domain model, and persists via `CustomerService`.

2. **Common Helpers**  
   * `setCountry()`: sets default or session country IDs, loads country and zone lists from `RefCache`.  
   * `prepareCustomerDetails()`: loads the customer from the database, authorises, and prepares UI state (e.g., `state`, `billingState`).  

3. **Validation**  
   * The `createCustomer()` method contains inline, field‑by‑field checks, adding field errors via `super.addFieldError`.  
   * No regex/email format checks; only non‑blank validation.  

4. **Persisting**  
   * After validation, the customer object is enriched with billing information (either copy of shipping or separate billing fields).  
   * `cservice.saveOrUpdateCustomer()` is called with the customer and locale.  

5. **Error Handling**  
   * `AuthorizationException` triggers a special page.  
   * All other exceptions are logged and result in a generic error page.  

6. **Return Values**  
   * `SUCCESS`, `ERROR`, `AUTHORIZATIONEXCEPTION` – typical Struts result strings.

### Dependencies & Constraints  

* **Context** must be present in the HTTP session; otherwise `ctx` will be `null`.  
* The action assumes a default country ID in `core.system.defaultcountryid`; if missing, defaults to the US.  
* `RefCache` must be pre‑populated with country and zone data; otherwise dropdowns will be empty.  
* The code is **not thread‑safe** as it stores state in instance fields (`customer`, `state`, etc.). Struts creates a new action instance per request, so this is acceptable, but care must be taken if the framework changes.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `displaySelectCompany()` | Populates `companyList` with distinct company names for current merchant. | None (reads session context). | Returns `SUCCESS`. | Logs errors. |
| `displayCustomerCreate()` | Prepares an empty `Customer` instance for the create form. | None (reads session context). | Returns `SUCCESS`. | Sets session attribute `COUNTRY`. |
| `displayCustomerDetails()` | Loads existing customer for editing, authorises access. | `customer.customerId` from action property. | Returns `SUCCESS` or `AUTHORIZATIONEXCEPTION`. | Populates form data. |
| `setCountry()` | Determines shipping/billing country IDs, loads countries and zones into collections. | None (reads `customer` & session context). | None. | Populates `countries`, `shippingZonesByCountry`, `billingZonesByCountry`. |
| `prepareCustomerDetails()` | Loads a customer from DB, authorises, sets UI state. | `customer.customerId`. | None. | Sets `state`/`billingState`, populates `customer`. |
| `resetPassword()` | Triggers password reset for current customer. | None. | Returns `SUCCESS` or `ERROR`. | Calls `cservice.resetCustomerPassword`. |
| `createCustomer()` | Handles create or update logic with validation, mapping, and persistence. | `customer` property, form fields. | Returns `SUCCESS`, `ERROR`, or `AUTHORIZATIONEXCEPTION`. | Updates database, sets UI messages. |
| `getCustomer()` / `setCustomer(Customer)` | Accessors for `customer`. | N/A. | `Customer`. | No side‑effects. |
| `getCompanyList()` / `setCompanyList(Collection)` | Accessors for `companyList`. | N/A. | `Collection`. | No side‑effects. |
| `getState()` / `setState(String)` | Accessors for `state`. | N/A. | `String`. | No side‑effects. |
| `getBillingState()` / `setBillingState(String)` | Accessors for `billingState`. | N/A. | `String`. | No side‑effects. |
| `getSetbilling()` / `setSetbilling(int)` | Accessors for `setbilling` flag. | N/A. | `int`. | No side‑effects. |
| `getBillingZonesByCountry()` / `setBillingZonesByCountry(Collection<Zone>)` | Accessors for billing zones list. | N/A. | `Collection<Zone>`. | No side‑effects. |
| `getCountries()` / `setCountries(Collection<Country>)` | Accessors for country list. | N/A. | `Collection<Country>`. | No side‑effects. |
| `getShippingZonesByCountry()` / `setShippingZonesByCountry(Collection<Zone>)` | Accessors for shipping zones list. | N/A. | `Collection<Zone>`. | No side‑effects. |

> **Note** – The class uses raw `Collection` types for several fields (`companyList`, `countries`, etc.) even though generic type parameters are declared for zone and country collections. This inconsistency can lead to `ClassCastException` and hampers compile‑time safety.

---

## 4. Dependencies  

| Library / Package | Type | Notes |
|-------------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For null/blank checks. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.central.*` | Project | BaseAction, AuthorizationException, ProfileConstants, etc. |
| `com.salesmanager.core.*` | Project | Domain entities (Customer, Country, Zone), services, utilities. |
| `com.salesmanager.core.service.cache.RefCache` | Project | In‑memory reference data cache. |
| `com.salesmanager.core.service.customer.CustomerService` | Project | CRUD for Customer. |
| `com.salesmanager.core.util.*` | Project | LabelUtil, LanguageUtil, MessageUtil, PropertiesUtil. |
| `com.salesmanager.core.constants.Constants` | Project | Constant values (e.g., US country ID). |

No external APIs or platform‑specific features are required beyond standard Java EE servlet container support.

---

## 5. Additional Notes  

### Strengths  
* **Separation of concerns** – business logic is delegated to `CustomerService`.  
* **Internationalization** – uses `LabelUtil` and language utilities throughout.  
* **Caching** – reference data is retrieved from `RefCache`, reducing database hits.  

### Weaknesses & Edge Cases  
1. **Raw Types & Generics** – Mixing raw and generic collections leads to unsafe casts.  
2. **Null Handling** – Many methods assume non‑null `Context`, `customer`, and `Locale`. A missing session or null fields will throw `NullPointerException`.  
3. **Validation** – Only non‑blank checks; email format, telephone format, and duplicate email checks are missing or simplistic.  
4. **State Parsing** – `Integer.parseInt` on `state`/`billingState` without validation may throw `NumberFormatException` if the value is not numeric. The exception is swallowed, but the fallback logic might leave inconsistent state.  
5. **Error Messages** – The action writes generic `ERROR` result even when the issue is a user input error. It would be better to keep field errors and redisplay the form.  
6. **Thread‑safety** – The action relies on Struts to create a new instance per request, which is fine, but any future change to a singleton scope would break it.  
7. **Magic Numbers** – Hard‑coded defaults like `-1`, `1`, `0` for `setbilling` lack clarity; constants would improve readability.  
8. **Security** – Password reset is performed without any CSRF protection or audit logging.  
9. **Performance** – `setCountry()` reloads all countries and zones on every request, even for edit mode; could be cached per session or per merchant.  

### Potential Enhancements  
* **Refactor Validation** – Extract validation into a separate `CustomerValidator` bean or use Struts form validation XML.  
* **Use Generics Consistently** – Replace raw `Collection` types with parameterized ones (`List<Country>`, `List<Zone>`).  
* **Improve Error Handling** – Return distinct error codes for validation vs system errors; preserve field errors.  
* **Add Input Sanitization** – Trim whitespace, escape HTML to prevent XSS.  
* **Add Logging Context** – Include merchant ID, customer ID in log statements for better traceability.  
* **Unit Tests** – Create tests for the action logic (e.g., createCustomer) to assert validation, persistence, and message setting.  
* **Security Enhancements** – Add CSRF tokens for password reset, audit logs for changes, enforce password strength.  
* **Constants for Flags** – Replace `setbilling` magic values with an enum or constant fields (`USE_SHIPPING_ADDRESS_FOR_BILLING`, `USE_SEPARATE_BILLING_ADDRESS`).  

Overall, the action is functional and follows typical Struts patterns, but modernizing it with stronger typing, clearer validation, and better separation of concerns would greatly improve maintainability, testability, and security.

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
package com.salesmanager.central.customer;

import java.util.ArrayList;
import java.util.Collection;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.SystemUrlEntryType;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.PropertiesUtil;

public class CustomerDetailsAction extends BaseAction {

	private Customer customer;

	private Collection<Zone> shippingZonesByCountry = new ArrayList();
	private Collection<Country> countries;
	private Collection<Zone> billingZonesByCountry = new ArrayList();

	private String state;
	private String billingState;

	private int setbilling = -1;

	private Collection companyList = new ArrayList();

	private Logger log = Logger.getLogger(CustomerDetailsAction.class);

	public String displaySelectCompany() {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			this.setCompanyList(cservice
					.getUniqueCustomerCompanyNameList(merchantid));

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;
	}

	/**
	 * Displays the form in create mode
	 * 
	 * @return
	 */
	public String displayCustomerCreate() {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		setCountry();

		super.prepareLanguages();

		super.getServletRequest().getSession().setAttribute("COUNTRY",
				ctx.getCountryid());

		Customer c = new Customer();
		c.setCustomerCountryId(ctx.getCountryid());
		c.setCustomerBillingCountryId(ctx.getCountryid());

		this.setCustomer(c);

		return SUCCESS;

	}

	/**
	 * Displays the form in edit mode
	 * 
	 * @return
	 */
	public String displayCustomerDetails() {

		try {
			
			super.setPageTitle("label.customer.customerdetails.title");

			if (this.getCustomer() == null
					|| this.getCustomer().getCustomerId() == 0) {
				super.setAuthorizationMessage();
				return "AUTHORIZATIONEXCEPTION";
			}

			this.prepareCustomerDetails();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "AUTHORIZATIONEXCEPTION";
		}

		return SUCCESS;

	}

	private void setCountry() {

		int shippingCountryId = PropertiesUtil.getConfiguration().getInt(
				"core.system.defaultcountryid", Constants.US_COUNTRY_ID);
		int billingCountryId = PropertiesUtil.getConfiguration().getInt(
				"core.system.defaultcountryid", Constants.US_COUNTRY_ID);

		if (this.getCustomer() != null) {
			shippingCountryId = this.getCustomer().getCustomerCountryId();
			billingCountryId = this.getCustomer().getCustomerBillingCountryId();

			if (this.getCustomer().getCustomerCountryId() == 0) {

				shippingCountryId = super.getContext().getCountryid();
			}

			if (this.getCustomer().getCustomerBillingCountryId() == 0) {

				billingCountryId = super.getContext().getCountryid();
			}
		} else {
			Customer c = new Customer();
			c.setCustomerCountryId(shippingCountryId);
			c.setCustomerBillingCountryId(billingCountryId);
			this.setCustomer(c);
		}

		Collection lcountries = RefCache.getAllcountriesmap(
				LanguageUtil.getLanguageNumberCode(super.getLocale()
						.getLanguage())).values();
		setCountries(lcountries);

		Collection shipZones = RefCache.getFilterdByCountryZones(
				shippingCountryId, LanguageUtil.getLanguageNumberCode(super
						.getLocale().getLanguage()));
		Collection billZones = RefCache.getFilterdByCountryZones(
				billingCountryId, LanguageUtil.getLanguageNumberCode(super
						.getLocale().getLanguage()));

		if (billZones != null && billZones.size() > 0) {
			setBillingZonesByCountry(billZones);
		}

		if (shipZones != null && shipZones.size() > 0) {
			setShippingZonesByCountry(shipZones);
		}

	}

	private void prepareCustomerDetails() throws Exception {
		
		super.setPageTitle("label.customer.customerdetails.title");

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		CustomerService cservice = (CustomerService) ServiceFactory
				.getService(ServiceFactory.CustomerService);


		Customer c = cservice.getCustomer(this.getCustomer().getCustomerId());

		if (c == null) {
			throw new AuthorizationException("Customer is null for customerId "
					+ this.getCustomer().getCustomerId());
			// Check if user is authorized

		}

		super.prepareLanguages();

		super.authorize(c);

		this.setCustomer(c);

		this.setCountry();

		if (this.getCustomer().getCustomerZoneId() == 0) {
			this.setState(this.getCustomer().getCustomerState());
		} else {
			this.setState(String
					.valueOf(this.getCustomer().getCustomerZoneId()));
		}

		if (this.getCustomer().getCustomerBillingZoneId() == 0) {
			this.setBillingState(this.getCustomer().getCustomerBillingState());
		} else {
			this.setBillingState(String.valueOf(this.getCustomer()
					.getCustomerBillingZoneId()));
		}
		// }

	}

	public String resetPassword() {

		try {
			this.prepareCustomerDetails();
			Customer c = this.getCustomer();

			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			cservice.resetCustomerPassword(c);

			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance()
					.getText("label.customer.passwordresetnotice"));

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return ERROR;
		}

		return SUCCESS;

	}

	/**
	 * Creates or edit a Customer
	 * 
	 * @return
	 */
	public String createCustomer() {
		
		super.setPageTitle("label.customer.customerdetails.title");

		try {

			this.setCountry();

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			Customer c = this.getCustomer();

			if (c.getMerchantId() > 0) {// in edit mode
				super.authorize(c);
			}

			// validation
			boolean hasError = false;
			if (StringUtils.isBlank(c.getCustomerFirstname())) {
				super.addFieldError("customer.customerFirstname",
						getText("messages.required.firstname"));
				hasError = true;
			}

			if (StringUtils.isBlank(c.getCustomerLastname())) {
				super.addFieldError("customer.customerLastname",
						getText("messages.required.firstname"));
				hasError = true;
			}

			if (StringUtils.isBlank(c.getCustomerEmailAddress())) {
				super.addFieldError("customer.customerEmailAddress",
						getText("messages.required.email"));
				hasError = true;
			}

			if (StringUtils.isBlank(c.getCustomerTelephone())) {
				super.addFieldError("customer.customerTelephone",
						getText("messages.required.phone"));
				hasError = true;
			}

			if (StringUtils.isBlank(c.getCustomerCity())) {
				super.addFieldError("customer.customerCity",
						getText("messages.required.city"));
				hasError = true;
			}

			if (StringUtils.isBlank(c.getCustomerPostalCode())) {
				super.addFieldError("customer.customerPostalCode",
						getText("messages.required.postalcode"));
				hasError = true;
			}

			if (StringUtils.isBlank(c.getCustomerStreetAddress())) {
				super.addFieldError("customer.customerStreetAddress",
						getText("messages.required.streetaddress"));
				hasError = true;
			}

			if (hasError) {
				return ERROR;
			}

			String state = this.getState();

			int stateId = 0;
			try {
				stateId = Integer.parseInt(this.getState());
				c.setCustomerZoneId(stateId);
				c.setCustomerState(" ");
			} catch (Exception ignore) {
				c.setCustomerState(this.getState());
				c.setCustomerZoneId(0);
			}

			if (this.getSetbilling() == 1) {
				c.setCustomerBillingFirstName(c.getCustomerFirstname());
				c.setCustomerBillingLastName(c.getCustomerLastname());
				c.setCustomerBillingCity(c.getCustomerCity());
				c.setCustomerBillingCountryId(c.getCustomerCountryId());
				c.setCustomerBillingPostalCode(c.getCustomerPostalCode());
				c.setCustomerBillingState(c.getCustomerState());
				c.setCustomerBillingStreetAddress(c.getCustomerStreetAddress());
				c.setCustomerBillingZoneId(c.getCustomerZoneId());

			} else {

				String billingState = this.getState();

				int billingStateId = 0;
				try {
					billingStateId = Integer.parseInt(this.getBillingState());
					c.setCustomerBillingZoneId(billingStateId);
					c.setCustomerBillingState(" ");
				} catch (Exception ignore) {
					c.setCustomerBillingState(this.getBillingState());
					c.setCustomerBillingZoneId(0);
				}

				if (StringUtils.isBlank(c.getCustomerBillingFirstName())) {
					super.addFieldError("customer.customerBillingFirstname",
							getText("messages.required.billing.firstname"));
					hasError = true;
				}

				if (StringUtils.isBlank(c.getCustomerBillingLastName())) {
					super.addFieldError("customer.customerBillingLastname",
							getText("messages.required.billing.lastname"));
					hasError = true;
				}

				if (StringUtils.isBlank(c.getCustomerBillingStreetAddress())) {
					super.addFieldError(
							"customer.customerBillingStreetAddress",
							getText("messages.required.streetaddress"));
					hasError = true;
				}

				if (StringUtils.isBlank(c.getCustomerBillingCity())) {
					super.addFieldError("customer.customerBillingCity",
							getText("messages.required.billing.city"));
					hasError = true;
				}

				if (StringUtils.isBlank(c.getCustomerBillingPostalCode())) {
					super.addFieldError("customer.customerBillingPostalCode",
							getText("messages.required.billing.postalcode"));
					hasError = true;
				}

				if (hasError) {
					return ERROR;
				}

			}

			c.setMerchantId(merchantid);

			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);

			if (!customer.isCustomerAnonymous()) {// changing state
				// look for an existing non anonymous customer with the same
				// nick name
				Customer cust = cservice.findCustomerByUserName(customer
						.getCustomerEmailAddress(), super.getContext()
						.getMerchantid());
				if (cust != null) {
					if (cust.getCustomerId() != customer.getCustomerId()) {
						super
								.setErrorMessage("messages.customer.nonanonymous.alreadyexist");
						return ERROR;
					}
				}

				customer.setCustomerNick(customer.getCustomerEmailAddress());
			}

			// get current customer
			if (customer.getCustomerId() > 0 && !customer.isCustomerAnonymous()) {// existing
																					// customer
				Customer tmpCustomer = cservice.getCustomer(customer
						.getCustomerId());
				// if was anonymous and become a real customer, check if one
				// exist first
				// add a column anonymous to customer list
				if (StringUtils.isBlank(tmpCustomer.getCustomerNick())) {
					tmpCustomer.setCustomerNick(customer
							.getCustomerEmailAddress());
				} else {
					customer.setCustomerNick(tmpCustomer.getCustomerNick());
					customer.setCustomerPassword(tmpCustomer
							.getCustomerPassword());

				}

			}

			cservice.saveOrUpdateCustomer(c, SystemUrlEntryType.WEB, super
					.getLocale());

			if (this.getCustomer().getCustomerZoneId() == 0) {
				this.setState(this.getCustomer().getCustomerState());
			} else {
				this.setState(String.valueOf(this.getCustomer()
						.getCustomerZoneId()));
			}

			if (this.getCustomer().getCustomerBillingZoneId() == 0) {
				this.setBillingState(this.getCustomer()
						.getCustomerBillingState());
			} else {
				this.setBillingState(String.valueOf(this.getCustomer()
						.getCustomerBillingZoneId()));
			}

			this.setCustomer(c);

			super.setSuccessMessage();

		} catch (AuthorizationException ae) {
			super.setAuthorizationMessage();
			return "AUTHORIZATIONEXCEPTION";
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return ERROR;
		}

		return SUCCESS;

	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

	public Collection getCompanyList() {
		return companyList;
	}

	public void setCompanyList(Collection companyList) {
		this.companyList = companyList;
	}

	public String getState() {
		return state;
	}

	public void setState(String state) {
		this.state = state;
	}

	public String getBillingState() {
		return billingState;
	}

	public void setBillingState(String billingState) {
		this.billingState = billingState;
	}

	public int getSetbilling() {
		return setbilling;
	}

	public void setSetbilling(int setbilling) {
		this.setbilling = setbilling;
	}

	public Collection<Zone> getBillingZonesByCountry() {
		return billingZonesByCountry;
	}

	public void setBillingZonesByCountry(Collection<Zone> billingZonesByCountry) {
		this.billingZonesByCountry = billingZonesByCountry;
	}

	public Collection<Country> getCountries() {
		return countries;
	}

	public void setCountries(Collection<Country> countries) {
		this.countries = countries;
	}

	public Collection<Zone> getShippingZonesByCountry() {
		return shippingZonesByCountry;
	}

	public void setShippingZonesByCountry(
			Collection<Zone> shippingZonesByCountry) {
		this.shippingZonesByCountry = shippingZonesByCountry;
	}

}



```
