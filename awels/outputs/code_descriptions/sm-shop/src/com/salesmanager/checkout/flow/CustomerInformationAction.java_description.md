# CustomerInformationAction.java

## Review

## 1. Summary

**Purpose**  
`CustomerInformationAction` is a Struts‑style action that drives the customer‑information step of a checkout flow. It gathers and validates customer details (billing and shipping addresses, e‑mail, telephone, payment method) and updates the current order and session state.

**Key components**

| Component | Role |
|-----------|------|
| `displayCustomer()` | Renders the customer‑information form, populates country/zone dropdowns and prepares credit‑card UI if needed. |
| `logonCustomer()` | Authenticates a returning customer and loads their profile into the session. |
| `validateShippingBilling()` | Validates the submitted customer data, updates the order total, and persists the customer record back into the session. |
| `prepare()` | Shared helper that configures payment‑method and credit‑card objects. |
| `orderHasShipping()` | Determines if any product in the order requires shipping. |

**Patterns / Libraries**

* **Action‑Controller** – follows Struts 1/2 conventions (`execute`, `input`, `success`).  
* **DAO/Service Layer** – uses `MerchantService`, `ServiceFactory` and custom `SessionUtil` / `RefUtil` utilities.  
* **Validation** – manual field‑by‑field validation with `addFieldError` and `super.addFieldMessage`.  
* **Caching** – reference data (countries, zones) retrieved via `RefCache`.  
* **Internationalisation** – `getText(...)` and locale handling throughout.

---

## 2. Detailed Description

### Flow of execution

1. **Request handling** – The action is invoked by the Struts dispatcher.  
2. **Preparation** – `prepare()` loads the payment method from the session, sets up a credit‑card instance if the chosen payment module is a card, and populates the request with the selected payment method.  
3. **Display** –  
   * `displayCustomer()` pulls the merchant store, loads (or creates) a `Customer` bean, populates country/zone lists, and sets attributes that the JSP will use to render dropdowns.  
   * `logonCustomer()` authenticates a user via `LogonAction` and updates the session.  
4. **Validation** – `validateShippingBilling()` runs when the customer submits the form. It:  
   * Re‑populates country/zone lists.  
   * Validates e‑mail, billing/shipping addresses, phone, and, if applicable, credit‑card fields.  
   * Handles the “use shipping info as billing” flag by copying values.  
   * Calls the merchant service to retrieve configuration data (e.g., whether shipping is domestic).  
   * Updates the order total via `super.updateOrderTotal(...)`.  
   * Persists the customer and payment method back into the session.  
5. **Return value** – Depending on whether shipping is required, the action returns `SUCCESS` or `"SUCCESS-SUMMARY"`; otherwise, it returns `INPUT` on validation failure.

### Assumptions & Constraints

* **Session‑bound data** – Customer, order, and payment method are stored in the HTTP session; concurrent requests for the same session must be serialized by the container.  
* **Locale** – The system assumes a valid `Locale` is available via `super.getLocale()`.  
* **Reference data** – Countries, zones and merchant configurations are fetched from in‑memory caches (`RefCache`) or service calls; these must be kept in sync.  
* **No Bean Validation** – The code relies on custom string‑based checks rather than JSR‑303/JSR‑380 annotations.  
* **Error handling** – Exceptions are caught at the top level, logged, and a generic “technical” message is shown; no fine‑grained error propagation.

### Architecture & Design Choices

