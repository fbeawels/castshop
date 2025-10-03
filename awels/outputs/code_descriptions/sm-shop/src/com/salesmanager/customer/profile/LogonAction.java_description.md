# LogonAction.java

## Review

## 1. Summary

**Purpose**  
`LogonAction` is a Struts‑style action that handles all customer‑facing authentication and registration flows for the SalesManager e‑commerce platform.  
It supports:

* Remote and local log‑in (`remoteLogon()`, `localLogon()`)
* Registration (`displayRegistration()`, `registerCustomer()`)
* Password reset (`displayResetPassword()`, `resetPassword()`)
* CAPTCHA image generation (`generateCaptchaImage()`)

**Key components**

| Component | Role |
|-----------|------|
| `CustomerService` | Persist and retrieve `Customer` entities |
| `MerchantStore` | Store‑level configuration (id, default country, etc.) |
| `RefCache` | Cache for countries and zones |
| `CaptchaModule` | Generate and validate CAPTCHA challenges |
| `ServiceFactory` | Lookup of service beans |
| `SpringUtil` | Spring bean lookup (e.g., `CaptchaModule`) |
| `SessionUtil` | Access to the current `MerchantStore` via the HTTP session |
| `CustomerUtil` / `CountryUtil` | Helper utilities for validation / lookup |

The class uses **Log4j** for logging, **Apache Commons Lang** for string handling, and a **custom CAPTCHA** implementation. It also relies on **JDK internal image codecs** (`com.sun.image.codec.jpeg.JPEGCodec`), which are deprecated.

---

## 2. Detailed Description

### Execution Flow

1. **Login**  
   * `remoteLogon()` or `localLogon()` delegate to `super.logon()`.  
   * On failure, `localLogon()` interprets `ServiceException` and sets a user‑visible error message, while the generic `remoteLogon()` just logs the exception.

2. **Registration Page**  
   * `displayRegistration()` prepares default data:
     * Determines default country (from `core.system.defaultcountryid` or the request locale).
     * Calls `prepareZones()` to load zones and countries for drop‑downs.
     * Generates a CAPTCHA image (`generateCaptchaImage()`).
     * Instantiates an empty `Customer` with the default country pre‑set.

3. **Registration Submission**  
   * `registerCustomer()` performs:
     * Re‑loading zones and re‑generating the CAPTCHA for display after a failed attempt.
     * CAPTCHA validation (`CaptchaModule.validateResponseForSessonId` – note the typo in method name).
     * Basic field validation (first/last name, email, email repeat, state if required).
     * Duplicate‑email check via `CustomerService.findCustomerByUserName`.
     * Populates default dummy values for fields that the database marks as required but the UI does not capture.
     * Persists the customer via `CustomerService.saveOrUpdateCustomer`.
     * Sets a success message.

4. **Password Reset**  
   * `resetPassword()` fetches the user by name and triggers `CustomerService.resetCustomerPassword`.  
   * The action does not send any email; it merely updates the password in the database.

5. **CAPTCHA Generation**  
   * `generateCaptchaImage()` creates a JPEG image from the CAPTCHA module, writes it to the session under `"CAPTCHAIMAGE"`, and handles various error conditions.

### Assumptions & Constraints

* The action relies on a **Servlet container** and the Struts 2 framework (e.g., `execute()`, return strings map to result pages).
* A **pre‑configured Spring context** must contain a bean named `"captcha"` implementing `CaptchaModule`.
* `RefCache` is assumed to be thread‑safe (caching countries/zones).
* The platform uses **SessionUtil** to keep the `MerchantStore` and uses **Locale** from the request for internationalisation.
* The code does **not** validate phone, address, or postal code, inserting placeholder values instead.

### Architectural Observations

* **Thin controller, heavy business logic** – The action mixes UI handling, validation, and persistence. A cleaner separation (Service/Validator layers) would improve testability.
* **Inconsistent error handling** – Some methods catch generic `Exception`, others catch `ServiceException` only. Logging is present but user‑visible errors are often hardcoded.
* **Deprecated image API** – `com.sun.image.codec.jpeg.*` should be replaced with `javax.imageio.ImageIO`.
* **Hard‑coded logger class** – `private static Logger logger = Logger.getLogger(CheckoutAction.class);` references `CheckoutAction` instead of `LogonAction`, likely a copy‑paste bug.
* **CAPTCHA session key** – The image is stored as a byte array under `"CAPTCHAIMAGE"`; no expiration or invalidation strategy is visible.

---

