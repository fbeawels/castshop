# JAASCustomerSecurityFilter.java

## Review

## 1. Summary  
**Purpose & Functionality**  
`JAASCustomerSecurityFilter` is a servlet filter that protects customer‑specific URLs by ensuring that a request either carries a valid customer authentication token or has an active customer session. If neither condition is met, the user is redirected to the sign‑in page.  

**Key Components**  
| Component | Role |
|-----------|------|
| `CustomerLogonModule` | Validates the `customerAuthToken` supplied as a request parameter. |
| `SecurityConstants` | Holds the session attribute key (`SM_CUSTOMER_USER`) that marks a logged‑in customer. |
| `escapeActionList` | List of URL patterns that should bypass authentication (e.g., log‑in, log‑out). |
| `AuthFilter` | The superclass (not shown) that likely provides common filter logic such as `init` delegation. |

**Design Patterns & Libraries**  
* **Servlet Filter** – standard Java EE component for request preprocessing.  
* **Spring IoC** – used to retrieve the `customerlogon` bean.  
* **Apache Commons Lang** – `StringUtils.isBlank` for safe string handling.  
* **Log4j** – logging framework (though currently only a private logger is declared).  
* **Template Method** – `getLogonPage` and `getUser` override abstract methods from `AuthFilter`.

---

## 2. Detailed Description  
1. **Initialization**  
   * `init(FilterConfig)` delegates to the parent (`AuthFilter.init`) and then obtains a reference to the `CustomerLogonModule` bean from the Spring context.  
   * The filter’s instance variables are otherwise immutable, ensuring thread‑safety.

2. **Request Handling (`doFilter`)**  
   * **Cache‑control**: Sets headers to prevent caching of protected resources.  
   * **URL Escaping**: If the request URI matches one of the `escapeActionList` items or ends with `.css` / `.js`, the filter simply forwards the request downstream (`chain.doFilter`).  
   * **Authentication Flow**  
     * Checks for a `customerAuthToken` request parameter.  
     * **No token** – retrieves the current HTTP session (`req.getSession()` which creates one if missing).  
       * If the session does not contain `SM_CUSTOMER_USER`, the user is redirected to the sign‑on page.  
       * Otherwise the request proceeds.  
     * **With token** – delegates to `logonModule.isValidAuthToken`. If validation fails, the filter returns HTTP 401 Unauthorized.  

3. **Utility Methods**  
   * `isEscapeUrlFromFilter` performs a simple substring match on each escape URL.  
   * `bypassUrl` is a placeholder that currently returns `false`; likely intended for subclasses to override.

4. **Assumptions & Constraints**  
   * Sessions are considered valid simply by the presence of the `SM_CUSTOMER_USER` attribute; no expiration logic is present.  
   * Authentication tokens are expected to be passed as request parameters, not cookies or headers.  
   * The filter is not aware of HTTPS or secure cookie handling.  

5. **Architecture & Design Choices**  
   * The filter’s logic is straightforward and relies heavily on the external `CustomerLogonModule`.  
   * The decision to use a request parameter for tokens keeps the implementation stateless but exposes the token in URLs, which can be less secure.  
   * Using a `List<String>` for escape URLs keeps the list immutable and easy to extend.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `init(FilterConfig)` | Initializes the filter; obtains Spring bean. | `FilterConfig` | None | Sets `logonModule` |
| `getLogonPage(HttpServletRequest)` | Returns the sign‑in page URL. | `HttpServletRequest` | `String` | None |
| `getUser(HttpServletRequest)` | Retrieves logged‑in customer username from session. | `HttpServletRequest` | `String` | None |
| `doFilter(ServletRequest, ServletResponse, FilterChain)` | Core request processing; enforces authentication. | `ServletRequest`, `ServletResponse`, `FilterChain` | None (may redirect or send 401) | Sets response headers, may redirect, may forward |
| `isEscapeUrlFromFilter(String)` | Checks if a URI should bypass authentication. | `String` (URI) | `boolean` | None |
| `bypassUrl(HttpServletRequest, HttpServletResponse, FilterChain)` | Placeholder for subclasses; currently unused. | `HttpServletRequest`, `HttpServletResponse`, `FilterChain` | `boolean` (always `false`) | None |

**Reusable/Utility Methods**  
* `isEscapeUrlFromFilter` can be reused by other filters that need similar escape logic.  
* `getLogonPage` and `getUser` could be abstracted into a common authentication helper if multiple filters share the same behaviour.

---

## 4. Dependencies  

| Library / Framework | Type | Purpose |
|---------------------|------|---------|
| `javax.servlet.*` | Standard | Servlet API for filter, request, response, session handling. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Safe string checks (`isBlank`). |
| `org.apache.log4j.Logger` | Third‑party | Logging (though currently only a logger instance is declared). |
| `com.salesmanager.core.constants.SecurityConstants` | Internal | Holds security constants (e.g., session attribute key). |
| `com.salesmanager.core.module.model.application.CustomerLogonModule` | Internal | Validates authentication tokens. |
| `com.salesmanager.core.util.SpringUtil` | Internal | Retrieves Spring beans (`customerlogon`). |

