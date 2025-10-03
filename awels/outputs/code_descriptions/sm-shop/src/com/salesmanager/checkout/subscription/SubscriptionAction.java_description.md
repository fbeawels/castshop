# SubscriptionAction.java

## Review

## 1. Summary  

**Purpose** – `SubscriptionAction` is a Struts‑2 action that drives the subscription workflow for a merchant store.  
It allows a customer to (1) add a single product to a provisional “order”, (2) fill in billing information, (3) choose a payment method, and (4) move to a subscription‑summary page.  

**Key components**

| Component | Role |
|-----------|------|
| `CheckoutParams value` | Holds the parameters sent from the front‑end (merchantId, productId, language, etc.). |
| `MerchantStore store`, `Customer customer`, `PaymentMethod paymentMethod` | Core entities used during the flow. |
| `SessionUtil` | Central utility for persisting order, customer, payment data in the HTTP session. |
| `ServiceFactory` | Provides singleton services (`CatalogService`, `MerchantService`, `OrderService`). |
| `RefCache`, `RefUtil` | Provide cached reference data (countries, zones, credit cards). |
| `PaymentUtil` | Determines supported payment modules. |
| `CheckoutBaseAction` | Base class that provides utilities such as `addActionError`, `addFieldError`, `setLocale`, etc. |

**Frameworks / libraries**

* **Struts‑2** – the action implements `ModelDriven<CheckoutParams>` and uses the Struts action lifecycle (`SUCCESS`, `INPUT`, `GLOBALERROR`).  
* **Apache Commons** – `StringUtils`, `Configuration`.  
* **Log4j** – `Logger`.  
* **SalesManager Core** – a collection of domain entities, services and utilities.

---

## 2. Detailed Description  

### Flow of execution  

1. **Entering the subscription flow**  
   * `addSubscriptionItem()` – first invoked when the customer posts the product to subscribe.  
   * Calls `validateAddSubscription()` to verify the request (merchant/product existence, stock, etc.).  
   * If validation passes, `addItem()` constructs a provisional `Order`, adds an `OrderProduct` (with optional attributes), and stores the order and its totals in the session.  
   * The method finishes with a `SUCCESS` result, which forwards to the subscription form.

2. **Displaying the subscription form**  
   * `displaySubscriptionForm()` checks that an order exists in the session (session timeout handling), prepares zones/countries and payment methods, then forwards to the form view (`SUCCESS`).  

3. **Submitting the form**  
   * `subscribe()` is called when the customer submits billing information and selects a payment method.  
   * It calls `preparePage()` (loads payment methods & credit cards), validates the customer data (`validateCustomer()`), checks that a payment method was chosen, validates credit‑card details if required, then stores the payment method in the session.  
   * If everything succeeds, the action returns `SUCCESS`, which normally goes to the subscription‑summary page.

### Core responsibilities

| Responsibility | Where it lives |
|-----------------|----------------|
| Validation of request parameters | `validateAddSubscription()` |
| Building a provisional order | `addItem()` |
| Preparing reference data (countries, zones, payment methods) | `preparePage()`, `prepareZones()`, `displaySubscriptionForm()` |
| Customer data validation | `validateCustomer()` |
| Payment method loading & credit‑card validation | `preparePage()`, `subscribe()` |

### Design decisions  

* **Session‑centric state** – all intermediate data (order, customer, payment method) is stored in the HTTP session via `SessionUtil`.  
* **Service lookup via static factory** – the class uses `ServiceFactory` to pull singletons instead of dependency injection.  
* **Caching** – heavy use of `RefCache`/`RefUtil` to avoid repeated DB calls.  
* **ModelDriven** – the entire request is wrapped in `CheckoutParams` and automatically exposed to the view layer.  

### Assumptions / Preconditions  

* The user already has an active HTTP session containing an `Order`.  
* `CheckoutBaseAction` correctly implements Struts’ error‑handling helpers.  
* The request parameters (`merchantId`, `productId`, `lang`, etc.) are correctly populated by the framework.  

---

## 3. Functions / Methods  

