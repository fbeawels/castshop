# AuthFilter.java

## Review

## 1. Summary  

**Purpose**  
The `AuthFilter` is a generic servlet filter that enforces authentication and (optionally) authorization for all HTTP requests processed by a web application. It intercepts every request, determines whether the user is already logged in, handles session expiration, and redirects unauthenticated users to a log‑on page.

**Key components**  
| Component | Role |
|-----------|------|
| `AuthFilter` | Abstract base class implementing `javax.servlet.Filter`. Concrete subclasses supply the actual authentication logic via three abstract methods: `getUser`, `getLogonPage`, and `bypassUrl`. |
| `doFilter()` | Central logic that examines the incoming request, session state, and target URL to decide whether to pass the request through the filter chain or redirect. |
| `Constants.CONTEXT` | Session attribute key that indicates a valid user context. |
| `StringUtils.isBlank` | Utility method from Apache Commons Lang used to test for an empty username parameter. |

**Notable design patterns / libraries**  
* **Template Method Pattern** – `doFilter()` provides the common algorithm while delegating authentication‑specific parts to abstract methods.  
* **Filter Pattern** – Standard servlet filter that is part of the Java EE web‑app architecture.  
* **Apache Commons Lang** – Used for null/empty string handling.  
* **Log4j** – For debug logging.  

---

## 2. Detailed Description  

### Execution Flow  
1. **Initialization** – `init(FilterConfig)` stores the filter config for later use; `destroy()` clears the reference.  
2. **Request Interception** – `doFilter()` casts the generic `ServletRequest/Response` to their HTTP counterparts, retrieves the request URI and logs it.  
3. **Character Encoding** – Request encoding is forced to `UTF8`.  
4. **Bypass Logic** – Calls the abstract `bypassUrl()` method; if it returns `true`, the request is passed to the next filter or servlet immediately.  
5. **Session & User Presence**  
   * If `session == null`, it attempts to retrieve the username via `getUser(req)`.  
   * If no username is found, the filter checks if the user is attempting a log‑on (`/logon.action`).  
     * If yes → let it through.  
     * If no → redirect to the log‑on page (`getLogonPage(req)`).  
   * If a username *is* found, the same logic applies: if the request is a log‑on action it passes through; otherwise it redirects to the log‑on page (this block is effectively redundant).  
6. **Session Expiration Check** – Retrieves the attribute `Constants.CONTEXT` from the session.  
   * If missing, the same login‑check logic as above is executed.  
   * If present, the request is considered authenticated and the filter chain proceeds.  
7. **Filter Chain Continuation** – If none of the above redirects, `chain.doFilter()` is invoked.

### Design Choices  
* **Abstract Delegation** – Allows different authentication providers (e.g., database, LDAP, token) to plug in by overriding the three abstract methods.  
* **Centralized Redirect** – All unauthenticated access ends up at a single log‑on page, ensuring a uniform user experience.  
* **Hardcoded URL Checks** – The logic explicitly looks for `"/logon.action"` in the request URI. This couples the filter to a particular URL scheme.  

### Assumptions & Constraints  
* A user is considered authenticated only if the session contains an attribute keyed by `Constants.CONTEXT`.  
* The filter trusts the presence of a `username` parameter to indicate a login attempt.  
* The application uses Log4j for logging and Apache Commons Lang for string handling.  
* The filter is designed for a servlet‑based web application (no Servlet 3.0+ annotations).  

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑effects |
|--------|-----------|---------|--------|---------|--------------|
| `init(FilterConfig)` | `public void init(FilterConfig filterConfig)` | Store filter configuration for later use. | `FilterConfig` | None | Saves config to instance field |
| `destroy()` | `public void destroy()` | Clean up configuration reference. | None | None | Nullifies config field |
| `doFilter(ServletRequest, ServletResponse, FilterChain)` | `public void doFilter(ServletRequest, ServletResponse, FilterChain)` | Main authentication flow. | `request`, `response`, `chain` | None (either forwards or redirects) | May redirect or call `chain.doFilter` |
| `getUser(HttpServletRequest)` | `abstract String getUser(HttpServletRequest request)` | Sub‑class implementation that extracts the username from the request (or session). | `HttpServletRequest` | Username string or `null` | No side‑effects |
| `getLogonPage(HttpServletRequest)` | `abstract String getLogonPage(HttpServletRequest request)` | Sub‑class implementation that returns the URL of the log‑on page. | `HttpServletRequest` | URL string | No side‑effects |
| `bypassUrl(HttpServletRequest, HttpServletResponse, FilterChain)` | `abstract boolean bypassUrl(HttpServletRequest request, HttpServletResponse response, FilterChain chain)` | Sub‑class implementation that decides if the current request should bypass authentication (e.g., static resources, public APIs). | `request`, `response`, `chain` | `true`/`false` | May call `chain.doFilter` if it chooses to bypass |

### Utility Points  
* `StringUtils.isBlank()` is used to avoid `NullPointerException` when checking the `username` request parameter.  
* The filter logs every requested URL at DEBUG level, which aids troubleshooting but may clutter logs in production.

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `javax.servlet` (`Filter`, `FilterChain`, `FilterConfig`, etc.) | Standard API | Core servlet filter infrastructure |
| `javax.servlet.http` (`HttpServletRequest`, `HttpServletResponse`, `HttpSession`) | Standard API | HTTP‑specific request/response handling |
| `org.apache.commons.lang.StringUtils` | Third‑party | String utility methods |
| `org.apache.log4j.Logger` | Third‑party | Logging framework |
| `com.salesmanager.core.constants.Constants` | In‑house | Holds the `CONTEXT` key used for session validation |
| `com.salesmanager.core.security.AuthFilter` | In‑house | The filter itself |