* **Monolithic Action** – All checkout logic resides in a single class; responsibilities are tightly coupled.  
* **Manual Validation** – While giving fine‑grained control, it makes the code verbose and harder to test.  
* **Use of SessionUtil & RefUtil** – Utility classes centralise session manipulation and reference data lookup, but also hide complexity.  
* **Legacy API Style** – Raw types, unchecked casts, and the use of `Map` without generics indicate a legacy codebase that predates Java generics.  

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `orderHasShipping()` | Determines if any order product requires shipping. | None (reads session order) | `boolean` |
| `prepare()` | Loads payment and credit‑card objects from the session and configures the request. | None | Sets `paymentMethod`, `hasCreditCardPayment`, `creditCard`, request attributes |
| `displayCustomer()` | Renders the customer form, populating country/zone lists. | None | Sets request attributes, session customer; returns `SUCCESS` or error |
| `logonCustomer()` | Authenticates a user and loads their profile. | None | Sets session customer; returns `SUCCESS` or `INPUT` |
| `validateShippingBilling()` | Validates submitted customer data, updates the order, and persists customer & payment info. | None | Returns `"SUCCESS-SUMMARY"`, `SUCCESS`, or `INPUT` |
| Getters/Setters | Simple accessors for fields (e.g., `getCustomer`, `setCustomer`). | None | Return/modify the corresponding field |

### Reusable / Utility Methods

* **`prepare()`** – Shared by all public actions; could be extracted into the base class or a dedicated helper.  
* **`orderHasShipping()`** – Pure helper that could be unit‑tested in isolation.  

---

## 4. Dependencies