## 3. Functions / Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `remoteLogon()` | Delegates to parent `logon()` for remote authentication. | None | `SUCCESS` or `SUCCESS` on error (no change) | Logs exception |
| `localLogon()` | Same as `remoteLogon()` but sets error message on invalid credentials. | None | `SUCCESS` or `ERROR` | Sets error/technical messages |
| `authenticateCustomer(HttpServletRequest)` | Calls `super.logonCustomer()`. | `HttpServletRequest` | `Customer` | Sets servlet request, throws if error |
| `prepareZones()` | Populates `zones` and `countries` based on default or locale country. | None | None | Queries cache, sets fields |
| `displayRegistration()` | Prepares the registration page (customer object, zones, captcha). | None | `SUCCESS` | Generates captcha, logs exceptions |
| `registerCustomer()` | Validates input, checks duplicate, persists new customer. | None | `SUCCESS`, `INPUT`, or `GENERICERROR` | Sets messages, updates DB |
| `resetPassword()` | Resets password for a given username. | None | `SUCCESS` | Calls service, sets message |
| `displayResetPassword()` | Simple navigation. | None | `SUCCESS` | None |
| `generateCaptchaImage()` | Creates JPEG CAPTCHA image and stores it in session. | None | None | Sets session attribute, may send HTTP error |

**Reusable / Utility Methods**

* `prepareZones()` is a helper used by multiple actions.
* `generateCaptchaImage()` is a self‑contained utility for CAPTCHA handling.

---

## 4. Dependencies

| Library | Purpose | Third‑Party / Standard |
|---------|---------|------------------------|
| `org.apache.commons.lang.StringUtils` | String null/blank checks | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `com.octo.captcha.service.CaptchaServiceException` | CAPTCHA service exceptions | Third‑party |
| `com.salesmanager.core.*` | Core domain entities, services, utilities, constants | Project‑specific |
| `com.sun.image.codec.jpeg.*` | JPEG encoding of CAPTCHA | JDK internal (deprecated) |
| `javax.servlet.http.*` | Servlet request/response | Standard |
| `org.apache.struts2.*` (implied) | Action mapping, result handling | Framework |

**Platform‑Specific Assumptions**

* Runs under a Servlet container supporting JDK 8 or earlier (due to use of `com.sun.image.codec.jpeg`).
* Relies on Spring’s `ApplicationContext` for bean lookup.
* Uses the SalesManager’s custom cache (`RefCache`) and localization utilities.

---

## 5. Additional Notes & Recommendations

### Security & Robustness

1. **CAPTCHA** – The image is stored as raw JPEG bytes in the session. Consider using a more modern encoding (PNG) via `ImageIO`.  
2. **Password Reset** – The method silently resets the password without verifying ownership or sending an email. This could allow attackers to reset any user’s password. A more secure flow would involve:
   * Sending a time‑limited reset token to the user’s verified email.
   * Prompting the user to set a new password via a secure link.
3. **Email Validation** – The code checks email format only via `CustomerUtil.validateEmail`. Ensure this method uses RFC‑compliant regex or a library like Apache Commons Validator.
4. **Dummy Data** – Storing `"---"` for mandatory fields is risky. The database schema should allow `NULL` for optional fields, or the UI should collect them.

### Code Quality

1. **Logger** – Replace the incorrect `Logger.getLogger(CheckoutAction.class)` with `LogonAction.class`.
2. **Deprecated APIs** – Replace `com.sun.image.codec.jpeg.*` with `ImageIO` or a third‑party JPEG writer.
3. **Method Naming** – `validateResponseForSessonId` contains a typo. Verify the actual method name in `CaptchaModule`.
4. **Exception Handling** – Catch specific exceptions; avoid swallowing generic `Exception` without providing context. Use a `try‑catch` block only around code that may legitimately throw.
5. **Action Result Strings** – Use constants (`SUCCESS`, `INPUT`, `ERROR`) instead of hard‑coded literals for maintainability.
6. **Internationalization** – The action uses `super.getText()` for message keys. Ensure all keys exist and are correctly translated.

### Architectural Enhancements

1. **Separation of Concerns** – Move validation logic into a dedicated `Validator` or `Service` layer. The action should only orchestrate UI flow.
2. **DTO Usage** – Instead of populating the `Customer` entity directly from the UI, use a Data Transfer Object (DTO) and map it to the entity in the service layer.
3. **Unit Tests** – With the current structure, unit testing is difficult. Refactoring would allow mocking services and testing business logic in isolation.
4. **CAPTCHA Configuration** – Store CAPTCHA settings (image dimensions, noise level) in a properties file and inject them via Spring.

### Performance & Scalability

* **Cache Refresh** – `RefCache.getAllcountriesmap` and `getFilterdByCountryZones` should be monitored for memory usage. Use a time‑to‑live policy if the data changes rarely.
* **Session Size** – Storing the entire JPEG byte array in the session may increase memory usage for each user. Consider streaming the image directly to the response instead of caching.

