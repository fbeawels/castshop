# BaseAction.java

## Review

## 1. Summary

`BaseAction` is a **Struts 2** action base class that consolidates common behaviour for all actions in the *SalesManager* application.  
It:

| Purpose | Key Responsibility |
|---------|--------------------|
| **Locale handling** | Overrides `getLocale()` and `setLocale()` to persist the locale in the session and request attributes. |
| **Internationalisation** | Uses `LabelUtil` and `MessageUtil` to fetch localized texts and to queue user‑visible messages (success, error, authorization, technical). |
| **Language support** | Loads supported `Language` entities from `MerchantService` and exposes a collection plus a reference map for view rendering. |
| **Authorization** | Provides a generic `authorize(IMerchant)` helper that throws an `AuthorizationException` when a merchant does not match the current context. |
| **Session utilities** | Offers helpers for retrieving the current `Principal`, for setting an admin token, and for accessing the `Context` (merchant‑specific data). |
| **Validation** | Implements `ValidationAware` by routing Struts field‑ and action‑errors into the `MessageUtil` error stream. |

Design patterns/techniques used:

* **Facade / Service Locator** – `ServiceFactory` is used to obtain the `MerchantService`.  
* **Proxy** – `SalesManagerPrincipalProxy` wraps the J2EE `Principal`.  
* **Internationalisation** – custom `LabelUtil`/`MessageUtil` wrappers around Java resource bundles.  

The class is meant to be extended; concrete actions inherit locale, message, language, and authorization helpers.

---

## 2. Detailed Description

### Execution Flow

1. **Framework entry** – Struts 2 creates an instance of a concrete subclass, then injects the `HttpServletRequest` and `HttpServletResponse` via `ServletRequestAware`/`ServletResponseAware`.
2. **Session retrieval** – `init()` (intended to be overridden) pulls the `Context` from the HTTP session; this context stores the current merchant ID and language.
3. **Locale handling** – `getLocale()` first attempts to read the locale from the session (`WW_TRANS_I18N_LOCALE`), falling back to Struts’ default.
4. **Action execution** – The subclass’s `execute()` (not shown) runs. Throughout, the subclass can:
   * Call `prepareLanguages()` to populate the language collection.
   * Call `authorize()` to enforce merchant scoping.
   * Call `setSuccessMessage()`, `setErrorMessage()`, etc., to queue user messages.
   * Call `setPageTitle()` to localise the page title.
5. **Error handling** – The class implements `ValidationAware`. Whenever a field or action error is added, it sets the `actionError` flag and forwards the message to `MessageUtil`.
6. **Response** – After `execute()`, Struts 2 renders the view, which can read the locale, page title, languages, and messages that have been stored in the request/session.

### Key Components

| Component | Role |
|-----------|------|
| `ActionSupport` | Base Struts action providing i18n and validation support. |
| `ServletRequestAware / ServletResponseAware` | Injection of `HttpServletRequest/Response`. |
| `ServiceFactory` | Service locator for `MerchantService`. |
| `LabelUtil` | Internationalised text lookup. |
| `MessageUtil` | Centralised error/success message queue (likely stored in request/session). |
| `Context` | Stores merchant‑specific data (merchant ID, language, etc.). |
| `MerchantService` | Business layer for merchants and languages. |

### Dependencies & Constraints

