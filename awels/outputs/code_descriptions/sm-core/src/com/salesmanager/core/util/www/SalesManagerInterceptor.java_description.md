# SalesManagerInterceptor.java

## Review

## 1. Summary

**Purpose & Functionality**  
`SalesManagerInterceptor` is a Struts 2 interceptor that runs before every action in the SalesManager application. Its responsibilities are:

1. **Merchant Store resolution** – Determine the active `MerchantStore` (the shop or vendor) from request parameters, session, or cookie and load it into the request/session.
2. **Locale configuration** – Set the locale for the current request based on the store’s settings.
3. **Principal handling** – Wrap the authenticated `Principal` in a proxy and expose it to actions that implement `PrincipalAware`.
4. **Hook for subclass logic** – Delegates additional, application‑specific logic to an abstract `baseIntercept` method that concrete subclasses must implement.

**Key Components**

| Component | Role |
|-----------|------|
| `ServiceFactory` | Provides access to business services (`MerchantService`, `ReferenceService`). |
| `MerchantStore` | Represents a merchant’s configuration (template module, country, etc.). |
| `LocaleUtil` | Helper to set the locale on a request. |
| `RefCache` | Caches reference data; the interceptor ensures it’s loaded. |
| `SalesManagerPrincipalProxy` | Wraps the `Principal` so that actions can access application‑specific user data. |

**Design Patterns / Frameworks**

* **Interceptor** – classic Struts 2 interceptor pattern.  
* **Factory** – `ServiceFactory` is a simple service locator.  
* **Proxy** – `SalesManagerPrincipalProxy` acts as a façade over the raw `Principal`.  
* **Singleton** – `RefCache` is a global cache.

---

## 2. Detailed Description

### Execution Flow

1. **Request/Response Acquisition**  
   - Obtains the `HttpServletRequest` and `HttpServletResponse` from the Struts 2 context.
   - Forces UTF‑8 request encoding.

2. **Cookie Collection**  
   - Builds a `Map<String, Cookie>` (`cookiesMap`) from the request cookies for quick lookup.

3. **Merchant ID Resolution**  
   - Tries to read `merchantId` from the query string.
   - If absent, looks for a `STORE` cookie or a session attribute named `STORE`.
   - If still unknown, falls back to `Constants.DEFAULT_MERCHANT_ID`.

4. **Store Loading**  
   - Calls `setMerchantStore(...)` whenever a merchantId is determined or the existing session store does not match.
   - `setMerchantStore` loads the `MerchantStore` via `MerchantService`, stores it in the session and request, sets a `STORE` cookie, and populates a session‑level configuration map from `ReferenceService`.

5. **Template & Locale**  
   - Validates that the store has a `templateModule`; otherwise aborts with `NOSTORE`.
   - Stores the template ID in the request.
   - Calls `LocaleUtil.setLocaleForRequest(...)` to adjust the locale.

6. **Principal Proxying**  
   - If a `Principal` is present in the session, creates a `SalesManagerPrincipalProxy` and injects it into the action via a custom `BaseActionAware` interface.

7. **Subclass Hook**  
   - Invokes the abstract `baseIntercept`.  If it returns a non‑null string, that result is returned immediately.

8. **Proceed**  
   - Otherwise, delegates to the next interceptor or action via `invoke.invoke()`.

9. **Exception Handling**  
   - Catches any exception, logs it, registers a generic action error, and returns either `Action.ERROR` or `GENERICERROR` depending on the exception type.

### Assumptions & Constraints

| Assumption | Reasoning |
|------------|-----------|
| `MerchantStore` will always be present in the session after the first request. | Enables quick lookup for subsequent requests. |
| `Constants.DEFAULT_MERCHANT_ID` is a valid fallback store. | Simplifies error handling when no merchant is specified. |
| All actions requiring a `Principal` implement `BaseActionAware`. | Allows safe injection of the proxy. |
| `setMerchantStore` always succeeds if a valid `merchantId` is provided. | The code returns `null` only when the merchant is not found. |