| Method | Purpose | Highlights | Issues / Suggested improvements |
|--------|---------|------------|---------------------------------|
| `preparePage()` | Loads supported payment methods (`paymentMethods`) and cached credit cards (`creditCards`). | Uses raw `Map` and `Collection`; no generics. | • Replace raw types with typed `Map<String, PaymentMethod>` and `Collection<CreditCard>`.<br>• Move this into a dedicated helper class – it is called by both `subscribe()` and `addItem()`. |
| `getModel()` | Returns the `CheckoutParams` model for Struts‑2. | Straightforward. | None. |
| `subscribe()` | Handles final form submission. | • Calls `validateCustomer()` and performs credit‑card validation for selected modules.<br>• Persists customer & payment data in the session. | • The method is long – split into smaller private helpers (`validateBillingInfo()`, `checkPaymentSelection()`).<br>• Catch specific exceptions (e.g., `InvalidPaymentDetailsException`) rather than a generic `Exception`. |
| `validateCustomer()` | Performs a large number of checks on `Customer` fields (email, name, address, etc.). | Uses `StringUtils.isBlank()` and a heavy amount of manual string handling. | • Duplicate logic with the “STUB CUSTOMER” block in `displaySubscriptionForm()` – consider a helper. <br>• Magic numbers (e.g., `OUT_OF_STOCK_PRODUCT_QUANTITY`, `LOW_STOCK_PRODUCT_QUANTITY`) should be constants. |
| `addItem()` | Builds a provisional `Order`, adds an `OrderProduct` (including optional attributes), calculates totals, and sets the locale. | Very involved; many lines of raw‑type manipulation. | • Raw `Map` and `Collection` usage makes type safety impossible. <br>• Repeated code for zones/countries between `addItem()` and `prepareZones()` – centralise. <br>• Hard‑coded product quantity checks – refactor into a service. |
| `prepareZones()` | Determines which countries & zones are available based on the current product or existing customer. | Handles several branches; populates session data. | • Complex branching logic – hard to test. <br>• Uses raw `Collection` and `Configuration` without type safety. |
| `displaySubscriptionForm()` | Shows the initial subscription form. | Performs a single session‑check, sets a progress‑bar attribute, and calls `preparePage()`. | • The “STUB CUSTOMER” comment block suggests a debugging stub that should be removed. |
| `addSubscriptionItem()` | Main entry point – called when adding a product to the subscription list. | Delegates to `validateAddSubscription()` and `addItem()`. | • No CSRF protection shown; should be added if not handled elsewhere. |
| `validateAddSubscription()` | Validates that the request contains the required parameters and that the product is available. | Does many DB look‑ups, stock checks, and even sends “out‑of‑stock” emails. | • Email logic should be delegated to a dedicated service. <br>• Many repeated blocks for “out‑of‑stock” and “low‑stock” handling – factor out. <br>• Throws `Exception` for a missing `MerchantUserInformation` – this should be a checked exception or handled differently. |
| Getters/Setters | Standard JavaBean accessors for action properties. | Most of them are boilerplate; a few expose raw collections (`creditCards`, `getCountries()`). | • Prefer typed generics for all collection properties. <br>• Consider using immutable DTOs for the view layer. |

---

## 3. Functions / Methods (Detailed)  

| Method | Signature | Responsibilities | Observations |
|--------|-----------|------------------|--------------|
| `preparePage()` | `private void` | Loads supported payment modules and cached credit cards into instance variables. | Uses raw `Map`/`Collection`; no error handling. |
| `getModel()` | `public CheckoutParams` | Required by `ModelDriven`. | Straight‑forward. |
| `subscribe()` | `public String` | Validates billing data, checks payment selection, validates card details, stores data in session. | Long method; mixes UI logic with business validation. |
| `validateCustomer()` | `private void` | Checks each customer field; sets country/currency values. | No separation of validation concerns; uses hard‑coded date logic. |
| `addItem()` | `public void` | Creates provisional `Order`, adds `OrderProduct`, calculates totals, sets locale. | Heavy use of raw types; duplicate code for zone/country handling; possible `NullPointerException` if `customer` is null. |
| `prepareZones()` | `private void` | Populates `countries`, `zonesByCountry` or `zone` based on `customer` and request language. | Complex branching; repeated logic in two branches. |
| `displaySubscriptionForm()` | `public String` | Shows the subscription form page after verifying session order. | Handles session expiry; prepares progress bar. |
| `addSubscriptionItem()` | `public String` | Entry point for adding a product to a subscription. | Calls `validateAddSubscription()` and `addItem()`. |
| `validateAddSubscription()` | `public boolean` | Validates merchantId, productId, product availability, stock levels, and sends out‑of‑stock emails if necessary. | Long, complex method; contains repeated blocks for out‑of‑stock / low‑stock handling; mixes service usage, email logic, and validation. |
| `getPaymentMethods()` / `setPaymentMethods(Map)` | getters / setters | Expose payment methods map. | Use raw `Map`. |
| `getCountries()` / `setCountries(Collection<Country>)` | getters / setters | Expose country list. | Type‑safe. |
| `getZonesByCountry()` / `setZonesByCountry(Collection<Zone>)` | getters / setters | Expose zone list. | Type‑safe. |
| `getCreditCards()` / `setCreditCards(Collection)` | getters / setters | Expose cached credit cards. | Raw type. |
| `getPaymentMethod()` / `setPaymentMethod(PaymentMethod)` | getters / setters | Expose selected payment method. | Type‑safe. |
| `getCustomer()` / `setCustomer(Customer)` | getters / setters | Expose customer entity. | Type‑safe. |
| `getConfirmEmailAddress()`, `getConfirmPassword()`, `getZone()`, `getFormstate()`, etc. | Standard accessors | Provide form data to the view. | Raw types used for some collections (`getCreditCards()`). |

---

## 3. Dependencies  

| External library / module | Purpose in this class |
|---------------------------|-----------------------|
| **Struts‑2** (`com.opensymphony.xwork2.ModelDriven`) | Provides the action lifecycle and model binding. |
| **Apache Commons Lang** (`StringUtils`) | Simplifies string checks. |
| **Apache Commons Configuration** (`Configuration`) | Loads system properties (default country, language). |
| **Log4j** (`Logger`) | Logging of errors and debug info. |
| **SalesManager Core** (various `com.salesmanager.core.*`) | Domain entities (`Customer`, `Order`, `MerchantStore`, etc.), services (`CatalogService`, `MerchantService`, `OrderService`, `CommonService`), and utilities (`RefCache`, `RefUtil`, `SessionUtil`, `PaymentUtil`, `LabelUtil`, etc.). |
| **Java EE / Servlet API** (implicitly via `HttpServletRequest`) | HTTP session handling. |