* Relies on **Apache Struts 2** (ActionSupport, ActionContext, ValidationAware).  
* Uses **Log4j** for logging.  
* Depends on custom core classes (`MerchantService`, `LabelUtil`, `MessageUtil`, etc.) that are presumed thread‑safe because actions are request‑scoped.  
* Assumes a J2EE container that stores a `Principal` object under the session key `PRINCIPAL`.  
* The `Context` attribute key is `ProfileConstants.context`; it must be set before any action runs.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `getLocale()` | Overrides Struts locale to use the session‑stored locale (`WW_TRANS_I18N_LOCALE`). | None | `Locale` | Sets the context locale if present. |
| `setLocale(Locale)` | Persists a locale in session, request, and Struts context. | `Locale locale` | None | Stores locale in session and request attributes. |
| `setPageTitle(String key)` | Localises a page title using `LabelUtil`. | `String key` | None | Sets `pageTitle`. |
| `getPrincipal()` | Retrieves the current user principal (if any). | None | `PrincipalProxy` or `null` | None. |
| `prepareLanguages()` | Loads supported languages from `MerchantService`. | None | None | Populates `languages` and `reflanguages`. |
| `setSuccessMessage()` | Queues a generic success message. | None | None | Adds message via `MessageUtil`. |
| `setTechnicalMessage()` | Queues a generic technical error. | None | None | Adds error via `MessageUtil`. |
| `setAuthorizationMessage()` | Queues an authorization failure. | None | None | Adds error via `MessageUtil`. |
| `setMessage(String)` | Queues an arbitrary i18n message. | `String key` | None | Adds message via `MessageUtil`. |
| `setErrorMessage(String)` | Queues an arbitrary i18n error. | `String key` | None | Adds error via `MessageUtil`. |
| `setErrorMessage(String, List)` | Queues a formatted i18n error. | `String key`, `List parameters` | None | Adds formatted error via `MessageUtil`. |
| `addErrorMessages(List<String>)` | Queues a list of error messages. | `List<String>` | None | Delegates to `MessageUtil`. |
| `authorize(IMerchant)` | Checks that a merchant entity belongs to the current context. | `IMerchant entity` | None | Throws `AuthorizationException` on mismatch. |
| `init()` | Hook for subclasses; currently retrieves context from session. | None | None | May reset form errors (commented out). |
| `getContext()` | Returns the `Context` stored in session, also sets language. | None | `Context` | Updates context language. |
| `setAdminTokenToSession(int)` | Stores an admin token flag for super‑merchant registrations. | `int merchantId` | None | Sets session attribute `ADMIN_TOKEN_PARAM`. |
| `isInvalid(String)` | Utility to test for null or empty strings (unused). | `String value` | `boolean` | None. |
| `addFieldError(String, String)` | Implements `ValidationAware`; records a field error. | `String fieldName`, `String errorMessage` | None | Sets `actionError` flag, logs via `MessageUtil`. |
| `addActionMessage(String)` | Implements `ValidationAware`; records an action message. | `String aMessage` | None | Sets `actionError`, logs via `MessageUtil`. |
| `addActionError(String)` | Implements `ValidationAware`; records an action error. | `String anErrorMessage` | None | Sets `actionError`, logs via `MessageUtil`. |
| `setFieldErrors(Map)` | Placeholder that logs a message. | `Map` | None | Sets `actionError`. |
| `setActionMessages(Collection)` | Placeholder that logs a message. | `Collection` | None | Sets `actionError`. |
| `setActionErrors(Collection)` | Placeholder that logs a message. | `Collection` | None | Sets `actionError`. |
| `isActionError()` | Getter for `actionError`. | None | `boolean` | None. |

**Reusable / Utility methods** – `setLocale`, `setPageTitle`, `prepareLanguages`, `authorize`, and the various `set…Message()` helpers are the main reusable pieces.

---

## 4. Dependencies

| Library / Package | Role | Std / 3rd‑Party |
|-------------------|------|-----------------|
| `org.apache.struts2.*` | MVC framework, action base, locale handling, validation interface | 3rd‑party |
| `org.apache.log4j.Logger` | Logging | 3rd‑party |
| `javax.servlet.http.*` | Request/Response handling | Java EE |
| `com.salesmanager.core.*` | Business entities, services, exceptions | Internal |
| `com.salesmanager.core.util.*` | i18n helpers (`LabelUtil`, `MessageUtil`) | Internal |
| `com.salesmanager.central.profile.*` | Session context (`Context`) | Internal |
| `com.salesmanager.central.web.Constants` | Constant values (e.g., token names) | Internal |

All dependencies are either part of the Java EE stack or internal to the *SalesManager* application. No external build‑time libraries beyond Struts 2 and Log4j are required.

---

## 5. Additional Notes & Recommendations

### 5.1 Code‑Quality / Style

| Issue | Impact | Fix / Improvement |
|-------|--------|--------------------|
| **Raw types** – e.g., `Map languagesMap`, `Map reflanguages = new HashMap();` | Compile‑time warnings, potential `ClassCastException`. | Use generics: `Map<Integer, Language> languagesMap = new HashMap<>();` |
| **Missing `@Override` annotations** – several overridden methods lack the annotation. | Minor readability issue; risk of silent signature mismatches. | Add `@Override` to all overridden methods. |
| **Duplicate error handling** – `addFieldError`, `addActionMessage`, etc., set `actionError` but also rely on Struts’ own error collection. | Confusion; may lead to double‑displayed messages. | Prefer using Struts’ built‑in error collection (`addFieldError(...)`) and let `MessageUtil` only handle user‑facing messages. |
| **Unused `isInvalid(String)`** – dead code. | Noise. | Remove method. |
| **Potential NPE in `getContext()`** – if the session attribute `ProfileConstants.context` is missing. | Runtime crash. | Guard against null and possibly throw a descriptive exception. |
| **`init()` left commented out** – suggests incomplete implementation. | Possible bug if subclasses rely on it. | Document the contract clearly or remove the method if unused. |
| **Hard‑coded i18n keys** (`message.confirmation.success`, `errors.technical`, etc.) – risk of key typos. | Runtime `MissingResourceException`. | Centralise keys in an enum or constants class. |

### 5.2 Design / Architecture

