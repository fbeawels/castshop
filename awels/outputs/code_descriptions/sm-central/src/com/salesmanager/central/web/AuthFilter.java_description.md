# AuthFilter.java

## Review

## 1. Summary

The `AuthFilter` is a **Servlet Filter** that protects the web tier of the SalesManager application.  
Its main responsibilities are:

| Responsibility | What it does |
|----------------|--------------|
| **Authentication** | Checks whether a user is logged in (via HTTP session or `RemoteUser`). If not, redirects to the login page. |
| **Authorization** | Matches the requested URL against a set of regular‑expression patterns (stored in a static map) to determine the required role (`level`). If the user does not belong to that role, the request is redirected to the login page. |
| **Locale handling** | Transfers the locale stored in the session (`WW_TRANS_I18N_LOCALE`) to a request attribute named `LOCALE`. |
| **Cache control** | Adds headers to prevent browser caching. |

The filter relies on the `Context` object stored in the session (key `ProfileConstants.context`) to retrieve user details such as the merchant ID.

**Notable design aspects**

* Uses **raw collections** (`Map`, `Set`, `Iterator`) without generics – a legacy style.
* The map of URL patterns to required role levels is static and **shared across all filter instances**.
* No explicit configuration mechanism is visible (e.g., init‑params for pattern registration).
* The filter is **not thread‑safe** because it writes to a static, unsynchronised map and uses non‑synchronised iterators.

---

## 2. Detailed Description

### Core components

| Component | Role |
|-----------|------|
| `patterns` (`static Map`) | Holds compiled `Pattern` objects keyed to a String representing the required role (`level`). |
| `Context` (`com.salesmanager.central.profile.Context`) | Stores user session information, including merchant ID and possibly roles. |
| `ProfileConstants.context` | Key used to fetch the `Context` from the HTTP session. |

### Execution flow

1. **Initialization**  
   The filter implements `Filter.init()` but does not populate `patterns`. It merely stores the `FilterConfig`. Thus the pattern map must be filled elsewhere (perhaps via static initialisers or by another component).

2. **Request handling (`doFilter`)**  
   a. **Extract request/response** – cast to HTTP equivalents, log the requested URL.  
   b. **Session check** – retrieve the current session.  
      * If `session == null`, the user is considered not logged in: redirect to `/index.jsp`.  
      * If `session` exists but the user is *not* authenticated (`req.getRemoteUser()` null) – same redirect.  
      * If the user is authenticated but no profile (`Context` missing) – redirect unless the request is `/logon.action`.  
   c. **Pattern matching** – iterate over all entries in `patterns`.  
      * Find the first pattern that matches the URL; retrieve its `level`.  
   d. **Authorization** – if a `level` was found, call `req.isUserInRole(level)`; if false, redirect to login.  
   e. **Locale handling** – copy the session locale into a request attribute.  
   f. **Cache control** – set HTTP headers to disable caching.  
   g. **Chain continuation** – invoke the next element in the filter chain.  

3. **Cleanup** – `destroy()` simply clears the reference to `FilterConfig`.

### Assumptions & constraints

* The application uses container‑managed authentication (the presence of `req.getRemoteUser()` implies a container login mechanism).  
* The mapping between URLs and required roles is static and immutable during runtime.  
* The `Context` bean is always stored under `ProfileConstants.context` when a user is fully authenticated.  
* The filter is thread‑safe only if the static map is never mutated after construction.

---

## 3. Functions / Methods

| Method | Signature | Purpose | Inputs | Outputs / Side‑effects |
|--------|-----------|---------|--------|------------------------|
| `init(FilterConfig)` | `void` | Stores the filter configuration. | `FilterConfig` | Sets the instance variable `filterConfig`. |
| `destroy()` | `void` | Cleans up references. | – | Sets `filterConfig` to `null`. |
| `doFilter(ServletRequest, ServletResponse, FilterChain)` | `void` | Main entry point. Handles authentication, authorization, locale setting, cache control, and forwards the request. | `ServletRequest`, `ServletResponse`, `FilterChain` | Redirects or forwards the request, sets attributes, modifies response headers. |

### Utility logic inside `doFilter`

* **Pattern lookup** – loops over `patterns.keySet()` to find a matching URL.  
* **Role check** – uses `HttpServletRequest.isUserInRole(String)` to verify permissions.  
* **Locale transfer** – `request.setAttribute("LOCALE", locale)`.

No other public or private helper methods are present.

---

## 4. Dependencies