No platform‑specific APIs beyond standard Java EE and the custom application framework. The filter assumes a Spring context is already initialized.

---

## 5. Additional Notes  

### Edge Cases & Limitations  
1. **Session Creation** – `req.getSession()` always creates a new session if none exists. The `if (session == null)` branch is therefore unreachable. Use `req.getSession(false)` to test for an existing session.  
2. **Token Exposure** – Passing the authentication token as a request parameter (`customerAuthToken`) exposes it in URLs, which can be logged or cached. Consider using a secure cookie or HTTP header instead.  
3. **URL Matching** – `isEscapeUrlFromFilter` uses `String.contains`, which may produce false positives (e.g., `/foo/logon.actionbar`). A more robust path matcher (e.g., regex or AntPathMatcher) would be safer.  
4. **Redirect Logic** – When a redirect occurs, the filter does not preserve the originally requested URL. The application must handle redirect back to the desired page after login.  
5. **Error Response** – For an invalid token, the filter simply sets `SC_UNAUTHORIZED` but does not send a body or redirect to a login page. Depending on client expectations, this may lead to a confusing user experience.  
6. **Thread Safety** – All instance fields are either immutable or only set during initialization, so the filter is thread‑safe.  

### Potential Enhancements  
| Category | Suggested Improvement |
|----------|-----------------------|
| **Security** | Move token handling to a secure cookie or `Authorization` header; add token expiration checks. |
| **Logging** | Log authentication failures and redirects for audit purposes. |
| **Configuration** | Externalize the escape URL list to a properties file or Spring bean for easier maintenance. |
| **Error Handling** | Return a JSON error payload for REST clients; provide a friendly HTML page for browsers. |
| **Testing** | Add unit tests for each branch of `doFilter`; mock `CustomerLogonModule` and session. |
| **Documentation** | Add Javadoc comments explaining each method’s contract and expected usage. |
| **Filter Chain** | Consider chaining to another filter for CSRF protection or rate limiting after authentication. |

Overall, the filter implements a simple, functional authentication guard for customer URLs. With the above refinements, it would become more secure, maintainable, and user‑friendly.

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
import java.util.Arrays;
import java.util.List;

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

import com.salesmanager.core.constants.SecurityConstants;
import com.salesmanager.core.module.model.application.CustomerLogonModule;

public class JAASCustomerSecurityFilter extends AuthFilter {

	private static final String CUSTOMER_AUTH_TOKEN = "customerAuthToken";

	private CustomerLogonModule logonModule = null;

	private Logger log = Logger.getLogger(JAASCustomerSecurityFilter.class);

	private static final List<String> escapeActionList = Arrays
			.asList(new String[] { "/logon.action", "/signin.action",
					"/authenticate.action", "/logout.action",
					"/sendCustomerInformation.action" });

	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
		super.init(filterConfig);
		logonModule = (CustomerLogonModule) com.salesmanager.core.util.SpringUtil
				.getBean("customerlogon");
	}

	@Override
	String getLogonPage(HttpServletRequest request) {
		return request.getContextPath() + "/signin.action";
	}

	@Override
	String getUser(HttpServletRequest request) {
		return (request.getSession() != null ? ((String) request.getSession()
				.getAttribute(SecurityConstants.SM_CUSTOMER_USER)) : null);

	}

	@Override
	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		HttpServletRequest req = (HttpServletRequest) request;
		HttpServletResponse resp = (HttpServletResponse) response;
		resp.setHeader("Cache-Control", "no-cache");
		resp.setHeader("Pragma", "no-cache");
		resp.setDateHeader("Expires", 0);
		String url = req.getRequestURI();
		if (isEscapeUrlFromFilter(url) || url.endsWith(".css")
				|| url.endsWith(".js")) {
			chain.doFilter(request, response);
			return;
		}

		String authToken = request.getParameter(CUSTOMER_AUTH_TOKEN);
		if (StringUtils.isBlank(authToken)) {

			HttpSession session = req.getSession();
			if (session == null) {
				resp.sendRedirect(getLogonPage(req));
			} else {
				if (session.getAttribute(SecurityConstants.SM_CUSTOMER_USER) != null) {
					chain.doFilter(request, response);
				} else {
					resp.sendRedirect(getLogonPage(req));
				}
			}
		} else {
			if (logonModule.isValidAuthToken(authToken)) {
				chain.doFilter(request, response);
			} else {
				((HttpServletResponse) response)
						.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
				return;
			}
		}

	}

	private boolean isEscapeUrlFromFilter(String url) {
		for (String escapeUrl : escapeActionList) {
			if (url.contains(escapeUrl)) {
				return true;
			}
		}
		return false;
	}
	
	public boolean bypassUrl(HttpServletRequest request, HttpServletResponse response, FilterChain chain) {
		return false;
	}
}



```