---

## 4. Additional Notes (Strengths & Weaknesses)  

### Strengths  

1. **Clear separation of responsibilities** – the action coordinates services, session storage, and view rendering while delegating business rules to dedicated services (`CatalogService`, `MerchantService`, `OrderService`).  
2. **Use of Struts‑2 conventions** – `ModelDriven<CheckoutParams>` and result strings (`SUCCESS`, `INPUT`, `GLOBALERROR`) make the action easy to wire in a Struts configuration.  
3. **Caching strategy** – `RefCache`/`RefUtil` reduce database hits for reference data.  

### Weaknesses & Areas for Improvement  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (`Map`, `Collection`) in many fields and methods | Compile‑time type safety is lost, making bugs harder to detect. | Replace all raw collections with generic versions, e.g. `Map<String, PaymentMethod>` or `Collection<CreditCard>`. |
| **Magic numbers** (`OUT_OF_STOCK_PRODUCT_QUANTITY`, `LOW_STOCK_PRODUCT_QUANTITY`) | Hard to understand and change. | Define constants in a dedicated `StockConstants` class and use descriptive names. |
| **Duplicate country/zone logic** | Maintenance overhead and higher chance of bugs when changing logic. | Extract the two branches of `prepareZones()` into helper methods (`loadZonesForNewCustomer()` / `loadZonesForExistingCustomer()`). |
| **Large monolithic methods** (`addItem()`, `validateAddSubscription()`) | Hard to unit‑test, violates Single Responsibility Principle. | Move business logic to a separate service (`SubscriptionService`) and keep the action thin. |
| **Narrow exception handling** (`catch (Exception e)`) | Swallows all errors, making it hard to distinguish between technical failures and business rule violations. | Catch specific exceptions (`ProductNotFoundException`, `InsufficientStockException`, `PaymentValidationException`) and provide meaningful feedback. |
| **Potential `NullPointerException`s** – e.g. `customer` may be `null` in `validateCustomer()` or `subscribe()`. | Runtime crashes if the session is corrupted. | Perform defensive null checks and use early‑return patterns. |
| **Hard‑coded email templates** (`email_template_outofstock.ftl`, `email_template_lowstock.ftl`) | Tightly couples the action to specific email layouts. | Externalise template names to properties or use a dedicated email service. |
| **Logging** – uses `log.error(e)` without context for many failures. | Hard to trace root causes. | Add contextual messages (`log.error("Failed to validate product id=" + value.getProductId(), e)`). |
| **No CSRF protection shown** | If the rest of the stack does not provide it, the subscription flow is vulnerable. | Ensure tokens are validated or rely on Struts‑2 `@SkipValidation`/`Token` mechanisms. |
| **Session handling** – relies on manual checks (`SessionUtil.getOrder()`) and manual `GLOBALERROR` forwarding. | Could be replaced with a global interceptor that validates session presence. | Create a `SessionValidatingInterceptor`. |
| **Hardcoded language logic** (`value.setLangId(LanguageUtil.getLanguageNumberCode(value.getLang()))`). | Repeating language conversion logic. | Provide a utility method `setLanguageFromParams()` that handles conversion and validation. |
| **Email sending inside the action** – side‑effects in a Struts action. | Makes testing harder and violates separation of concerns. | Delegate email notifications to a `NotificationService`. |

### Recommendations for Future Refactor

1. **Introduce Dependency Injection** – replace `ServiceFactory.getXxx()` with constructor or field injection (Spring, Guice).  
2. **Thin Action Layer** – keep the action focused on request/response mapping; push validation, order construction, and notification logic into dedicated services.  
3. **Use Validation Frameworks** – Struts‑2 `Validator` XML or annotations (`@Valid`) for customer fields instead of hand‑rolled `validateCustomer()`.  
4. **Consistent Typing** – remove all raw types, adopt generics everywhere.  
5. **Centralized Error Codes** – create a `SubscriptionErrorCodes` enum or constants class instead of hard‑coding message keys in multiple places.  
6. **Unit‑Testable Design** – extract zone/country loading into a `ZoneService`, email notification into an `EmailService`, and pass those services into the action.  

---  

### Bottom‑Line