### Architecture & Design Choices

* **Single interceptor for multiple concerns** – The class bundles store resolution, locale, and principal handling.  While convenient, it violates the *Single Responsibility Principle*; each aspect could live in its own interceptor for better testability.
* **Hard‑coded cookie name (`STORE`)** – The cookie name is hard‑coded; a constant would make future changes easier.
* **Use of raw types** – `Map` and `List` are used without generics, which risks `ClassCastException` and erases type safety.
* **Service lookup** – `ServiceFactory` is used directly, coupling the interceptor tightly to that locator pattern.

---

## 3. Functions / Methods

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `destroy()` | Lifecycle hook; currently no-op. | – | – | – |
| `init()` | Lifecycle hook; currently no-op. | – | – | – |
| `intercept(ActionInvocation)` | Main interceptor logic; orchestrates all other steps. | `ActionInvocation invoke` | `String` result code | Sets request attributes, cookies, session attributes, and potentially aborts. |
| `setMerchantStore(HttpServletRequest, HttpServletResponse, String)` | Loads a `MerchantStore` for a given `merchantId` and stores it in session/request. | `HttpServletRequest req`, `HttpServletResponse resp`, `String merchantId` | `MerchantStore` or `null` | Sets session attribute `STORE`, request attribute `STORE`, cookie `STORE`, and session attribute `STORECONFIGURATION`. |
| `baseIntercept(ActionInvocation, HttpServletRequest, HttpServletResponse)` | Abstract hook for subclasses to inject custom logic. | `ActionInvocation invoke`, `HttpServletRequest req`, `HttpServletResponse resp` | `String` result or `null` | Subclass‑defined. |

### Reusable / Utility Methods

* The cookie handling loop could be refactored into a static utility (`CookieUtils.mapFrom(Cookie[])`) to avoid duplication in future interceptors.
* `setMerchantStore` could be extracted to a `MerchantStoreResolver` service to decouple the interceptor from business logic.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.struts2.ServletActionContext` | Framework | Provides access to request/response. |
| `org.apache.struts2.interceptor.PrincipalAware` | Framework | Interface for principal injection. |
| `com.opensymphony.xwork2.*` | Framework | Core Struts 2 classes (`Action`, `ActionContext`, etc.). |
| `org.apache.commons.lang.StringUtils` | Third‑party | Utility for string handling. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.core.*` | Internal | Various services (`MerchantService`, `ReferenceService`), entities (`MerchantStore`, `Language`), utilities, and constants. |
| `javax.servlet.*` | Servlet API | Request, response, session, cookie handling. |

All dependencies are either standard Java EE APIs or well‑known third‑party libraries. No platform‑specific constraints beyond a servlet container.

---

## 5. Additional Notes

### Edge Cases & Limitations

1. **Null `merchantId` vs. Blank** – The code checks `StringUtils.isBlank(merchantId)` and then again `if (merchantId == null)` inside that block. The inner check is redundant and can mis‑lead developers.
2. **Parsing Failure** – `setMerchantStore` silently logs a parsing error and proceeds with `iMerchantId = 1`. This could mask user errors and lead to unexpected store resolution.
3. **Missing Store in Session** – If the store is not present in the session and the cookie value is invalid, the interceptor falls back to the default store, potentially masking configuration errors.
4. **Raw Types** – Using raw `Map`/`List` removes compile‑time type safety; modern Java would prefer generics (`Map<String, Cookie>`, `List<Cookie>`).
5. **Error Path Handling** – Returning `"NOSTORE"` or `"GENERICERROR"` assumes that the action mapping contains those results. If not, the user will see a 404/500.
6. **Hard‑coded Cookie Max‑Age** – The `STORE` cookie is set to 1 year; this may not be appropriate for all deployments.
7. **Locale Setting** – `LocaleUtil.setLocaleForRequest` is called unconditionally; if the store is null, a `NullPointerException` would be thrown earlier (the code checks for `store` after setting the attribute though).