---

### Bottom Line

`LogonAction` covers a wide range of user‑management features but mixes UI handling, validation, and persistence in a single class. While functional, the implementation relies on outdated APIs, contains a few copy‑paste bugs, and would benefit from a more modular architecture. Addressing the security concerns around password reset and refactoring to separate business logic will make the codebase more robust, testable, and maintainable.

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

import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Locale;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.octo.captcha.service.CaptchaServiceException;
import com.salesmanager.catalog.cart.CheckoutAction;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.SystemUrlEntryType;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.module.model.application.CaptchaModule;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.CustomerUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.AuthenticateCustomerAction;
import com.salesmanager.core.util.www.SessionUtil;
import com.sun.image.codec.jpeg.JPEGCodec;
import com.sun.image.codec.jpeg.JPEGImageEncoder;

/**
 * Manages logon, logout and register functions
 * 
 * @author Carl Samson
 * 
 */
public class LogonAction extends AuthenticateCustomerAction {

	private static Logger logger = Logger.getLogger(CheckoutAction.class);

	private Customer customer = null;// from registration form
	private Collection<Zone> zones = new ArrayList();// collection for drop down
														// list
	private Collection<Country> countries;// collection for drop down list
	private String customerEmailAddressRepeat;

	public String getCustomerEmailAddressRepeat() {
		return customerEmailAddressRepeat;
	}

	public void setCustomerEmailAddressRepeat(String customerEmailAddressRepeat) {
		this.customerEmailAddressRepeat = customerEmailAddressRepeat;
	}

	private String formstate = "";

	public String getFormstate() {
		return formstate;
	}

	public void setFormstate(String formstate) {
		this.formstate = formstate;
	}

	public Collection<Zone> getZones() {
		return zones;
	}

	public void setZones(Collection<Zone> zones) {
		this.zones = zones;
	}

	public Collection<Country> getCountries() {
		return countries;
	}

