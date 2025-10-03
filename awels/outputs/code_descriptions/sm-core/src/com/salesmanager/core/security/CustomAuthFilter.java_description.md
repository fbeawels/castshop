# CustomAuthFilter.java

## Review

## 1. Summary  

**Purpose** – `CustomAuthFilter` is a servlet filter that plugs into the Sales‑Manager web application’s security pipeline.  
It supplies two pieces of behaviour that the base `AuthFilter` class expects:

| Component | Role |
|-----------|------|
| `getUser(HttpServletRequest)` | Retrieves the authenticated username from the HTTP session. |
| `getLogonPage(HttpServletRequest)` | Provides the URL to the login page that should be displayed when a request is unauthenticated. |
| `bypassUrl(HttpServletRequest, HttpServletResponse, FilterChain)` | Declares which request URIs the filter should ignore (i.e., should pass through unchanged). |

**Design** – The class is a thin extension of a framework filter, overriding only the methods that need custom behaviour. No complex patterns are used; it relies on standard Java EE servlet APIs and a single constant from the application (`SecurityConstants.SM_ADMIN_USER`).

---

## 2. Detailed Description  

### Execution Flow  

1. **Filter Initialization** – The servlet container creates an instance of `CustomAuthFilter` and calls its `init()` method (inherited from `AuthFilter`).  
2. **Request Processing** – For each incoming request:
   * The framework’s `doFilter()` (inherited from `AuthFilter`) is invoked.  
   * `bypassUrl()` is consulted first; if it returns `true`, the request bypasses the rest of the filter chain.  
   * Otherwise, `getUser()` is called to obtain the username from the session.  
   * If a user is present, the request proceeds; if not, `getLogonPage()` supplies the redirect target.
3. **Cleanup** – When the servlet container destroys the filter (`destroy()`), any resources allocated by `AuthFilter` are released.

### Assumptions & Constraints  

* The session attribute `SM_ADMIN_USER` holds the authenticated username as a `String`.  
* A request that contains `/rest/` or `/anonymous/` in its URI should **not** trigger authentication logic.  
* The filter is stateless (no instance fields), so it is thread‑safe by default.  
* No additional dependencies beyond the servlet API and the application’s constants.

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side Effects | Notes |
|--------|-----------|---------|--------|---------|--------------|-------|
| `getUser` | `String getUser(HttpServletRequest request)` (package‑private) | Pulls the current username from the HTTP session. | `request` | Username string (or `null` if absent) | None | Declared package‑private; could be `protected` or `public` if required by the base class. |
| `getLogonPage` | `String getLogonPage(HttpServletRequest request)` | Returns the login page URL relative to the application context. | `request` | `/profile/logon.jsp` prefixed with context path | None | Simple helper; could be static if no state is needed. |
| `bypassUrl` | `boolean bypassUrl(HttpServletRequest request, HttpServletResponse response, FilterChain chain)` | Decides whether the current request should skip authentication. | `request`, `response`, `chain` | `true` if the URI contains `/rest/` or `/anonymous/`; otherwise `false` | None | Not annotated with `@Override`; confirm that the base class defines this method. |

All methods are `public` (except `getUser`) and stateless, making them straightforward to test.

---

## 4. Dependencies  

| Dependency | Type | Comments |
|------------|------|----------|
| `javax.servlet.FilterChain` | Standard J2EE API | Used for filter chaining. |
| `javax.servlet.http.HttpServletRequest` / `HttpServletResponse` | Standard J2EE API | Core servlet objects. |
| `com.salesmanager.core.constants.SecurityConstants` | Application‑specific | Provides the session attribute key (`SM_ADMIN_USER`). |
| `com.salesmanager.core.security.AuthFilter` | Application‑specific | Base filter class that defines the contract (`getUser`, `bypassUrl`, etc.). |

No external third‑party libraries are required. The code is therefore portable across any servlet container (Tomcat, Jetty, etc.) that supports the Servlet 3.x API.

---

## 5. Additional Notes & Recommendations  

### Edge Cases & Potential Issues  

1. **Null Session** – `request.getSession()` will create a session if one does not exist. This might unintentionally create sessions for unauthenticated users. If the intention is only to read an existing session, use `request.getSession(false)` and handle the `null` case.  
2. **Attribute Type Safety** – The code casts the session attribute to `String` without validation. If a different type is stored, a `ClassCastException` will be thrown.  
3. **URI Matching** – The `contains()` checks are very permissive; a request to `/myapp/restfulapi/` would also bypass the filter, which might be fine, but it could also unintentionally allow access to unintended resources. Consider using a more precise pattern (e.g., `startsWith` or a regular expression).  
4. **Thread‑Safety of Constants** – `SecurityConstants.SM_ADMIN_USER` is immutable, so there is no risk.  
5. **Missing `@Override` Annotation** – Adding `@Override` to `getUser` and `bypassUrl` would catch accidental signature mismatches during compilation.

### Possible Enhancements  

| Idea | Benefit |
|------|---------|
| **Lazy session lookup** – use `request.getSession(false)` to avoid creating sessions. |
| **Explicit null handling** – return a sentinel value or throw a custom exception if the session attribute is missing. |
| **Pattern‑based bypass** – replace hard‑coded strings with a list of patterns or a configurable property. |
| **Logging** – add debug logs when bypassing a URL or when a user is not found, to aid troubleshooting. |
| **Unit tests** – because the class is stateless, it can be unit‑tested with mock `HttpServletRequest` objects. |
| **Make `getLogonPage` static** – no instance state, so it can be reused without creating a filter instance. |

### Code Quality Checklist  

- [x] Clear, self‑describing method names.  
- [ ] Missing `@Override` annotations.  
- [ ] No exception handling for session attribute retrieval.  
- [ ] No documentation (JavaDoc) on the public methods.  
- [ ] No logging.  
- [ ] No unit tests shown.

Implementing the above improvements would increase robustness, maintainability, and observability of the filter.

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

import javax.servlet.FilterChain;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import com.salesmanager.core.constants.SecurityConstants;

public class CustomAuthFilter extends AuthFilter {

	@Override
	String getUser(HttpServletRequest request) {
		// TODO Auto-generated method stub

		// check in the httpsession first
		String user = (String) request.getSession().getAttribute(
				SecurityConstants.SM_ADMIN_USER);

		return user;

	}

	public String getLogonPage(HttpServletRequest request) {
		return request.getContextPath() + "/profile/logon.jsp";
	}
	
	public boolean bypassUrl(HttpServletRequest request, HttpServletResponse response, FilterChain chain) {
		
		String url = request.getRequestURI();
		if (url.contains("/rest/")) {
			return true;
		}
		else if (url.contains("/anonymous/")) {
			return true;
		}
		return false;
	}

}



```