| External | Type | Notes |
|----------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For null/blank checks. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.*` | In‑house | Core entities, services, utilities, and constants. |
| `ServiceFactory` / `MerchantService` | In‑house | Retrieves configuration. |
| `RefCache` / `RefUtil` | In‑house | Caches for countries/zones. |
| `SessionUtil` | In‑house | Session access helper. |
| `LogonAction` | In‑house | Authenticates a customer. |
| `javax.servlet.http.*` | Standard | Implied by `ServletActionContext` (not shown). |
| `java.util.*` | Standard | Collections, maps, etc. |

All third‑party libraries are legacy (log4j 1.x, Commons Lang 2.x). No modern frameworks (e.g., Spring, JPA) are used.

---

## 5. Additional Notes & Recommendations

### Strengths

* **Comprehensive Validation** – Covers a wide range of fields and conditions (shipping vs. billing, credit card checks).  
* **Session‑centric Flow** – Keeps checkout state in the session, simplifying multi‑step navigation.  
* **Internationalisation** – Uses `Locale` and language codes to fetch localized country/zone data.

### Weaknesses & Edge Cases

1. **Duplicate Email Validation**  
   * The repeated e‑mail check incorrectly uses the same field key (`customer.customerEmailAddress`) for both error messages. It should use `customer.customerEmailAddressRepeat`.  
   * There is **no equality check** between the two e‑mail addresses; the system only validates each independently.

2. **Raw Types & Unchecked Casts**  
   * The code uses raw `Map`, `Collection`, and unchecked casts (e.g., `(OrderProduct) orderProducts.get(line)`). This introduces potential `ClassCastException` and hinders IDE/static‑analysis warnings.

3. **Null‑Pointer Risks**  
   * Many getters return `null` if the session is missing a key; the code sometimes proceeds without null checks (e.g., `customer.getCustomerBillingCountryId()`).  
   * The `orderHasShipping()` method will silently return `false` if the session is empty; an explicit session‑missing check would improve robustness.

4. **Hard‑coded Language Logic**  
   * Language is derived from `Locale` and mapped to numeric IDs via `LanguageUtil`. This coupling makes it difficult to add new languages without touching the code.

5. **Code Duplication**  
   * Country/zone retrieval and population logic appear in `displayCustomer()`, `logonCustomer()`, and `validateShippingBilling()`. Extracting a helper method would reduce duplication.

6. **Legacy API Usage**  
   * `StringUtils.isBlank()` is fine, but modern code would use `String.isBlank()` (Java 11+).  
   * `log4j 1.x` is end‑of‑life; migrating to `log4j 2.x` or SLF4J is advisable.

7. **Single Responsibility Violation**  
   * The action mixes presentation preparation, business logic (order total calculation), and data persistence. Introducing separate services (e.g., `CustomerService`, `OrderService`, `PaymentService`) would improve testability and maintainability.

### Suggested Enhancements

| Area | Recommendation |
|------|----------------|
| **Validation** | Use Bean Validation (JSR‑303/380) with a DTO; move email matching logic into a single constraint. |
| **Generics** | Replace raw collections/maps with typed versions (`Map<Long, OrderProduct>`, `Collection<Zone>`). |
| **Refactoring** | Extract common zone/country lookup into a `ReferenceDataService`. |
| **Error Messages** | Correct field keys for repeat e‑mail errors and add an equality check. |
| **Logging** | Switch to SLF4J / log4j 2.x; include correlation IDs for traceability. |
| **Configuration** | Centralise shipping‑type determination in a `MerchantConfigurationService`. |
| **Testing** | Write unit tests for `orderHasShipping()`, `validateShippingBilling()` (mocking `SessionUtil` and services). |
| **Security** | Avoid storing raw passwords in the session; use hashed passwords and a proper authentication framework. |
| **Thread‑safety** | While not required in a per‑session context, declare fields as `final` where possible to signal immutability. |

Implementing these changes would reduce runtime errors, lower coupling, and open the code to modern Java best practices while keeping the core checkout workflow intact.

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

import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.checkout.util.RefUtil;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.payment.CreditCard;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.CustomerUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.www.SessionUtil;
import com.salesmanager.customer.profile.LogonAction;

public class CustomerInformationAction extends CheckoutBaseAction {

	private PaymentMethod paymentMethod;// submited

	private Collection<Zone> shippingZonesByCountry = new ArrayList();
	private Collection<Country> countries;
	private Collection<Zone> billingZonesByCountry = new ArrayList();
	private String shippingZone;
	private String billingZone;

	private String storeCountry;

	private String customerEmailAddressRepeat;

	private boolean hasCreditCardPayment = false;

	private String formstate;
	private String formstate2;

	private String shippingOriginCountry = null;// if shipping is domestic only

	private boolean hasShipping = false;

	private Customer customer;

	private Logger log = Logger.getLogger(CustomerInformationAction.class);

	private boolean useShippingInformation = false;

	private boolean orderHasShipping() {
		boolean isShipping = false;
		Map orderProducts = SessionUtil.getOrderProducts(getServletRequest());

		if (orderProducts != null) {
			Iterator i = orderProducts.keySet().iterator();
			while (i.hasNext()) {
				String line = (String) i.next();
				OrderProduct op = (OrderProduct) orderProducts.get(line);
				if (op.isShipping()) {
					isShipping = true;
				}
			}
		}
		return isShipping;
	}

	private void prepare() throws Exception {

		MerchantStore store = SessionUtil.getMerchantStore(getServletRequest());

		super.preparePayments();

		this.paymentMethod = SessionUtil.getPaymentMethod(getServletRequest());

		if (this.paymentMethod != null) {

			if (com.salesmanager.core.util.PaymentUtil
					.isPaymentModuleCreditCardType(this.getPaymentMethod()
							.getPaymentModuleName())) {
				hasCreditCardPayment = true;
				CreditCard cCard = (CreditCard) this.getPaymentMethod()
						.getConfig("CARD");
				if (cCard != null && super.getCreditCard() == null) {
					cCard.setCardNumber(null);
					this.setCreditCard(cCard);
				}
				getServletRequest().setAttribute("SELECTEDPAYMENT",
						paymentMethod);
			}
		}
		super.prepareCreditCards();
	}

	public String displayCustomer() {

		try {

			super.getServletRequest().setAttribute("STEP", 1);

			prepare();

			// Populate country and zone combo box
			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());

			// customer = SessionUtil.getLoggedInCustomer(getServletRequest());

			// if (customer == null) {

			customer = SessionUtil.getCustomer(getServletRequest());
			if (customer == null) {
				customer = new Customer();
				customer.setMerchantId(store.getMerchantId());
				customer.setLocale(getLocale());
				customer.setCustomerLang(getLocale().getLanguage());
				customer.setCustomerBillingCountryId(store.getCountry());// assign
				// country
				// to
				// store
				customer.setCustomerCountryId(store.getCountry());// assign
				// country
				// to
				// store
			}

			// }

			setCountries(RefUtil.getCountries(customer.getCustomerLang()));

			SessionUtil.setCustomer(customer, getServletRequest());

			Collection billZones = RefUtil.getZonesByCountry(customer
					.getCustomerBillingCountryId(), customer.getCustomerLang());
			Collection shipZones = RefUtil.getZonesByCountry(customer
					.getCustomerCountryId(), customer.getCustomerLang());

			// check if order has shipping
			hasShipping = orderHasShipping();

			// check where merchant ship
			ConfigurationRequest configRequest = new ConfigurationRequest(store
					.getMerchantId(),
					ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING);
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			ConfigurationResponse configResponse = mservice
					.getConfiguration(configRequest);

			if (configResponse != null) {
				MerchantConfiguration config = configResponse
						.getMerchantConfiguration(ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING);
				if (config != null) {
					String shippingType = config.getConfigurationValue();
					if (shippingType
							.equals(ShippingConstants.DOMESTIC_SHIPPING)) {
						Map cMap = RefCache.getAllcountriesmap(LanguageUtil
								.getLanguageNumberCode(super.getLocale()
										.getLanguage()));
						Country c = (Country) cMap.get(store.getCountry());
						if (c != null) {
							this.setShippingOriginCountry(c.getCountryName());
							customer.setCountryName(c.getCountryName());
							customer.setCustomerCountryId(c.getCountryId());
							shipZones = RefUtil.getZonesByCountry(customer
									.getCustomerCountryId(), customer
									.getCustomerLang());

						}
					}
				}
			}

			if (billZones != null && billZones.size() > 0) {
				setBillingZonesByCountry(billZones);
			} else {
				setBillingZone(customer.getBillingState());
			}

			if (shipZones != null && shipZones.size() > 0) {
				setShippingZonesByCountry(shipZones);
			} else {
				setShippingZone(customer.getCustomerState());
			}

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return "GENERICERROR";
		}

		return SUCCESS;

	}

	public String logonCustomer() {

		try {

			super.getServletRequest().setAttribute("STEP", 1);

			prepare();

			// check if order has shipping
			hasShipping = orderHasShipping();

			setCountries(RefUtil.getCountries(super.getLocale().getLanguage()));

			Collection billZones = RefUtil.getZonesByCountry(customer
					.getCustomerBillingCountryId(), customer.getCustomerLang());
			Collection shipZones = RefUtil.getZonesByCountry(customer
					.getCustomerCountryId(), customer.getCustomerLang());

			if (billZones != null && billZones.size() > 0) {
				setBillingZonesByCountry(billZones);
			} else {
				setBillingZone(customer.getBillingState());
			}

			if (shipZones != null && shipZones.size() > 0) {
				setShippingZonesByCountry(shipZones);
			} else {
				setShippingZone(customer.getCustomerState());
			}

			LogonAction logon = new LogonAction();

			customer = logon.authenticateCustomer(super.getServletRequest());

		} catch (ServiceException e) {

			super.setInputMessage("login.invalid");
			return INPUT;
		} catch (Exception ex) {
			log.error(ex);
			super.setTechnicalMessage();
			return INPUT;
		}

		return SUCCESS;

	}

	public String validateShippingBilling() {

		try {

			// Customer tmpCustomer = SessionUtil
			// .getLoggedInCustomer(getServletRequest());

			Customer tmpCustomer = SessionUtil.getCustomer(getServletRequest());

			// prepare combo objects

			setCountries(RefUtil.getCountries(super.getLocale().getLanguage()));

			Collection billZones = RefUtil.getZonesByCountry(customer
					.getCustomerBillingCountryId(), customer.getCustomerLang());
			Collection shipZones = RefUtil.getZonesByCountry(customer
					.getCustomerCountryId(), customer.getCustomerLang());

			if (billZones != null && billZones.size() > 0) {
				setBillingZonesByCountry(billZones);
			} else {
				setBillingZone(customer.getBillingState());
			}

			if (shipZones != null && shipZones.size() > 0) {
				setShippingZonesByCountry(shipZones);
			} else {
				setShippingZone(customer.getCustomerState());
			}

			prepare();

			if (StringUtils.isBlank(customer.getCustomerEmailAddress())) {
				addFieldError("customer.customerEmailAddress",
						getText("messages.required.email"));
				super.addFieldMessage("customer.customerEmailAddress",
						"messages.required.email");
			} else {
				if (!CustomerUtil.validateEmail(customer
						.getCustomerEmailAddress())) {
					addFieldError("customer.customerEmailAddress",
							getText("messages.invalid.email"));
					super.addFieldMessage("customer.customerEmailAddress",
							"messages.invalid.email");
				}
			}
			if (StringUtils.isBlank(customerEmailAddressRepeat)) {
				addFieldError("customer.customerEmailAddressRepeat",
						getText("messages.required.repeatemail"));
				super.addFieldMessage("customer.customerEmailAddressRepeat",
						"messages.required.repeatemail");
			} else {
				if (!CustomerUtil.validateEmail(customerEmailAddressRepeat)) {
					addFieldError("customer.customerEmailAddress",
							getText("messages.invalid.email"));
					super.addFieldMessage("customer.customerEmailAddress",
							"messages.invalid.email");
				}
			}

			hasShipping = orderHasShipping();

			// if order has shipping
			if (hasShipping) {

				if (StringUtils.isBlank(customer.getCustomerFirstname())) {
					addFieldError("customer.customerFirstname",
							getText("messages.required.shippingfirstname"));
					super.addFieldMessage("customer.customerFirstname",
							"messages.required.shippingfirstname");
				}
				if (StringUtils.isBlank(customer.getCustomerLastname())) {
					addFieldError("customer.customerLastname",
							getText("messages.required.shippinglastname"));
					super.addFieldMessage("customer.customerLastname",
							"messages.required.shippinglastname");
				}
				if (StringUtils.isBlank(customer.getCustomerStreetAddress())) {
					addFieldError("customer.customerStreetAddress",
							getText("messages.required.shippingstreetaddress"));
					super.addFieldMessage("customer.customerStreetAddress",
							"messages.required.shippingstreetaddress");
				}
				if (StringUtils.isBlank(customer.getCustomerCity())) {
					addFieldError("customer.customerCity",
							getText("messages.required.shippingcity"));
					super.addFieldMessage("customer.customerCity",
							"messages.required.shippingcity");
				}
				if (StringUtils.isBlank(customer.getCustomerPostalCode())) {
					addFieldError("customer.customerPostalCode",
							getText("messages.required.shippingstreetaddress"));
					super.addFieldMessage("customer.customerPostalCode",
							"messages.required.shippingstreetaddress");
				}
				if (!StringUtils.isBlank(this.getFormstate2())
						&& this.getFormstate2().equals("text")) {
					if (StringUtils.isBlank(customer.getCustomerState())) {
						addFieldError(
								"customer.customerState",
								getText("messages.required.shippingstateprovince"));
						super.addFieldMessage("customer.customerState",
								"messages.required.shippingstateprovince");
					}
				}
				if (StringUtils.isBlank(customer.getCustomerTelephone())) {
					addFieldError("customer.customerTelephone",
							getText("messages.required.shippingphone"));
					super.addFieldMessage("customer.customerTelephone",
							"messages.required.shippingphone");
				}

				String cName = "";
				Map lcountries = RefCache.getCountriesMap();
				if (lcountries != null) {
					Country country = (Country) lcountries.get(customer
							.getCustomerCountryId());
					Set descriptions = country.getDescriptions();
					if (descriptions != null) {
						Iterator cIterator = descriptions.iterator();
						while (cIterator.hasNext()) {
							CountryDescription desc = (CountryDescription) cIterator
									.next();
							cName = desc.getCountryName();
							if (desc.getId().getLanguageId() == LanguageUtil
									.getLanguageNumberCode(super.getLocale()
											.getLanguage())) {
								cName = desc.getCountryName();
								customer.setCountryName(cName);
								break;
							}
						}
					}
				}

				if (StringUtils.isBlank(customer.getCustomerState())) {
					Map lzones = RefCache.getAllZonesmap(LanguageUtil
							.getLanguageNumberCode(super.getLocale()
									.getLanguage()));
					if (lzones != null) {
						Zone z = (Zone) lzones
								.get(customer.getCustomerZoneId());
						if (z != null) {
							customer.setCustomerState(z.getZoneName());
						}
					}
				}

			}

			// if flag shipping = billing
			if (useShippingInformation) {

				customer.setCustomerBillingCountryName(customer
						.getCountryName());
				customer.setCustomerBillingCity(customer.getCustomerCity());
				customer.setCustomerBillingCountryId(customer
						.getCustomerCountryId());
				customer.setCustomerBillingFirstName(customer
						.getCustomerFirstname());
				customer.setCustomerBillingLastName(customer
						.getCustomerLastname());
				customer.setCustomerBillingPostalCode(customer
						.getCustomerPostalCode());
				customer.setCustomerBillingState(customer.getCustomerState());
				customer.setCustomerBillingStreetAddress(customer
						.getCustomerStreetAddress());
				customer.setCustomerBillingZoneId(customer.getCustomerZoneId());

				super.getServletRequest().getSession().setAttribute(
						"useShippingInformation",
						this.isUseShippingInformation());

			} else {

				// billing
				if (StringUtils.isBlank(customer.getCustomerBillingFirstName())) {
					addFieldError("customer.customerBillingFirstName",
							getText("messages.required.billingfirstname"));
					super.addFieldMessage("customer.customerBillingFirstName",
							"messages.required.billingfirstname");
				}
				if (StringUtils.isBlank(customer.getCustomerBillingLastName())) {
					addFieldError("customer.customerBillingLastName",
							getText("messages.required.billinglastname"));
					super.addFieldMessage("customer.customerBillingLastName",
							"messages.required.billinglastname");
				}
				if (StringUtils.isBlank(customer
						.getCustomerBillingStreetAddress())) {
					addFieldError("customer.customerBillingStreetAddress",
							getText("messages.required.billingstreetaddress"));
					super.addFieldMessage(
							"customer.customerBillingStreetAddress",
							"messages.required.billingstreetaddress");
				}
				if (StringUtils.isBlank(customer.getCustomerBillingCity())) {
					addFieldError("customer.customerBillingCity",
							getText("messages.required.billingcity"));
					super.addFieldMessage("customer.customerBillingCity",
							"messages.required.billingcity");
				}
				if (!StringUtils.isBlank(this.getFormstate())
						&& this.getFormstate().equals("text")) {
					if (StringUtils.isBlank(customer.getCustomerBillingState())) {
						addFieldError(
								"customer.customerBillingState",
								getText("messages.required.billingstateprovince"));
						super.addFieldMessage("customer.customerBillingState",
								"messages.required.billingstateprovince");
					}
				}
				if (StringUtils
						.isBlank(customer.getCustomerBillingPostalCode())) {
					addFieldError("customer.customerBillingPostalCode",
							getText("messages.required.billingpostalcode"));
					super.addFieldMessage("customer.customerBillingPostalCode",
							"messages.required.billingpostalcode");
				}

				String cName = "";
				Map lcountries = RefCache.getCountriesMap();
				if (lcountries != null) {
					Country country = (Country) lcountries.get(customer
							.getCustomerBillingCountryId());
					Set descriptions = country.getDescriptions();
					if (descriptions != null) {
						Iterator cIterator = descriptions.iterator();
						while (cIterator.hasNext()) {
							CountryDescription desc = (CountryDescription) cIterator
									.next();
							cName = desc.getCountryName();
							if (desc.getId().getLanguageId() == LanguageUtil
									.getLanguageNumberCode(super.getLocale()
											.getLanguage())) {
								cName = desc.getCountryName();
								customer.setCustomerBillingCountryName(cName);
								break;
							}
						}
					}
				}

				if (StringUtils.isBlank(customer.getCustomerBillingState())) {
					Map lzones = RefCache.getAllZonesmap(LanguageUtil
							.getLanguageNumberCode(super.getLocale()
									.getLanguage()));
					if (lzones != null) {
						Zone z = (Zone) lzones.get(customer
								.getCustomerBillingZoneId());
						if (z != null) {
							customer.setCustomerBillingState(z.getZoneName());
						}
					}
				}

			}

			if (this.getPaymentMethod() != null) {

				if (com.salesmanager.core.util.PaymentUtil
						.isPaymentModuleCreditCardType(this.getPaymentMethod()
								.getPaymentModuleName())) {

					MerchantStore store = SessionUtil
							.getMerchantStore(getServletRequest());
					super.validateCreditCard(this.getPaymentMethod(), store
							.getMerchantId());

				}

				this.preparePayments();
				Map pms = super.getPaymentMethods();
				PaymentMethod tmpMethod = (PaymentMethod) pms.get(this
						.getPaymentMethod().getPaymentModuleName());
				if (tmpMethod != null) {
					this.getPaymentMethod().setPaymentMethodName(
							tmpMethod.getPaymentMethodName());
					this.getPaymentMethod().setPaymentModuleText(
							tmpMethod.getPaymentModuleText());
				}

				SessionUtil.setPaymentMethod(this.getPaymentMethod(),
						getServletRequest());

			}

			if (getFieldErrors().size() > 0) {
				return INPUT;
			}

			if (tmpCustomer != null) {
				customer
						.setCustomerAnonymous(tmpCustomer.isCustomerAnonymous());
				customer.setCustomerNick(tmpCustomer.getCustomerNick());
				customer.setCustomerPassword(tmpCustomer.getCustomerPassword());
				customer.setCustomerAuthorization(tmpCustomer
						.getCustomerAuthorization());
				customer.setCustomerLang(tmpCustomer.getCustomerLang());
				customer.setCustomerReferral(tmpCustomer.getCustomerReferral());
				customer.setCustomerNewsletter(tmpCustomer
						.getCustomerNewsletter());
				customer.setCustomerId(tmpCustomer.getCustomerId());
				customer.setCustomerGroupPricing(tmpCustomer
						.getCustomerGroupPricing());
				customer.setMerchantId(tmpCustomer.getMerchantId());

				if (!hasShipping) {

					customer.setCountryName(tmpCustomer.getCountryName());
					customer.setCustomerCity(tmpCustomer.getCustomerCity());
					customer.setCustomerCountryId(tmpCustomer
							.getCustomerCountryId());
					customer.setCustomerFirstname(tmpCustomer
							.getCustomerFirstname());
					customer.setCustomerLastname(tmpCustomer
							.getCustomerLastname());
					customer.setCustomerPostalCode(tmpCustomer
							.getCustomerPostalCode());
					customer.setCustomerState(tmpCustomer.getCustomerState());
					customer.setCustomerStreetAddress(tmpCustomer
							.getCustomerStreetAddress());
					customer.setCustomerZoneId(tmpCustomer.getCustomerZoneId());

				}
			}

			customer.setCustomerLang(super.getLocale().getLanguage());

			// SessionUtil.setCustomer(customer, getServletRequest());
			SessionUtil.setPaymentMethod(this.getPaymentMethod(),
					getServletRequest());

			Order order = SessionUtil.getOrder(getServletRequest());
			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());

			Map orderProducts = SessionUtil
					.getOrderProducts(getServletRequest());
			List products = new ArrayList();
			if (orderProducts != null) {
				Iterator i = orderProducts.keySet().iterator();
				while (i.hasNext()) {
					String line = (String) i.next();
					OrderProduct op = (OrderProduct) orderProducts.get(line);
					products.add(op);
				}
			}

			// update order with tax if it applies
			super.updateOrderTotal(order, products, customer, store);

			SessionUtil.setCustomer(customer, super.getServletRequest());

			// if (tmpCustomer != null) {
			// SessionUtil.setLoggedInCustomer(getServletRequest(), customer);
			// }

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}

		if (SessionUtil.getIsShipping(getServletRequest())) {
			return SUCCESS;
		} else {
			return "SUCCESS-SUMMARY";
		}

	}

	public String getStoreCountry() {
		return storeCountry;
	}

	public void setStoreCountry(String storeCountry) {
		this.storeCountry = storeCountry;
	}

	public String getFormstate() {
		return formstate;
	}

	public void setFormstate(String formstate) {
		this.formstate = formstate;
	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

	public boolean isHasShipping() {
		return hasShipping;
	}

	public void setHasShipping(boolean hasShipping) {
		this.hasShipping = hasShipping;
	}

	public Collection<Zone> getShippingZonesByCountry() {
		return shippingZonesByCountry;
	}

	public void setShippingZonesByCountry(
			Collection<Zone> shippingZonesByCountry) {
		this.shippingZonesByCountry = shippingZonesByCountry;
	}

	public Collection<Zone> getBillingZonesByCountry() {
		return billingZonesByCountry;
	}

	public void setBillingZonesByCountry(Collection<Zone> billingZonesByCountry) {
		this.billingZonesByCountry = billingZonesByCountry;
	}

	public String getBillingZone() {
		return billingZone;
	}

	public void setBillingZone(String billingZone) {
		this.billingZone = billingZone;
	}

	public String getShippingZone() {
		return shippingZone;
	}

	public void setShippingZone(String shippingZone) {
		this.shippingZone = shippingZone;
	}

	public Collection<Country> getCountries() {
		return countries;
	}

	public void setCountries(Collection<Country> countries) {
		this.countries = countries;
	}

	public String getShippingOriginCountry() {
		return shippingOriginCountry;
	}

	public void setShippingOriginCountry(String shippingOriginCountry) {
		this.shippingOriginCountry = shippingOriginCountry;
	}

	public String getFormstate2() {
		return formstate2;
	}

	public void setFormstate2(String formstate2) {
		this.formstate2 = formstate2;
	}

	public boolean isUseShippingInformation() {
		return useShippingInformation;
	}

	public void setUseShippingInformation(boolean useShippingInformation) {
		this.useShippingInformation = useShippingInformation;
	}

	public PaymentMethod getPaymentMethod() {
		return paymentMethod;
	}

	public void setPaymentMethod(PaymentMethod paymentMethod) {
		this.paymentMethod = paymentMethod;
	}

	public boolean isHasCreditCardPayment() {
		return hasCreditCardPayment;
	}

	public void setHasCreditCardPayment(boolean hasCreditCardPayment) {
		this.hasCreditCardPayment = hasCreditCardPayment;
	}

	public String getCustomerEmailAddressRepeat() {
		return customerEmailAddressRepeat;
	}

	public void setCustomerEmailAddressRepeat(String customerEmailAddressRepeat) {
		this.customerEmailAddressRepeat = customerEmailAddressRepeat;
	}

}



```