	public void setCountries(Collection<Country> countries) {
		this.countries = countries;
	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

	public String remoteLogon() {

		try {
			return super.logon();
		} catch (Exception e) {
			logger.error(e);
			return SUCCESS;
		}
	}

	public String localLogon() {

		try {
			return super.logon();
		} catch (Exception e) {
			
			if(e instanceof ServiceException) {
				if(((ServiceException) e).getReason()==ErrorConstants.INVALID_CREDENTIALS) {
					super.setErrorMessage("login.invalid");
				}
			} else {
				super.setTechnicalMessage();
			}
			
			return ERROR;
		}
	}

	public Customer authenticateCustomer(HttpServletRequest request)
			throws ServiceException, Exception {

		super.setServletRequest(request);

		customer = super.logonCustomer();
		return customer;

	}

	private void prepareZones() throws Exception {
		int shippingCountryId = PropertiesUtil.getConfiguration().getInt(
				"core.system.defaultcountryid", Constants.US_COUNTRY_ID);
		Locale locale = super.getLocale();
		String countryCode = locale.getCountry();

		if (!StringUtils.isBlank(countryCode)) {
			CountryDescription country = CountryUtil.getCountryByIsoCode(
					countryCode, locale);
			shippingCountryId = country.getId().getCountryId();
		}

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
	 * Prepares object for registration form
	 * 
	 * @return
	 */
	public String displayRegistration() {

		try {

			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());
			Integer merchantid = store.getMerchantId();

			int shippingCountryId = PropertiesUtil.getConfiguration().getInt(
					"core.system.defaultcountryid", Constants.US_COUNTRY_ID);

			Locale locale = super.getLocale();
			String countryCode = locale.getCountry();

			if (!StringUtils.isBlank(countryCode)) {
				CountryDescription country = CountryUtil.getCountryByIsoCode(
						countryCode, locale);
				shippingCountryId = country.getId().getCountryId();
			}

			prepareZones();

			generateCaptchaImage();

			Customer c = new Customer();
			c.setCustomerCountryId(shippingCountryId);
			c.setCustomerBillingCountryId(shippingCountryId);
			this.setCustomer(c);

		} catch (Exception e) {
			logger.error(e);
		}

		return SUCCESS;

	}

	public String registerCustomer() {

		try {

			prepareZones();

			String captchaId = getServletRequest().getSession().getId();
			// retrieve the response

			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());

			CaptchaModule module = (CaptchaModule) SpringUtil
					.getBean("captcha");
			boolean isResponseCorrect = module.validateResponseForSessonId(
					captchaId, (String) getServletRequest().getParameter(
							"captcha_response"));

			generateCaptchaImage();

			// check fields
			boolean hasError = false;

			if (!isResponseCorrect) {
				super.addFieldMessage("captcha_response",
						"messages.error.captcha");
				hasError = true;
			}

			if (customer == null) {
				logger.error("Customer is null");
				return "GENERICERROR";
			}
			if (StringUtils.isBlank(customer.getCustomerFirstname())) {
				super.addFieldMessage("customer.customerFirstName",
						"messages.required.firstname");
				hasError = true;
			}
			if (StringUtils.isBlank(customer.getCustomerLastname())) {
				super.addFieldMessage("customer.customerLastName",
						"messages.required.lastname");
				hasError = true;
			}

			if (StringUtils.isBlank(customer.getCustomerEmailAddress())) {
				super.addFieldMessage("customer.customerEmailAddress",
						"messages.invalid.email");
				hasError = true;
			}

			if (StringUtils.isBlank(this.getCustomerEmailAddressRepeat())) {
				super.addFieldMessage("customerEmailAddressRepeat",
						"messages.invalid.email");
				hasError = true;
			}

			if (!this.getCustomerEmailAddressRepeat().equals(
					customer.getCustomerEmailAddress())) {
				super.addFieldMessage("customerEmailAddressRepeat",
						"messages.invalid.email");
				hasError = true;
			}

			if (!CustomerUtil.validateEmail(customer.getCustomerEmailAddress())) {
				super.addFieldMessage("customer.customerEmailAddress",
						"messages.invalid.email");
			}

			if (!StringUtils.isBlank(this.getFormstate())
					&& this.getFormstate().equals("text")) {
				if (StringUtils.isBlank(customer.getCustomerState())) {
					super.addFieldMessage("customer.customerState",
							"messages.required.state");
					hasError = true;
				}
			}

			if (hasError) {
				return INPUT;
			}

			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);

			// check if email address already exist
			Customer tmpCustomer = cservice.findCustomerByUserName(customer
					.getCustomerEmailAddress(), store.getMerchantId());
			if (tmpCustomer != null) {
				// user already exist, display reset password message
				super.addActionError(getText("messages.customer.alreadyexist"));
				return INPUT;
			}

			customer.setMerchantId(store.getMerchantId());
			customer.setCustomerBillingCountryId(customer.getCustomerZoneId());
			customer.setCustomerBillingState(customer.getBillingState());
			customer.setCustomerBillingZoneId(customer.getCustomerZoneId());
			customer.setCustomerAnonymous(false);
			customer.setCustomerLang(super.getLocale().getLanguage());

			// telephone, address, city and postal code are req in the db but
			// not during reistration
			// so here is a dummy string
			customer.setCustomerTelephone("---");
			customer.setCustomerPostalCode("---");
			customer.setCustomerStreetAddress("---");
			customer.setCustomerCity("---");

			cservice.saveOrUpdateCustomer(this.getCustomer(),
					SystemUrlEntryType.WEB, super.getLocale());

			// display message to customer
			super.setMessage("messages.customer.customerregistered");

		} catch (Exception e) {
			logger.error(e);

			super.setTechnicalMessage();
			return INPUT;
		}

		return SUCCESS;

	}

	public String resetPassword() {

		try {

			String userName = getServletRequest().getParameter(
					"resetpasswordusername");
			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);

			MerchantStore store = SessionUtil
					.getMerchantStore(getServletRequest());
			Customer customer = cservice.findCustomerByUserName(userName, store
					.getMerchantId());

			if (customer != null) {
				cservice.resetCustomerPassword(customer);
			}

			super.setMessage("label.customer.passwordreset");

		} catch (Exception e) {
			logger.error(e);
		}
		return SUCCESS;
	}

	public String displayResetPassword() {
		return SUCCESS;
	}

	private void generateCaptchaImage() throws Exception {

		byte[] captchaChallengeAsJpeg = null;
		// the output stream to render the captcha image as jpeg into
		ByteArrayOutputStream jpegOutputStream = new ByteArrayOutputStream();

		try {

			String captchaId = getServletRequest().getSession().getId();

			CaptchaModule module = (CaptchaModule) SpringUtil
					.getBean("captcha");

			BufferedImage challenge = module.getImageForSessionId(captchaId,
					getServletRequest());

			// a jpeg encoder
			JPEGImageEncoder jpegEncoder = JPEGCodec
					.createJPEGEncoder(jpegOutputStream);
			jpegEncoder.encode(challenge);
		} catch (IllegalArgumentException e) {
			getServletResponse().sendError(HttpServletResponse.SC_NOT_FOUND);
			return;
		} catch (CaptchaServiceException e) {
			getServletResponse().sendError(
					HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
			return;
		}

		captchaChallengeAsJpeg = jpegOutputStream.toByteArray();

		getServletRequest().getSession().setAttribute("CAPTCHAIMAGE",
				captchaChallengeAsJpeg);
	}

}



```