### Potential Improvements

| Category | Recommendation |
|----------|----------------|
| **Separation of Concerns** | Split this interceptor into three: `MerchantStoreInterceptor`, `LocaleInterceptor`, `PrincipalInterceptor`. Each would be simpler and testable. |
| **Generics & Type Safety** | Replace raw collections with generics (`Map<String, Cookie>`, `List<Cookie>`) and add appropriate type checks. |
| **Configuration** | Move hard‑coded values (`"STORE"`, cookie age, default merchant id) to constants or configuration files. |
| **Error Handling** | Throw a custom checked exception (e.g., `MerchantNotFoundException`) instead of returning `null`. Let Struts map it to a proper error page. |
| **Logging** | Use parameterized logs (`log.error("Cannot parse merchantId to Integer {}", merchantId)`) to avoid string concatenation. |
| **Service Layer** | Inject services via dependency injection (e.g., Spring) instead of using a static `ServiceFactory`. |
| **Cache Initialization** | `RefCache` should be initialized during application startup rather than lazily in the interceptor. |
| **Testing** | Create unit tests that mock `HttpServletRequest`, `HttpServletResponse`, and `ActionInvocation` to verify each branch of the logic. |
| **Documentation** | Add Javadoc to public methods, especially `intercept` and `setMerchantStore`, to explain contract and side‑effects. |

---

### Summary of Strengths

* Centralizes common pre‑action logic that is reused across many actions.  
* Leverages Struts 2’s interceptor mechanism cleanly.  
* Provides a convenient hook (`baseIntercept`) for subclasses, keeping the core logic DRY.  

### Summary of Weaknesses

* Mixes several responsibilities in one class, making it bulky and harder to maintain.  
* Uses raw types and hard‑coded strings, reducing readability and type safety.  
* Error handling is simplistic; ambiguous return codes can lead to confusing user experience.  
* Tight coupling to a static service locator and a global cache undermines testability.  

By refactoring along the lines suggested above, the interceptor would become more modular, safer, and easier to evolve as the application grows.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 24, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.util.www;

import java.security.Principal;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.apache.struts2.ServletActionContext;
import org.apache.struts2.interceptor.PrincipalAware;

import com.opensymphony.xwork2.Action;
import com.opensymphony.xwork2.ActionContext;
import com.opensymphony.xwork2.ActionInvocation;
import com.opensymphony.xwork2.ActionSupport;
import com.opensymphony.xwork2.interceptor.Interceptor;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.ReferenceUtil;

public abstract class SalesManagerInterceptor implements Interceptor {

	private Logger log = Logger.getLogger(SalesManagerInterceptor.class);

	public void destroy() {
		// TODO Auto-generated method stub

	}

	public SalesManagerInterceptor() {
	}

	public void init() {
		// TODO Auto-generated method stub

	}