| Concern | Comment |
|---------|---------|
| **Service Locator (`ServiceFactory`)** – introduces hidden dependencies and hinders unit testing. | Consider dependency injection (e.g., Spring) to provide `MerchantService`. |
| **Session‑based `Context`** – tightly couples actions to a specific session attribute key. | Expose a dedicated `ContextProvider` or a Struts `SessionAware` implementation. |
| **Locale handling** – overriding `getLocale()` is fragile; Struts 2 already offers a locale interceptor. | Replace with a custom Struts interceptor that sets the locale based on session or request param. |
| **Authorization helper** – simple equality check may be insufficient for role‑based security. | Move to a dedicated security component, possibly integrated with JSR‑303 or Spring Security. |

### 5.3 Performance & Thread Safety

* Actions are request‑scoped in Struts 2, so the instance fields are safe.  
* The `Logger` is static‑ish (instance per class) – fine.  
* Service lookups (`ServiceFactory.getService()`) happen on each method call; caching the service reference could reduce overhead if the factory is expensive.

### 5.4 Security

* The `authorize(IMerchant)` method throws a generic `AuthorizationException`; ensure this exception is properly handled by the application (e.g., mapped to a 403 page).  
* `setAdminTokenToSession` sets a boolean flag; ensure it is cleared appropriately to avoid session fixation.

### 5.5 Extensibility

* The class could benefit from **template methods**: e.g., `populateContext()`, `handleLocale()` that subclasses override.  
* A **message bundle** abstraction would allow the UI to read messages directly from the `MessageUtil` queue without hard‑coding keys in the action.

---

### Bottom‑Line Verdict

`BaseAction` is a pragmatic, Struts‑2‑centric helper that centralises common concerns (i18n, language, session context, validation). It works but suffers from:

* **Raw‑type usage** and **missing generics** that undermine type safety.  
* **Redundant error handling** that mixes Struts’ native mechanism with a custom `MessageUtil`.  
* **Coupling to a service locator** that hampers testability.  
* **Some fragile code** (null‑checks, hard‑coded keys, unimplemented hooks).

With a few refactorings—introducing generics, moving to dependency injection, cleaning up the validation logic, and centralising i18n keys—the class would become cleaner, safer, and easier to maintain.

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
package com.salesmanager.central;

import java.security.Principal;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.apache.log4j.Logger;
import org.apache.struts2.interceptor.PrincipalProxy;
import org.apache.struts2.interceptor.ServletRequestAware;
import org.apache.struts2.interceptor.ServletResponseAware;

import com.opensymphony.xwork2.ActionContext;
import com.opensymphony.xwork2.ActionSupport;
import com.opensymphony.xwork2.ValidationAware;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.web.Constants;
import com.salesmanager.core.entity.merchant.IMerchant;
import com.salesmanager.core.entity.merchant.MerchantRegistration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantException;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.www.SalesManagerPrincipalProxy;

