# LogonAction.java

## Review

## 1. Summary  

| Aspect | Observation |
|--------|-------------|
| **Purpose** | Implements the authentication flow for the *SalesManager* web application. Handles user login, session creation, context population and locale setting. |
| **Key Components** | * `MerchantService` – business layer that validates credentials and retrieves merchant‑related data.<br>* `Context` – custom object stored in the HTTP session that holds the current user/merchant context.<br>* `RefCache` – in‑memory cache for reference data (countries, etc.).<br>* `LanguageHelper`, `LanguageUtil`, `LabelUtil` – i18n support. |
| **Frameworks / Patterns** | - Appears to be a **Struts 2** action (extends `BaseAction` and returns a string result).<br>- **Service Factory** pattern to obtain the `MerchantService` instance.<br>- **Cache** pattern for country data.<br>- **Command/Action** pattern: `logon()` and `logout()` are the command methods executed by the framework. |

---

## 2. Detailed Description  

### 2.1 Flow of Execution  

1. **Initialization** – `logon` object is instantiated.  
2. **Authentication** – `MerchantService.adminLogon()` is called with the supplied credentials.  
3. **Profile Retrieval** – merchant registration and store objects are fetched.  
4. **Role Determination** – user roles are pulled, the *master role* is derived, and the role list is stored in the session.  
5. **Context Creation** – a `Context` instance is populated with merchant id, registration code, promo code, language, user name, store data (currency, units, locale, zone, etc.).  
6. **Session Handling** – old context removed, new context & merchant id stored, an admin token is added to the session.  
7. **Locale Setup** – country data is retrieved from `RefCache` and a `Locale` is created based on the selected language and country.  
8. **Error Handling** – any exception is caught (currently `Throwable`).  
   * For `ServiceException` with `INVALID_CREDENTIALS` the user‑friendly error is set in the `logon` object.  
   * All other errors are logged and a generic technical error is set.  
9. **Result** – the method returns the Struts result string `SUCCESS` (or `SUCCESS` string literal for JSON requests).  

### 2.2 Assumptions & Constraints  

* The servlet container supplies a valid `HttpServletRequest`/`HttpSession`.  
* The `BaseAction` class already provides `getServletRequest()`, `getPrincipal()`, `setLocale()`, etc.  
* `MerchantService` throws a `ServiceException` for business‑level errors.  
* The `Context` object is serialisable (needed if session replication is used).  
* All strings and numbers retrieved from the service layer are assumed non‑null unless explicitly checked.  

### 2.3 Architecture & Design Choices  

* **Action‑Based** – Each request is handled by a Struts action (`logon`, `logout`).  
* **Service Layer** – Business logic lives in `MerchantService`.  
* **Caching** – Reference data (countries) is cached in `RefCache`.  
* **i18n** – Uses Apache Commons `StringUtils` for string checks and custom utilities for label look‑ups.  
* **Session‑Scoped Context** – The `Context` holds all state needed for the remainder of the user’s session.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `public String logon()` | Main login workflow. | None – uses `username` & `password` fields. | Struts result string (`SUCCESS`) | Creates `logon`, populates session, sets locale, logs errors. |
| `public String logout()` | Clears session data and prepares the logout view. | None | Struts result string (`SUCCESS`) | Sets `lang` attribute in request. |
| `public Logon getLogon()` | Getter for the `logon` object used by the view. | None | `Logon` instance | None |
| `public void setLogon(Logon logon)` | Setter for dependency injection or view usage. | `Logon` | None | None |

### Reusable / Utility Methods  

* None are defined in this class – all logic is contained in `logon()`/`logout()`.  
* Utility classes (`LabelUtil`, `LanguageHelper`, `LanguageUtil`, `MessageUtil`) are used but not defined here.

---

## 4. Dependencies  