No external services or databases are invoked directly by this class; all such interactions are expected to be implemented in the concrete subclasses.

---

## 5. Additional Notes  

### Strengths  
* **Extensibility** – The template‑method design lets developers plug in diverse authentication mechanisms without touching the core filter logic.  
* **Centralization** – All authentication checks and redirects are handled in one place, simplifying maintenance.  
* **Clear Separation** – By splitting session validation and user extraction into abstract methods, the class remains agnostic to the underlying user store.

### Weaknesses / Redundancies  
1. **Duplicate Logic** – The sections handling `session == null` and the session‑expiration block are almost identical; this duplication increases maintenance risk and could be consolidated.  
2. **Hardcoded URL Fragment** – Checking for `"/logon.action"` ties the filter to a specific endpoint. A more flexible approach would be to parameterize the allowed login action or use a configuration file.  
3. **Unnecessary Bypass Call** – The `bypassUrl()` method receives `FilterChain` and `HttpServletResponse` but the current implementation merely returns a boolean. If the bypass implementation intends to manipulate the chain, the design should be clarified or the signature simplified.  
4. **Encoding Setting** – `req.setCharacterEncoding("UTF8")` may not be sufficient for multi‑locale support; consider using a dedicated filter for encoding or configuring the connector.  
5. **Logging Verbosity** – Debug‑level logging of every URL may produce large log files; a more selective strategy (e.g., only failed auth attempts) could be preferable.  
6. **Null Session Handling** – The `session` variable is obtained via `req.getSession()` which will create a new session if one does not exist. The code then checks for `session == null`, which will never be `true`. The intention was likely to use `req.getSession(false)` to avoid session creation.  
7. **Missing Exception Handling** – `doFilter()` can throw `IOException` or `ServletException`; the current logic assumes no failure when redirecting. Defensive checks could be added around `resp.sendRedirect()`.

### Edge Cases & Scenarios Not Handled  
* **CSRF** – The filter does not guard against cross‑site request forgery.  
* **HTTPS Enforcement** – No redirect to HTTPS or validation of secure connections.  
* **Concurrent Sessions** – The filter does not prevent session fixation or duplicate logins.  
* **Ajax Requests** – Redirects may not be appropriate for asynchronous calls; an alternative (e.g., HTTP 401) could be better.  
* **REST APIs** – The filter assumes a web‑page flow; API endpoints may need token‑based authentication instead.

### Future Enhancements  
1. **Refactor to Remove Duplication** – Create a private method that encapsulates the “is user logged in?” logic.  
2. **Introduce a Configuration File** – Store login URL patterns and bypass patterns externally to avoid hardcoding.  
3. **Support Multiple Auth Strategies** – Allow multiple authentication handlers (e.g., SSO, OAuth) via a strategy pattern.  
4. **Improve Session Handling** – Use `getSession(false)` to avoid creating unused sessions.  
5. **Add AJAX Support** – Detect `X-Requested-With: XMLHttpRequest` and respond with 401 or JSON rather than redirect.  
6. **Security Hardening** – Implement CSRF tokens, enforce HTTPS, and rotate session IDs upon login.  
7. **Unit Tests** – Provide comprehensive JUnit tests for each logical branch, mocking the request/response objects.

--- 

**Verdict**  
The `AuthFilter` provides a solid, reusable foundation for request authentication in a servlet‑based Java web application. With some refactoring to eliminate duplication, loosening hardcoded URL dependencies, and addressing a few security considerations, it can serve as a robust, production‑ready component.

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
package com.salesmanager.core.security;

import java.io.IOException;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.Constants;

/**
 * Filter for managing authentication/autorization
 * 
 */
public abstract class AuthFilter implements Filter {

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
		
		req.setCharacterEncoding("UTF8");

		// check if user is logged in
		HttpSession session = req.getSession();
		
		if(bypassUrl(req, resp, chain)) {
			chain.doFilter(request, response);
			return;
		}

		if (session == null) {

			String username = getUser(req);

			if (username == null) {// not logged in

				if (!StringUtils.isBlank(req.getParameter("username"))) {// submiting
																			// login
					if (url.contains("/logon.action")) {
						chain.doFilter(request, response);
						return;
					} else {
						// resp.sendRedirect(req.getContextPath()+"/index.jsp");
						resp.sendRedirect(getLogonPage(req));
						return;
					}
				} else {

					resp.sendRedirect(getLogonPage(req));
					return;
				}

			} else {// logged in, but need to retreive the profile
				if (url.contains("/logon.action")) {
					chain.doFilter(request, response);
					return;
				} else {

					resp.sendRedirect(getLogonPage(req));
					return;
				}
			}
		}

		// check if session is expired

		Object o = session.getAttribute(Constants.CONTEXT);
		if (o == null) {

			String username = getUser(req);

			if (username == null) {// not logged in
				if (!StringUtils.isBlank(req.getParameter("username"))) {// submiting
																			// login
					if (url.contains("/logon.action")) {
						chain.doFilter(request, response);
						return;
					} else {

						resp.sendRedirect(getLogonPage(req));
						return;
					}
				} else {

					resp.sendRedirect(getLogonPage(req));
					return;
				}
			} else {// logged in, but need to retreive the profile
				if (url.contains("/logon.action")) {
					chain.doFilter(request, response);
					return;
				} else {

					resp.sendRedirect(getLogonPage(req));
					return;
				}
			}
		}

		chain.doFilter(request, response);
	}

	abstract String getUser(HttpServletRequest request);

	abstract String getLogonPage(HttpServletRequest request);
	
	abstract boolean bypassUrl(HttpServletRequest request, HttpServletResponse response, FilterChain chain);

}


```