	public String intercept(ActionInvocation invoke) throws Exception {
		try {

			HttpServletRequest req = (HttpServletRequest) ServletActionContext
					.getRequest();
			HttpServletResponse resp = (HttpServletResponse) ServletActionContext
					.getResponse();
			
			req.setCharacterEncoding("UTF-8");

			// get cookies
			Map cookiesMap = new HashMap();
			Cookie[] cookies = req.getCookies();
			if (cookies != null) {
				for (int i = 0; i < cookies.length; i++) {
					Cookie cookie = cookies[i];
					cookiesMap.put(cookie.getName(), cookie);
				}
			}

			/**
			 * MERCHANT ID
			 */

			// look at merchantId in url parameter
			String merchantId = (String) req.getParameter("merchantId");

			Cookie storeCookie = (Cookie) cookiesMap.get("STORE");

			int iMerchantId = Constants.DEFAULT_MERCHANT_ID;
			MerchantStore store = null;

			if (StringUtils.isBlank(merchantId)) {// no merchantId in the
													// request

				// check for store
				store = (MerchantStore) req.getSession().getAttribute("STORE");

				if (merchantId == null) {

					if (store != null) {

						iMerchantId = store.getMerchantId();
					} else {
						// check for cookie
						Cookie c = (Cookie) cookiesMap.get("STORE");
						if (c != null && !StringUtils.isBlank(c.getValue())) {

							String v = c.getValue();
							iMerchantId = Integer.valueOf(v);
						} else {
							// assign defaultMerchantId
							iMerchantId = Constants.DEFAULT_MERCHANT_ID;
						}
						// set store
						store = this.setMerchantStore(req, resp, merchantId);
						if (store == null) {
							return "NOSTORE";
						}
					}
				}

			} else {// merchantId in the request

				// check for store in the session
				store = (MerchantStore) req.getSession().getAttribute("STORE");

				if (store != null) {
					// check if both match
					if (!merchantId.equals(String
							.valueOf(store.getMerchantId()))) {// if they differ
						store = this.setMerchantStore(req, resp, merchantId);
					} else {
						iMerchantId = store.getMerchantId();
					}

				} else {
					// set store
					store = this.setMerchantStore(req, resp, merchantId);
					if (store == null) {
						return "NOSTORE";
					}
				}
			}

			req.setAttribute("STORE", store);

			if (StringUtils.isBlank(store.getTemplateModule())) {
				return "NOSTORE";
			}

			req.setAttribute("templateId", store.getTemplateModule());

			ActionContext ctx = ActionContext.getContext();
			LocaleUtil.setLocaleForRequest(req, resp, ctx, store);

			HttpSession session = req.getSession();
			Principal p = (Principal) session.getAttribute("PRINCIPAL");

			if (p != null) {
				try {
					SalesManagerPrincipalProxy proxy = new SalesManagerPrincipalProxy(
							p);
					BaseActionAware action = ((BaseActionAware) invoke.getAction());
					action.setPrincipalProxy(proxy);
				} catch (Exception e) {
					log
							.error("The current action does not implement PrincipalAware "
									+ invoke.getAction().getClass());
				}
			}

			String r = baseIntercept(invoke, req, resp);
			if (r != null) {
				return r;
			}

			return invoke.invoke();

		} catch (Exception e) {
			log.error(e);
			ActionSupport action = (ActionSupport) invoke.getAction();
			action.addActionError(action.getText("errors.technical") + " "
					+ e.getMessage());
			if (e instanceof ActionException) {
				return Action.ERROR;
			} else {
				return "GENERICERROR";
			}

		}

	}

	private MerchantStore setMerchantStore(HttpServletRequest req,
			HttpServletResponse resp, String merchantId) throws Exception {

		// different merchantId
		int iMerchantId = 1;

		try {
			iMerchantId = Integer.parseInt(merchantId);
		} catch (Exception e) {
			log.error("Cannot parse merchantId to Integer " + merchantId);
		}

		// get MerchantStore
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore mStore = mservice.getMerchantStore(iMerchantId);

		if (mStore == null) {
			// forward to error page
			log.error("MerchantStore does not exist for merchantId "
					+ merchantId);
			return null;
		}

		req.getSession().setAttribute("STORE", mStore);
		req.setAttribute("STORE", mStore);
		
		//get store configuration for template
		ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
		Map storeConfiguration = rservice.getModuleConfigurationsKeyValue(
					mStore.getTemplateModule(), mStore.getCountry());
	
		if (storeConfiguration != null) {
				req.getSession().setAttribute("STORECONFIGURATION",
						storeConfiguration);
		}

		Cookie c = new Cookie("STORE", merchantId);
		c.setMaxAge(365 * 24 * 60 * 60);
		resp.addCookie(c);
		
		if(!RefCache.isLoaded()) {
			RefCache.createCache();
		}

		return mStore;

	}

	protected abstract String baseIntercept(ActionInvocation invoke,
			HttpServletRequest req, HttpServletResponse resp) throws Exception;

}



```