`SubscriptionAction` accomplishes its goal but is a heavy, procedural action that intertwines UI, business logic, and data persistence. Refactoring the code into smaller, testable services and cleaning up the type usage will dramatically improve maintainability, testability, and robustness.

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
package com.salesmanager.checkout.subscription;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import javax.servlet.http.HttpSession;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.ModelDriven;
import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.checkout.CheckoutParams;
import com.salesmanager.checkout.util.PaymentUtil;
import com.salesmanager.checkout.util.RefUtil;
import com.salesmanager.checkout.web.Constants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttribute;
import com.salesmanager.core.entity.catalog.ProductDescription;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductAttribute;
import com.salesmanager.core.entity.orders.OrderTotal;
import com.salesmanager.core.entity.orders.OrderTotalSummary;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.reference.CentralCreditCard;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.CustomerUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.LogMerchantUtil;
import com.salesmanager.core.util.OrderUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class SubscriptionAction extends CheckoutBaseAction implements
		ModelDriven<CheckoutParams>, Constants {
	private static final long serialVersionUID = 1L;
	private Logger log = Logger.getLogger(SubscriptionAction.class);

	private Map paymentMethods;
	private Collection creditCards;
	private CheckoutParams value = new CheckoutParams();

	// combo box
	private Collection<Country> countries;
	private Collection<Zone> zonesByCountry = new ArrayList();
	private String zone;

	private String storeCountry;
	private String billingState;
	// private int selectedCountryId;

	private CatalogService cservice = (CatalogService) ServiceFactory
			.getService(ServiceFactory.CatalogService);
	private MerchantService mservice = (MerchantService) ServiceFactory
			.getService(ServiceFactory.MerchantService);
	private OrderService oservice = (OrderService) ServiceFactory
			.getService(ServiceFactory.OrderService);

	MerchantStore store = null;

	private Customer customer;// submited
	private String confirmEmailAddress;
	private String confirmPassword;
	private String customerBillingStreetAddress1;
	private String customerBillingStreetAddress2;

	private PaymentMethod paymentMethod;// submited

	private String formstate;

	private void preparePage() throws Exception {

		Map ccmap = com.salesmanager.core.service.cache.RefCache
				.getSupportedCreditCards();
		if (ccmap != null) {
			creditCards = new ArrayList();
			Iterator i = ccmap.keySet().iterator();

			while (i.hasNext()) {
				int key = (Integer) i.next();
				creditCards.add((CentralCreditCard) ccmap.get(key));

			}
		}

		paymentMethods = PaymentUtil.getPaymentMethods(1, super.getLocale());

		// too complex to be handled with webwork, will store the object in http
		super.getServletRequest().setAttribute("PAYMENTS", paymentMethods);

	}

	public CheckoutParams getModel() {
		return value;
	}

	// goes to summary page
	public String subscribe() {

		try {

			preparePage();
			validateCustomer();

			SessionUtil.setCustomer(customer, getServletRequest());

			prepareZones();

			if (this.getPaymentMethod() == null
					|| this.getPaymentMethod().getPaymentModuleName() == null) {
				super.addActionError("error.nopaymentmethod");
				return INPUT;
			}

			super.getServletRequest().setAttribute("SELECTEDPAYMENT",
					this.getPaymentMethod());

			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());

			// check if payment method is credit card type
			if (com.salesmanager.core.util.PaymentUtil
					.isPaymentModuleCreditCardType(this.getPaymentMethod()
							.getPaymentModuleName())) {
				super.validateCreditCard(this.getPaymentMethod(), store
						.getMerchantId());
			} else {
				super.setCreditCard(null);// reset credit card information
			}

			if (getFieldErrors().size() > 0) {
				return INPUT;
			}

			SessionUtil.setPaymentMethod(this.getPaymentMethod(),
					getServletRequest());

		} catch (Exception e) {
			log.error(e);
			super.addActionError(getText("error.process.notransaction"));
			return "GLOBALERROR";
		}

		return SUCCESS;
	}

	public void validateCustomer() {

		if (StringUtils.isBlank(customer.getCustomerEmailAddress())) {
			addFieldError("customer.customerEmailAddress",
					getText("messages.required.email"));
			super.addFieldMessage("customer.customerEmailAddress",
					"messages.required.email");
		} else {
			if (!CustomerUtil.validateEmail(customer.getCustomerEmailAddress())) {
				addFieldError("customer.customerEmailAddress",
						getText("messages.invalid.email"));
				super.addFieldMessage("customer.customerEmailAddress",
						"messages.invalid.email");
			}
		}
		/*
		 * if(StringUtils.isBlank(customer.getCustomerPassword())) {
		 * addFieldError("customer.customerPassword",
		 * getText("messages.required.password")); }
		 * if(StringUtils.isBlank(getConfirmEmailAddress())) {
		 * addFieldError("confirmEmailAddress",
		 * getText("messages.required.email.confirm")); }else{
		 * if(!getConfirmEmailAddress
		 * ().equals(customer.getCustomerEmailAddress())){
		 * addFieldError("confirmEmailAddress",
		 * getText("messages.unequal.email.confirm")); } }
		 * if(StringUtils.isBlank(getConfirmPassword())) {
		 * addFieldError("confirmPassword",
		 * getText("messages.required.password.confirm")); }else{
		 * if(!getConfirmPassword().equals(customer.getCustomerPassword())){
		 * addFieldError("confirmPassword",
		 * getText("messages.unequal.password.confirm")); } }
		 */
		if (StringUtils.isBlank(customer.getCustomerFirstname())) {
			addFieldError("customer.customerFirstname",
					getText("messages.required.firstname"));
			super.addFieldMessage("customer.customerFirstname",
					"messages.required.firstname");
		}
		if (StringUtils.isBlank(customer.getCustomerLastname())) {
			addFieldError("customer.customerLastname",
					getText("messages.required.lastname"));
			super.addFieldMessage("customer.customerLastname",
					"messages.required.lastname");
		}
		if (StringUtils.isBlank(customer.getCustomerBillingStreetAddress())) {
			addFieldError("customer.customerBillingStreetAddress",
					getText("messages.required.streetaddress"));
			super.addFieldMessage("customer.customerBillingStreetAddress",
					"messages.required.streetaddress");
		}
		if (StringUtils.isBlank(customer.getCustomerBillingCity())) {
			addFieldError("customer.customerBillingCity",
					getText("messages.required.city"));
			super.addFieldMessage("customer.customerBillingCity",
					"messages.required.city");
		}
		if (!StringUtils.isBlank(this.getFormstate())
				&& this.getFormstate().equals("text")) {
			if (StringUtils.isBlank(customer.getCustomerBillingState())) {
				addFieldError("customer.customerBillingState",
						getText("messages.required.stateprovince"));
				super.addFieldMessage("customer.customerBillingState",
						"messages.required.stateprovince");
			}
		}
		if (StringUtils.isBlank(customer.getCustomerBillingPostalCode())) {
			addFieldError("customer.customerBillingPostalCode",
					getText("messages.required.postalcode"));
			super.addFieldMessage("customer.customerBillingPostalCode",
					"messages.required.postalcode");
		}

		if (StringUtils.isBlank(customer.getCustomerTelephone())) {
			addFieldError("customer.customerTelephone",
					getText("messages.required.phone"));
			super.addFieldMessage("customer.customerTelephone",
					"messages.required.phone");
		}
		/**
		 * else
		 * if(!CustomerUtil.ValidatePhoneNumber(customer.getCustomerTelephone
		 * ())){ addFieldError("customer.customerTelephone",
		 * getText("messages.invalid.phone"));
		 * super.addFieldMessage("customer.customerTelephone",
		 * "messages.invalid.phone"); }
		 **/

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
						break;
					}
				}
			}
		}

		if (StringUtils.isBlank(customer.getCustomerBillingState())) {
			Map lzones = RefCache.getAllZonesmap(LanguageUtil
					.getLanguageNumberCode(super.getLocale().getLanguage()));
			if (lzones != null) {
				Zone z = (Zone) lzones.get(customer.getCustomerBillingZoneId());
				if (z != null) {
					customer.setCustomerBillingState(z.getZoneName());
					customer.setCustomerState(z.getZoneName());
				}
			}
		}

		String lang = super.getLocale().getLanguage();

		customer.setCountryName(cName);
		customer.setCustomerBillingCountryName(cName);
		customer.setCustomerLang(lang);

		customer.setCountryName(customer.getBillingCountry());
		customer.setCustomerCity(customer.getCustomerBillingCity());
		customer.setCustomerCountryId(customer.getCustomerBillingCountryId());
		customer.setCustomerLang(super.getLocale().getLanguage());
		customer.setCustomerPostalCode(customer.getCustomerBillingPostalCode());
		customer.setCustomerStreetAddress(customer
				.getCustomerBillingStreetAddress());
		customer.setCustomerState(customer.getBillingState());
		customer.setCustomerZoneId(customer.getCustomerBillingZoneId());

	}

	/**
	 * Invoked after addSubscriptionItem
	 * 
	 * @throws Exception
	 */
	public void addItem() throws Exception {

		boolean quantityUpdated = false;

		// get store country
		Map lcountries = RefCache.getAllcountriesmap(LanguageUtil
				.getLanguageNumberCode(value.getLang()));
		if (lcountries != null) {
			Country country = (Country) lcountries.get(store.getCountry());
			getServletRequest().getSession().setAttribute("COUNTRY", country);
		}

		// check if language is supported by the store
		if (lcountries != null) {
			Country country = (Country) lcountries.get(store.getCountry());
			getServletRequest().getSession().setAttribute("COUNTRY", country);
		}

		// store can not be null, if it is the case, generic error page
		if (store == null) {
			throw new Exception("Invalid Store!");
		}

		// check if order product already exist. If that orderproduct already
		// exist
		// and has no ptoperties, so just update the quantity
		if (value.getAttributeId() == null
				|| (value.getAttributeId() != null && value.getAttributeId()
						.size() == 0)) {
			Map savedProducts = SessionUtil
					.getOrderProducts(getServletRequest());
			if (savedProducts != null) {
				Iterator it = savedProducts.keySet().iterator();
				while (it.hasNext()) {
					String line = (String) it.next();
					OrderProduct op = (OrderProduct) savedProducts.get(line);
					if (op.getProductId() == value.getProductId()) {
						Set attrs = op.getOrderattributes();
						if (attrs.size() == 0) {
							int qty = op.getProductQuantity();
							qty = qty + value.getQty();
							op.setProductQuantity(qty);
							quantityUpdated = true;
							break;
						}
					}
				}
			}
		}

		// create an order with merchantId and all dates
		// will need to create a new order id when submited
		Order order = SessionUtil.getOrder(getServletRequest());
		if (order == null) {
			order = new Order();
		}

		order.setMerchantId(store.getMerchantId());
		order.setDatePurchased(new Date());
		SessionUtil.setOrder(order, getServletRequest());

		if (!StringUtils.isBlank(value.getReturnUrl())) {
			// Return to merchant site Url is set from store.
			value.setReturnUrl(store.getContinueshoppingurl());
		}
		SessionUtil.setMerchantStore(store, getServletRequest());

		if (!quantityUpdated) {// new submission

			// Prepare order
			OrderProduct orderProduct = com.salesmanager.core.util.CheckoutUtil
					.createOrderProduct(value.getProductId(), getLocale(),
							store.getCurrency());
			orderProduct.setProductQuantity(value.getQty());
			orderProduct.setProductId(value.getProductId());

			List<OrderProductAttribute> attributes = new ArrayList<OrderProductAttribute>();
			if (value.getAttributeId() != null
					&& value.getAttributeId().size() > 0) {
				for (Long attrId : value.getAttributeId()) {
					if (attrId != null && attrId != 0) {
						ProductAttribute pAttr = cservice
								.getProductAttributeByOptionValueAndProduct(
										value.getProductId(), attrId);
						if (pAttr != null
								&& pAttr.getProductId() == value.getProductId()) {
							OrderProductAttribute orderAttr = new OrderProductAttribute();
							orderAttr.setProductOptionValueId(pAttr
									.getOptionValueId());

							attributes.add(orderAttr);
						} else {
							LogMerchantUtil
									.log(
											value.getMerchantId(),
											getText(
													"error.validation.product.attributes.ids",
													new String[] {
															String
																	.valueOf(attrId),
															String
																	.valueOf(value
																			.getProductId()) }));
						}
					}
				}
			}

			if (!attributes.isEmpty()) {
				// ShoppingCartUtil.addAttributesFromRawObjects(attributes,
				// orderProduct, store.getCurrency(), getServletRequest());
				com.salesmanager.core.util.CheckoutUtil.addAttributesToProduct(
						attributes, orderProduct, store.getCurrency(),
						getLocale());
			}

			Set attributesSet = new HashSet(attributes);
			orderProduct.setOrderattributes(attributesSet);

			SessionUtil.addOrderProduct(orderProduct, getServletRequest());

		}

		// because this is a submission, cannot continue browsing, so that's it
		// for the OrderProduct
		Map orderProducts = SessionUtil.getOrderProducts(super
				.getServletRequest());
		// transform to a list
		List products = new ArrayList();

		if (orderProducts != null) {
			Iterator i = orderProducts.keySet().iterator();
			while (i.hasNext()) {
				String line = (String) i.next();
				OrderProduct op = (OrderProduct) orderProducts.get(line);
				products.add(op);
			}
			super.getServletRequest().getSession().setAttribute(
					"ORDER_PRODUCT_LIST", products);
		}

		// for displaying the order summary, need to create an OrderSummary
		// entity
		OrderTotalSummary summary = oservice.calculateTotal(order, products,
				store.getCurrency(), super.getLocale());

		Map totals = OrderUtil.getOrderTotals(order.getOrderId(), summary,
				store.getCurrency(), super.getLocale());

		HttpSession session = getServletRequest().getSession();

		// transform totals to a list
		List totalsList = new ArrayList();
		if (totals != null) {
			Iterator totalsIterator = totals.keySet().iterator();
			while (totalsIterator.hasNext()) {
				String key = (String) totalsIterator.next();
				OrderTotal total = (OrderTotal) totals.get(key);
				totalsList.add(total);
			}
		}

		SessionUtil.setOrderTotals(totalsList, getServletRequest());

		value.setLangId(LanguageUtil.getLanguageNumberCode(value.getLang()));
		prepareZones();

		// set locale according to the language passed in parameters and store
		// information
		Locale locale = LocaleUtil.getLocaleFromStoreEntity(store, value
				.getLang());
		setLocale(locale);

	}

	private void prepareZones() {

		if (value != null && value.getProductId() > 0) {
			setCountries(RefUtil.getCountries(value.getLang()));

			if (this.customer == null) {

				customer = SessionUtil.getCustomer(getServletRequest());

				if (customer == null) {

					customer = new Customer();
					customer.setCustomerBillingCountryId(value.getCountryId());

				}

			}

			customer.setLocale(getLocale());

			SessionUtil.setCustomer(customer, getServletRequest());

			Collection zones = RefUtil.getZonesByCountry(customer
					.getCustomerBillingCountryId(), value.getLang());

			if (zones != null && zones.size() > 0) {
				setZonesByCountry(zones);
			} else {
				setZone(customer.getBillingState());
			}

		} else {

			if (this.customer == null) {

				customer = SessionUtil.getCustomer(getServletRequest());

			}
			if (customer != null) {

				customer.setLocale(super.getLocale());

				setCountries(RefUtil.getCountries(super.getLocale()
						.getLanguage()));

				Collection zones = RefUtil.getZonesByCountry(customer
						.getCustomerBillingCountryId(), LocaleUtil
						.getDefaultLocale().getLanguage());

				if (zones != null && zones.size() > 0) {
					setZonesByCountry(zones);
				} else {
					setZone(customer.getBillingState());
				}
			} else {

				setCountries(RefUtil.getCountries(LocaleUtil.getDefaultLocale()
						.getLanguage()));

				Configuration conf = PropertiesUtil.getConfiguration();
				int defaultCountry = conf
						.getInt("core.system.defaultcountryid");
				customer = new Customer();
				customer.setCustomerBillingCountryId(defaultCountry);
				customer.setLocale(super.getLocale());

				Collection zones = RefUtil.getZonesByCountry(customer
						.getCustomerBillingCountryId(), LocaleUtil
						.getDefaultLocale().getLanguage());

				if (zones != null && zones.size() > 0) {
					setZonesByCountry(zones);
				} else {
					setZone(customer.getBillingState());
				}

				SessionUtil.setCustomer(customer, getServletRequest());

			}

		}

	}

	/**
	 * This methhod is for subscription step 1
	 * 
	 * @return
	 */
	public String displaySubscriptionForm() {

		try {

			// check if the session is still active
			Order o = SessionUtil.getOrder(getServletRequest());
			if (o == null) {
				super.addActionError(getText("error.sessionexpired"));
				return "GLOBALERROR";
			}

			// This is for the progress bar
			getServletRequest().setAttribute("STEP", "1");
			prepareZones();

			/*
			 * //STUB CUSTOMER
			 * customer=ShoppingCartUtil.getCustomer(getServletRequest());
			 * customer.setCustomerEmailAddress("carl@csticonsulting.com");
			 * customer.setCustomerFirstname("Carlito");
			 * customer.setCustomerLastname("Samsonos");
			 * customer.setCustomerBillingStreetAddress("358 Du Languedoc");
			 * customer.setCustomerBillingCity("Boucherville");
			 * customer.setCustomerBillingState("Quebec");
			 * customer.setCustomerBillingZoneId(76);
			 * customer.setCustomerBillingCountryId(38);
			 * customer.setCustomerBillingPostalCode("J4B8J9");
			 * customer.setCustomerTelephone("4504497181");
			 * ShoppingCartUtil.setCustomer(customer, getServletRequest());
			 */

			preparePage();
			return SUCCESS;
		} catch (Exception e) {
			super.addActionError(getText("error.process.notransaction"));
			log.error(e);
			return "GLOBALERROR";
		}
	}

	/**
	 * This is the main entry point to the subscription process. In this case
	 * Only one item can be added to the subscription process. Once the product
	 * added, the method displaySubscriptionForm is invoked
	 * 
	 * @return
	 */
	public String addSubscriptionItem() {

		try {

			if (!validateAddSubscription()) {
				return INPUT;
			}

			addItem();
			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			return "GLOBALERROR";
		}

	}

	/**
	 * Validates input parameters for a new subscription request
	 * 
	 * @return
	 */
	public boolean validateAddSubscription() {
		boolean success = true;
		if (value.getMerchantId() == 0) {
			addActionError(getText("error.validation.parameter.missing",
					new String[] { MERCHANT_ID_PARAM }));
			success = false;
		}
		if (value.getProductId() == 0) {
			addActionError(getText("error.validation.parameter.missing",
					new String[] { PRODUCT_ID_PARAM }));
			success = false;
		}

		if (success) {
			try {

				store = mservice.getMerchantStore(value.getMerchantId());
				Collection<MerchantUserInformation> minfo = mservice
						.getMerchantUserInfo(value.getMerchantId());

				if (store == null) {
					addActionError(getText("error.merchant.unavailable",
							new String[] { String
									.valueOf(value.getMerchantId()) }));
					return false;
				}

				// maybe this has to be done
				value.setCountryId(store.getCountry());

				// @TODO log to CommonService
				Product product = cservice.getProduct(value.getProductId());
				if (product == null
						|| product.getMerchantId() != value.getMerchantId()) {
					LogMerchantUtil.log(value.getMerchantId(), getText(
							"error.validation.merchant.product.ids",
							new String[] {
									String.valueOf(value.getProductId()),
									String.valueOf(value.getMerchantId()) }));
					addActionError(getText(
							"error.validation.merchant.product.ids",
							new String[] {
									String.valueOf(value.getProductId()),
									String.valueOf(value.getMerchantId()) }));
					success = false;
				} else {
					if (product.getProductDateAvailable().after(new Date())) {
						LogMerchantUtil.log(value.getMerchantId(), getText(
								"error.product.unavailable.purchase",
								new String[] { String.valueOf(value
										.getProductId()) }));
						addActionError(getText(
								"error.product.unavailable.purchase",
								new String[] { String.valueOf(value
										.getProductId()) }));
						success = false;
					}
					if (product.getProductQuantity() == OUT_OF_STOCK_PRODUCT_QUANTITY) {
						LogMerchantUtil.log(value.getMerchantId(), getText(
								"error.product.unavailable.purchase",
								new String[] { String.valueOf(value
										.getProductId()) }));
						addActionError(getText(
								"error.product.unavailable.purchase",
								new String[] { String.valueOf(value
										.getProductId()) }));

						Configuration config = PropertiesUtil
								.getConfiguration();

						// MerchantProfile profile =
						// mservice.getMerchantProfile(value.getMerchantId());

						String l = config.getString(
								"core.system.defaultlanguage", "en");

						if (minfo == null) {
							log
									.error("MerchantUserInformation is null for merchantId "
											+ value.getMerchantId());
							addActionError(getText(
									"error.product.unavailable.purchase",
									new String[] { String.valueOf(value
											.getProductId()) }));
							// goto global error
							throw new Exception(
									"Invalid MerchantId,Unable to find MerchantProfile");
						}

						MerchantUserInformation user = (MerchantUserInformation)((List)minfo).get(0);

						if (!StringUtils.isBlank(user.getUserlang())) {
							l = user.getUserlang();
						}

						String description = "";

						Collection descriptionslist = product.getDescriptions();
						if (descriptionslist != null) {
							Iterator i = descriptionslist.iterator();
							while (i.hasNext()) {
								Object o = i.next();
								if (o instanceof ProductDescription) {
									ProductDescription desc = (ProductDescription) o;
									description = desc.getProductName();
									if (desc.getId().getLanguageId() == LanguageUtil
											.getLanguageNumberCode(l)) {
										description = desc.getProductName();
										break;
									}
								}
							}
						}

						List params = new ArrayList();
						params.add(description);
						params.add(product.getProductId());

						LabelUtil lhelper = LabelUtil.getInstance();
						String subject = lhelper.getText(super.getLocale(),
								"label.email.store.outofstock.subject");
						String productId = lhelper.getText(super.getLocale(),
								"label.email.store.outofstock.product", params);

						Map emailctx = new HashMap();
						emailctx.put("EMAIL_STORE_NAME", store.getStorename());
						emailctx.put("EMAIL_PRODUCT_TEXT", productId);

						CommonService cservice = new CommonService();
						cservice.sendHtmlEmail(store.getStoreemailaddress(),
								subject, store, emailctx,
								"email_template_outofstock.ftl", store
										.getDefaultLang());

						success = false;

					} else if (product.getProductQuantity() < LOW_STOCK_PRODUCT_QUANTITY) {

						Configuration config = PropertiesUtil
								.getConfiguration();

						// MerchantProfile profile =
						// mservice.getMerchantProfile(value.getMerchantId());

						String l = config.getString(
								"core.system.defaultlanguage", "en");

						if (minfo == null) {
							log
									.error("MerchantUserInformationis null for merchantId "
											+ value.getMerchantId());
							addActionError(getText(
									"error.product.unavailable.purchase",
									new String[] { String.valueOf(value
											.getProductId()) }));
							// goto global error
							throw new Exception(
									"Invalid MerchantId,Unable to find MerchantProfile");
						}
						
						MerchantUserInformation user = (MerchantUserInformation)((List)minfo).get(0);

						if (!StringUtils.isBlank(user.getUserlang())) {
							l = user.getUserlang();
						}



						String description = "";

						Collection descriptionslist = product.getDescriptions();
						if (descriptionslist != null) {
							Iterator i = descriptionslist.iterator();
							while (i.hasNext()) {
								Object o = i.next();
								if (o instanceof ProductDescription) {
									ProductDescription desc = (ProductDescription) o;
									description = desc.getProductName();
									if (desc.getId().getLanguageId() == LanguageUtil
											.getLanguageNumberCode(l)) {
										description = desc.getProductName();
										break;
									}
								}
							}
						}

						List params = new ArrayList();
						params.add(description);
						params.add(product.getProductId());

						LabelUtil lhelper = LabelUtil.getInstance();
						String subject = lhelper.getText(l,
								"label.email.store.lowinventory.subject");
						String productId = lhelper.getText(super.getLocale(),
								"label.email.store.lowinventory.product",
								params);

						Map emailctx = new HashMap();
						emailctx.put("EMAIL_STORE_NAME", store.getStorename());
						emailctx.put("EMAIL_PRODUCT_TEXT", productId);

						CommonService cservice = new CommonService();
						cservice.sendHtmlEmail(store.getStoreemailaddress(),
								subject, store, emailctx,
								"email_template_lowstock.ftl", store
										.getDefaultLang());

					}

				}

			} catch (Exception e) {
				log.error("Exception occurred while getting product by Id", e);
				addActionError(getText("errors.technical"));
			}
		}

		return success;

	}

	public Map getPaymentMethods() {
		return paymentMethods;
	}

	public void setPaymentMethods(Map paymentMethods) {
		this.paymentMethods = paymentMethods;
	}

	public Collection<Country> getCountries() {
		return countries;
	}

	public void setCountries(Collection<Country> countries) {
		this.countries = countries;
	}

	public Collection<Zone> getZonesByCountry() {
		return zonesByCountry;
	}

	public void setZonesByCountry(Collection<Zone> zonesByCountry) {
		this.zonesByCountry = zonesByCountry;
	}

	public String getConfirmEmailAddress() {
		return confirmEmailAddress;
	}

	public void setConfirmEmailAddress(String confirmEmailAddress) {
		this.confirmEmailAddress = confirmEmailAddress;
	}

	public String getConfirmPassword() {
		return confirmPassword;
	}

	public void setConfirmPassword(String confirmPassword) {
		this.confirmPassword = confirmPassword;
	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

	public String getCustomerBillingStreetAddress1() {
		return customerBillingStreetAddress1;
	}

	public void setCustomerBillingStreetAddress1(
			String customerBillingStreetAddress1) {
		this.customerBillingStreetAddress1 = customerBillingStreetAddress1;
	}

	public String getCustomerBillingStreetAddress2() {
		return customerBillingStreetAddress2;
	}

	public void setCustomerBillingStreetAddress2(
			String customerBillingStreetAddress2) {
		this.customerBillingStreetAddress2 = customerBillingStreetAddress2;
	}

	public String getStoreCountry() {
		return storeCountry;
	}

	public void setStoreCountry(String storeCountry) {
		this.storeCountry = storeCountry;
	}

	public Collection getCreditCards() {
		return creditCards;
	}

	public void setCreditCards(Collection creditCards) {
		this.creditCards = creditCards;
	}

	public PaymentMethod getPaymentMethod() {
		return paymentMethod;
	}

	public void setPaymentMethod(PaymentMethod paymentMethod) {
		this.paymentMethod = paymentMethod;
	}

	public String getBillingState() {
		return billingState;
	}

	public void setBillingState(String billingState) {
		this.billingState = billingState;
	}

	public String getZone() {
		return zone;
	}

	public void setZone(String zone) {
		this.zone = zone;
	}

	public String getFormstate() {
		return formstate;
	}

	public void setFormstate(String formstate) {
		this.formstate = formstate;
	}

}



```