| Library / API | Type | Role |
|---------------|------|------|
| `javax.servlet.*` | Java EE (now Jakarta EE) | Filter interface, request/response/session handling. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.central.profile.Context` | Internal | Holds user session data. |
| `com.salesmanager.central.profile.ProfileConstants` | Internal | Holds session attribute keys. |

*All dependencies are standard for a Java EE web application; no external frameworks such as Spring are used.*

---

## 5. Additional Notes

### Edge Cases & Potential Issues

1. **Uninitialised `patterns` map**  
   The map is never populated in this class; if no other component fills it, all requests will bypass authorization.  
2. **Thread‑safety**  
   Static, unsynchronised map + raw iterators make the filter **not thread‑safe**. Concurrent requests could lead to `ConcurrentModificationException` or inconsistent state.  
3. **Raw types & unchecked casts**  
   The code uses raw `Map`, `Set`, and `Iterator` which generate compiler warnings and can lead to `ClassCastException`.  
4. **Pattern matching logic**  
   Uses `Matcher.find()` – this will match any part of the URL, potentially leading to false positives. `matches()` would be stricter.  
5. **Redirection logic**  
   The filter always redirects to `/index.jsp` on failures, even for `GET /index.jsp`. This can cause a redirect loop if the login page also triggers the filter.  
6. **Locale fallback**  
   If `WW_TRANS_I18N_LOCALE` is missing from the session, the request attribute will be `null`. The rest of the application must handle that.  
7. **Hard‑coded paths**  
   The login page path (`/index.jsp`) is hard‑coded; a more flexible configuration (init‑params) would be preferable.  

### Suggested Enhancements

| Category | Recommendation |
|----------|----------------|
| **Configuration** | Load URL‑role patterns from a properties file or servlet context init‑params during `init()`. |
| **Thread‑safety** | Replace raw `Map` with `ConcurrentHashMap`, use generics (`Map<Pattern, String>`), and avoid mutable static state. |
| **Logging** | Migrate to SLF4J (or the application’s preferred logging facade) to allow better log level management. |
| **Pattern Matching** | Use `Pattern.compile("...")` with `^...$` anchors and `matcher.matches()` for stricter checks. |
| **Authorization** | Consider using a dedicated security framework (Spring Security, Apache Shiro) for role/permission handling. |
| **Redirect handling** | Use a dedicated login URL configured via init‑param; avoid redirect loops by checking the current target path. |
| **Locale handling** | Provide a default locale if session attribute is missing, or use the request’s `getLocale()` as a fallback. |
| **Code quality** | Apply generics, remove unused variables (`l0`), document the expected content of `patterns`, and add unit tests for the filter logic. |

---

### Final Verdict

The `AuthFilter` implements a straightforward, container‑managed security guard for the application. While functional, the code reflects older Java EE practices (raw collections, manual thread‑safety concerns) and lacks modern security frameworks. It would benefit from a redesign that externalises configuration, improves thread safety, and integrates with a proven security framework to reduce boilerplate and potential bugs.

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
package com.salesmanager.central.web;

import java.io.IOException;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Locale;
import java.util.Map;
import java.util.Set;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.apache.log4j.Logger;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;

/**
 * Filter for managing authentication/autorization
 * 
 */
public class AuthFilter implements Filter {

	private static Map patterns = new HashMap();// contains lvl 1 to lvl x

	private static Pattern l0 = null;



	private Logger log = Logger.getLogger(AuthFilter.class);
	private FilterConfig filterConfig = null;

	public void init(FilterConfig filterConfig) throws ServletException {
		this.filterConfig = filterConfig;
	}

	public void destroy() {
		this.filterConfig = null;
	}

	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {

		HttpServletRequest req = (HttpServletRequest) request;
		HttpServletResponse resp = (HttpServletResponse) response;
		String url = req.getRequestURI();
		log.debug("Requested URL " + url);

		// check if user is logged in
		HttpSession session = req.getSession();

		if (session == null) {
			log.debug("Session is null");

			String username = req.getRemoteUser();
			if (username == null) {// not logged in
				resp.sendRedirect(req.getContextPath() + "/index.jsp");
				return;
			} else {// logged in, but need to retreive the profile
				if (url.contains("/logon.action")) {
					chain.doFilter(request, response);
					return;
				} else {
					resp.sendRedirect(req.getContextPath() + "/index.jsp");
					return;
				}
			}
		}

		// check if session is expired
		Context ctx = (Context) session.getAttribute(ProfileConstants.context);
		if (ctx == null) {
			log.debug("Context is null");

			String username = req.getRemoteUser();
			if (username == null) {// not logged in
				resp.sendRedirect(req.getContextPath() + "/index.jsp");
				return;
			} else {// logged in, but need to retreive the profile
				if (url.contains("/logon.action")) {
					log.debug("In logon");
					chain.doFilter(request, response);
					return;
				} else {
					resp.sendRedirect(req.getContextPath() + "/index.jsp");
					return;
				}
			}
		}

		Set patternsets = patterns.keySet();
		Iterator patterniterator = patternsets.iterator();
		String level = null;
		while (patterniterator.hasNext()) {
			Pattern p = (Pattern) patterniterator.next();
			Matcher m = p.matcher(url);
			if (m.find()) {
				// get the associated level
				level = (String) patterns.get(p);
				break;
			}
		}

		if (level != null) {
			if (!req.isUserInRole(level)) {
				log.debug("User " + ctx.getMerchantid()
						+ " not authorized for url " + url);
				resp.sendRedirect(req.getContextPath() + "/index.jsp");
				return;
			}
		} // else let go

		// set locale in the request
		Locale locale = (Locale) req.getSession().getAttribute(
				"WW_TRANS_I18N_LOCALE");
		request.setAttribute("LOCALE", locale);



		// no browser cache
		resp.setHeader("Cache-Control", "no-cache");
		resp.setHeader("Pragma", "no-cache");
		resp.setDateHeader("Expires", -1);

		chain.doFilter(request, response);
	}

}


```