| Library / Package | Type | Notes |
|-------------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String helpers. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.*` | Internal | Core domain entities, services, constants, utilities. |
| `java.util` | Standard | Collections, Locale. |
| `javax.servlet.http.*` (implied via `BaseAction`) | Platform | Servlet API. |

*No external frameworks beyond Struts (inferred) and Log4j.*  

---

## 5. Additional Notes & Recommendations  

### 5.1 Error Handling  
* **Catching `Throwable`** – This will swallow `Error` subclasses (e.g., `OutOfMemoryError`) that should propagate. Switch to `Exception` or more specific exceptions.  
* **Return Value** – The method always returns `SUCCESS` even on login failure. This may be intentional for a JSON API, but better to return distinct result strings (`error`, `loginFailed`) to allow the framework to dispatch appropriately.  
* **User‑Facing Messages** – The code sets an error message on the `logon` object, but the view is not shown. Ensure the front‑end reads this object.  

### 5.2 Security  
* **Password Handling** – `username` and `password` fields are stored as plain strings; consider marking them `transient` and clearing after use.  
* **Session Fixation** – No session regeneration on login. Adding `request.getSession(true)` or `invalidate()` + `createNewSession()` can mitigate fixation attacks.  
* **Logging** – The stack trace is logged but not the credentials; verify that no sensitive data leaks.  

### 5.3 Null Checks & Defaults  
* `service.adminLogon()` may return `null` – guard against NPEs when accessing `merchantProfile`.  
* `store` can be `null`; the code handles this but some fields (e.g., `store.getSeizeunitcode()`) might still be accessed.  

### 5.4 Code Clean‑up  
* Unused imports (`Collection`, `Iterator`, `Map`) are actually used, but check for any that can be removed.  
* The `logon` variable is only used for error messages; consider returning error objects directly.  

### 5.5 Extensibility  
* **Logging & Metrics** – Wrap the login logic with a timer to record authentication latency.  
* **Internationalization** – The locale is derived from store country; consider a fallback chain (user preference → store → default).  
* **Session Management** – Centralise session attribute names (`ProfileConstants.context`, `ProfileConstants.merchant`) into a dedicated constants class for consistency.  

### 5.6 Unit Testing  
* The method is tightly coupled to the service layer; introduce dependency injection (constructor or setter) for `MerchantService` to enable mocking in tests.  
* Extract smaller private helper methods (e.g., `populateContext`, `setLocale`) to increase testability.  

---

**Overall:**  
The `LogonAction` class performs its core function – authenticating a merchant admin and setting up the session – but it could benefit from tighter exception handling, clearer result semantics, enhanced security practices, and better separation of concerns to improve maintainability and testability.

## Code Critique



## Code Preview

```java
package com.salesmanager.central.profile;

import java.util.Collection;
import java.util.Iterator;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.util.LanguageHelper;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.merchant.MerchantRegistration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.merchant.MerchantUserRole;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.MessageUtil;

public class LogonAction extends BaseAction {
	
	private Logger log = Logger.getLogger(LogonAction.class);
	
	private String username;
	private String password;
	