public abstract class BaseAction extends ActionSupport implements
		ServletRequestAware, ServletResponseAware, ValidationAware {
	
	

	private Logger log = Logger.getLogger(BaseAction.class);
	private HttpServletRequest request;
	private HttpServletResponse response;
	
	private boolean actionError = false;

	protected Collection<Language> languages;// used in the page as an index
	protected Map<Integer, Integer> reflanguages = new HashMap();// reference
																	// count -
																	// languageId

	private String pageTitle;
	
	/**
	 * Overwrites the crappy struts 2 locale management
	 */
	public Locale getLocale() {

		Locale locale = (Locale) ActionContext.getContext().getSession().get(
				"WW_TRANS_I18N_LOCALE");
		if (locale != null && (locale instanceof Locale)) {
			ActionContext.getContext().setLocale(locale);
			return locale;
		} else {
			return super.getLocale();
		}
	}
	
	protected void setPageTitle(String key) {
		
		LabelUtil l = LabelUtil.getInstance();
		l.setLocale(getLocale());
		
		String t = l.getText(key);
		this.pageTitle = t;
		
		
	}

	protected PrincipalProxy getPrincipal() {

		HttpSession session = this.getServletRequest().getSession();
		Principal p = (Principal) session.getAttribute("PRINCIPAL");

		if (p != null) {

			SalesManagerPrincipalProxy proxy = new SalesManagerPrincipalProxy(p);
			return proxy;

		} else {
			return null;
		}

	}

	protected void prepareLanguages() {

		try {

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore mstore = mservice.getMerchantStore(getContext()
					.getMerchantid());

			Map languagesMap = mstore.getGetSupportedLanguages();

			languages = languagesMap.values();// collection reverse the map

			int count = 0;
			Iterator langit = languagesMap.keySet().iterator();
			while (langit.hasNext()) {
				Integer langid = (Integer) langit.next();
				Language lang = (Language) languagesMap.get(langid);
				reflanguages.put(count, langid);
				count++;
			}

		} catch (Exception e) {
			log.error(e);
		}
	}

	protected void setLocale(Locale locale) {
		ActionContext.getContext().setLocale(locale);
		Map sessions = ActionContext.getContext().getSession();
		sessions.put("WW_TRANS_I18N_LOCALE", locale);

		this.getServletRequest().getSession().setAttribute(
				"WW_TRANS_I18N_LOCALE", locale);
		request.setAttribute("LOCALE", locale);
	}

	protected void setSuccessMessage() {
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(getLocale());
		MessageUtil.addMessage(getServletRequest(), label
				.getText("message.confirmation.success"));
	}

	protected void setTechnicalMessage() {
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(getLocale());
		MessageUtil.addErrorMessage(getServletRequest(), label.getText(super.getLocale(),"errors.technical"));
	}

	protected void setAuthorizationMessage() {
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(getLocale());
		MessageUtil.addErrorMessage(getServletRequest(), label.getText("messages.authorization"));
	}

	protected void setMessage(String text) {
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(getLocale());
		MessageUtil.addMessage(getServletRequest(), 
				label.getText(text));
	}

	protected void setErrorMessage(String text) {
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(getLocale());
		MessageUtil.addErrorMessage(getServletRequest(), label.getText(text));
	}
	
	protected void setErrorMessage(String text, List parameters) {
		LabelUtil label = LabelUtil.getInstance();
		MessageUtil.addErrorMessage(getServletRequest(), label.getText(getLocale(),text,parameters));
	}

	protected void addErrorMessages(List<String> messages) {
		MessageUtil.addErrorMessages(getServletRequest(), messages);
	}

	protected void authorize(IMerchant entity) throws RuntimeException {
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(getLocale());
		Context ctx = (Context) getServletRequest().getSession().getAttribute(
				ProfileConstants.context);
		if (entity == null || entity.getMerchantId() != ctx.getMerchantid()) {
			MessageUtil.addErrorMessage(getServletRequest(), label.getText("messages.authorization"));
			throw new AuthorizationException();
		}

	}

	public void init() {
		Context ctx = (Context) getServletRequest().getSession().getAttribute(
				ProfileConstants.context);
		// ctx.resetFormErrorMessages();
	}

	protected Context getContext() {
		Context ctx = (Context) getServletRequest().getSession().getAttribute(
				ProfileConstants.context);
		ctx.setLang(this.getLocale().getLanguage());
		return ctx;

	}

	protected void setAdminTokenToSession(int merchantId) {
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantRegistration reg = null;
		try {
			reg = mservice.getMerchantRegistration(merchantId);
		} catch (MerchantException e) {
			log.error(e);
		}
		if (reg != null) {
			if (reg.getMerchantRegistrationDefCode() == Constants.ADMIN_MERCHANT_REG_DEF_CODE) {
				getServletRequest().getSession().setAttribute(
						Constants.ADMIN_TOKEN_PARAM, true);
			}
		}
	}

	private boolean isInvalid(String value) {
		return (value == null || value.length() == 0);
	}

	private String username;

	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	private String password;

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	public void setServletRequest(HttpServletRequest request) {
		this.request = request;
	}

	public HttpServletRequest getServletRequest() {
		return request;
	}

	public void setServletResponse(HttpServletResponse response) {
		this.response = response;
	}

	public HttpServletResponse getServletResponse() {
		return response;
	}

	public Collection<Language> getLanguages() {
		return languages;
	}

	public void setLanguages(Collection<Language> languages) {
		this.languages = languages;
	}

	public Map<Integer, Integer> getReflanguages() {
		return reflanguages;
	}

	public void setReflanguages(Map<Integer, Integer> reflanguages) {
		this.reflanguages = reflanguages;
	}

	public String getPageTitle() {
		return pageTitle;
	}
	
	/**
	 * ValidationAware
	 */
	
	public void addFieldError(String fieldName, String errorMessage) {
		actionError = true;
		MessageUtil.addErrorMessage(getServletRequest(), errorMessage);
	}
	
	public void addActionMessage(String aMessage) {
		actionError = true;
		MessageUtil.addErrorMessage(getServletRequest(), aMessage);
	}

	public void addActionError(String anErrorMessage) {
		actionError = true;
		MessageUtil.addErrorMessage(getServletRequest(), anErrorMessage);
	}
	
	public void setFieldErrors(Map errorMap) {
		actionError = true;
		log.error("setFieldErrors invoked");
	}
	
	public void setActionMessages(Collection messages) {
		actionError = true;
		log.error("setActionMessages invoked");
	}
	
	public void setActionErrors(Collection errorMessages)  {
		actionError = true;
		log.error("setActionErrors invoked");
	}

	public boolean isActionError() {
		return actionError;
	}









}



```