	private Logon logon;
	/**
	 * This is the entry point to the system
	 * 
	 * @return
	 * @throws Exception
	 */
	public String logon() {

		logon = new Logon(); 
		
		try {

			// Logon script
			MerchantService service = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantUserInformation merchantProfile = service.adminLogon(super.getServletRequest(), this.getUsername(), this.getPassword());

			// get MerchantUserRegistration
			MerchantRegistration registration = service
					.getMerchantRegistration(merchantProfile.getMerchantId());

			// get MerchantStore
			MerchantStore store = service.getMerchantStore(merchantProfile.getMerchantId());

			
			//get roles
			Collection roles = service.getUserRoles(merchantProfile.getAdminName());
			
			super.getServletRequest().getSession().setAttribute("roles", roles);
			
			//create context - stored in http session
			Context ctx = new Context();
			
			//set master role
			if (roles != null && roles.size() > 0) {
				Iterator i = roles.iterator();
				while (i.hasNext()) {
					MerchantUserRole r = (MerchantUserRole) i.next();
					if (r.getRoleCode().equals("superuser") || r.getRoleCode().equals("admin") || r.getRoleCode().equals("user")) {
						ctx.setMasterRole(r.getRoleCode());
					}
				}
			}
			
			
			ctx.setMerchantid(merchantProfile.getMerchantId());
			ctx.setRegistrationCode(registration
					.getMerchantRegistrationDefCode());
			ctx.setPromoCode(new Integer(registration.getPromoCode()));
			if (!StringUtils.isBlank(merchantProfile.getUserlang())) {
				ctx.setLang(merchantProfile.getUserlang());
			} else {
				ctx.setLang(LanguageUtil.getDefaultLanguage());
			}
			ctx.setUsername(super.getPrincipal().getRemoteUser());

			if (store != null) {

				ctx.setCurrency(store.getCurrency());
				ctx.setSizeunit(store.getSeizeunitcode());
				ctx.setWeightunit(store.getWeightunitcode());
				LanguageHelper.setLanguages(store.getSupportedlanguages(), ctx);

				if (store.getCountry() == 0) {

					ctx.setCountryid(merchantProfile.getUsercountrycode());

				} else {

					ctx.setCountryid(store.getCountry());

				}

				if (StringUtils.isBlank(store.getZone())) {

					if (StringUtils.isNumeric(merchantProfile.getUserstate())) {
						ctx.setZoneid(Integer.parseInt(merchantProfile
								.getUserstate()));
					} else {
						ctx.setZoneid(0);
					}

				} else {

					if (StringUtils.isNumeric(store.getZone())) {
						ctx.setZoneid(Integer.parseInt(store.getZone()));
					} else {
						ctx.setZoneid(0);
					}
				}

				// set default values
				if (StringUtils.isBlank(store.getCurrency())) {
					ctx.setCurrency(Constants.CURRENCY_CODE_USD);
				}

				if (StringUtils.isBlank(store.getWeightunitcode())) {
					ctx.setWeightunit(Constants.LB_WEIGHT_UNIT);
				}

				if (StringUtils.isBlank(store.getSeizeunitcode())) {
					ctx.setWeightunit(Constants.INCH_SIZE_UNIT);
				}

			} else {

				ctx.setCountryid(Constants.US_COUNTRY_ID);
				ctx.setCurrency(Constants.CURRENCY_CODE_USD);
				ctx.setZoneid(0);
				ctx.setExistingStore(false);
			}

			// If country / zone not set, set default values of user until user
			// decides to
			// configure

			// end default settings

			// cleanup previous sessions object
			super.getServletRequest().getSession().removeAttribute(
					ProfileConstants.context);
			super.getServletRequest().getSession().setAttribute(
					ProfileConstants.merchant, merchantProfile.getMerchantId());
			super.getServletRequest().getSession().setAttribute(
					ProfileConstants.context, ctx);
			setAdminTokenToSession(merchantProfile.getMerchantId());

			RefCache cache = RefCache.getInstance();

			Map countries = cache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(ctx.getLang()));
			Country c = (Country) countries.get(ctx.getCountryid());

			//create locale
			Locale locale = new Locale(ctx.getLang(), c.getCountryIsoCode2());
			super.setLocale(locale);

		

		} catch (Throwable e) {
			
			LabelUtil label = LabelUtil.getInstance();
			label.setLocale(super.getLocale());

			//super.getServletRequest().setAttribute("error_message",
			//		LabelUtil.getInstance().getText("errors.technical"));
			logon.setErrorMessage(label.getText("errors.technical"));

			if (e instanceof ServiceException) {
				ServiceException se = (ServiceException) e;

				if (se.getReason() == ErrorConstants.INVALID_CREDENTIALS) {
					//MessageUtil.addErrorMessage(super.getServletRequest(),
					//		LabelUtil.getInstance().getText(
					//				"errors.invalidcredentials"));
					logon.setErrorMessage(label.getText("errors.invalidcredentials"));
					//super.getServletRequest().setAttribute(
					//		"error_message",
					//		LabelUtil.getInstance().getText(
					//				"errors.invalidcredentials"));
					return SUCCESS;
				} 
			}

			//MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
			//		.getInstance().getText("messages.profilecreationerror"));
			log.error(e);
			return "SUCCESS";//need to return this anyway for json request
		} 

		return SUCCESS;
	}
	
	/**
	 * Logout action
	 * 
	 * @return
	 * @throws Exception
	 */
	public String logout() throws Exception {

		Locale locale = getLocale();
		String lang = locale.getLanguage();
		super.getServletRequest().setAttribute("lang", lang);
		return SUCCESS;
	}

	public Logon getLogon() {
		return logon;
	}

	public void setLogon(Logon logon) {
		this.logon = logon;
	}

}



```
