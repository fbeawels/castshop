# CustomLogonImpl.java

## Review

## 1. Summary

`CustomLogonImpl` is a concrete implementation of a log‑on module for the **SalesManager** application.  
It handles user authentication, role checking, and session management by interacting with the underlying **MerchantService**.  

Key responsibilities:

| Responsibility | Where it lives | Notes |
|----------------|----------------|-------|
| **User role validation** | `isUserInRole(HttpServletRequest, String)` | Retrieves the current user’s roles (from the session or database) and checks for a specific role code. |
| **Credentials validation** | `validateUserNameAndPassword(HttpServletRequest, String, String)` | Authenticates a user against the merchant information table and creates a `UserPrincipal` in the HTTP session. |
| **User lookup** | `getUser(HttpServletRequest)` | Reads the stored `Principal` from the session and returns the user name. |

The class relies on a simple session‑based authentication mechanism and uses **Hibernate** for data access through `MerchantService`. No advanced frameworks (e.g., Spring Security) are used.

---

## 2. Detailed Description

### Flow of Execution

1. **Login Attempt**  
   - A client submits `userName` and `password`.  
   - `validateUserNameAndPassword` checks for non‑empty credentials, queries the database, and upon success:
     - Creates a `UserPrincipal` and stores it in the HTTP session under the key `"PRINCIPAL"`.
     - Leaves role information un‑loaded; the next role check will load it lazily.

2. **Role Checking**  
   - When the application needs to confirm whether the logged‑in user possesses a specific role, `isUserInRole` is called:
     - Retrieves the `UserPrincipal` from the session (throws if absent).  
     - Tries to fetch a collection of `MerchantUserRole` objects from the session under `"roles"`.  
     - If not present, it invokes `MerchantService.getUserRoles` to load them from the database and then stores them in the session for future calls.  
     - Iterates over the roles to find a match for the supplied `role` code.

3. **User Identification**  
   - `getUser` simply reads the `Principal` from the session and returns the user name or throws if the principal is missing.

### Dependencies & Assumptions

- **Servlet API**: Requires `HttpServletRequest` / `HttpSession`.  
- **Hibernate**: `Session` import is unused – likely a leftover.  
- **Apache Commons Lang**: `StringUtils.isBlank`.  
- **Custom Framework**: `ServiceFactory`, `ServiceException`, `MerchantService`, `MerchantUserInformation`, `MerchantUserRole`, `UserPrincipal` are all application‑specific.  
- **Session Management**: Assumes a single HTTP session per user; no concurrency handling is performed beyond the standard servlet container.  
- **Role Code Comparison**: Uses `equals`, which is case‑sensitive; assumes role codes are stored in a canonical case.  

### Architecture & Design Choices

- **Lazy Role Loading**: The role collection is only fetched on demand, which can reduce database traffic but also means the session may grow large if many users or roles exist.  
- **Session‑Based Principal**: A simple string‑based principal (`UserPrincipal`) is stored in the session instead of leveraging `HttpServletRequest.getUserPrincipal()`.  
- **Service Layer**: All data access is funneled through `MerchantService` obtained via a factory, suggesting a Service Locator pattern rather than dependency injection.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑Effects | Notes |
|--------|---------|------------|--------|--------------|-------|
| `isUserInRole(HttpServletRequest request, String role)` | Checks whether the current user has a given role. | `request`: the current servlet request. `role`: role code to look for. | `boolean` | Reads/updates session attributes (`PRINCIPAL`, `roles`). | Throws `ServiceException` on failure. |
| `validateUserNameAndPassword(HttpServletRequest request, String userName, String password)` | Authenticates credentials and stores a principal in the session. | `request`: servlet request. `userName`, `password`: login credentials. | `void` | May throw `ServiceException`; sets `PRINCIPAL` in session. | Does **not** check if a user is already logged in. |
| `getUser(HttpServletRequest request)` | Retrieves the logged‑in user name from the session. | `request`: servlet request. | `String` | Throws `ServiceException` if no principal present. | Overrides method from `AdministrationLogonModuleImpl`. |

### Utility / Reusable Methods
- No explicit utility methods are present; most logic is self‑contained.

---

## 4. Dependencies

| Library / Component | Nature | Role |
|---------------------|--------|------|
| `javax.servlet.http.*` | Standard API | HTTP request/response/session handling |
| `org.apache.commons.lang.StringUtils` | Third‑party | String null/blank checks |
| `org.hibernate.Session` | Third‑party (unused) | (Intended for DB sessions – not used) |
| `com.salesmanager.core.*` | Application | ServiceFactory, ServiceException, MerchantService, domain entities, constants |
| `com.salesmanager.core.module.impl.application.logon.AdministrationLogonModuleImpl` | Application | Base class providing abstract log‑on contract |

**Platform Specifics**: Designed for a Java EE servlet container (e.g., Tomcat, WildFly). No framework‑specific annotations or DI.

---

## 5. Additional Notes

### Strengths
- **Simplicity**: Straightforward session‑based authentication without heavy frameworks.  
- **Clear Separation**: Business logic (role lookup) is isolated in `MerchantService`.  
- **Error Handling**: Uses custom `ServiceException` with error codes for precise error classification.

### Potential Issues & Edge Cases

1. **Unnecessary Imports & Variables**  
   - `Session session = null;` is never used; remove to avoid confusion.  
   - Import of `org.hibernate.Session` is also unused.

2. **String Constants**  
   - Session attribute keys (`"PRINCIPAL"`, `"roles"`) are hard‑coded.  
   - Suggest extracting them into `static final` constants or using a dedicated `SessionAttribute` enum.

3. **Role Comparison**  
   - Case‑sensitivity may lead to false negatives if role codes vary in case.  
   - Consider using `equalsIgnoreCase` or normalizing role codes on retrieval.

4. **Concurrent Session Updates**  
   - `isUserInRole` writes to the session (`setAttribute`) if roles are null.  
   - In high‑concurrency scenarios, two parallel requests for the same user could trigger duplicate database calls.  
   - A simple `synchronized` block or session attribute locking can mitigate this.

5. **Missing Logout / Session Invalidation**  
   - No explicit logout method; session cleanup must be handled elsewhere.  
   - Recommend adding a `logout` method that removes `PRINCIPAL` and `roles`.

6. **Re‑authentication**  
   - `validateUserNameAndPassword` does not check whether the user is already authenticated.  
   - Could lead to session attribute overrides without notifying the caller.

7. **Exception Handling**  
   - `isUserInRole` catches generic `Exception` and wraps it in `ServiceException`.  
   - Narrower catches (e.g., `SQLException`) would provide clearer diagnostics.

8. **Testing**  
   - Methods depend heavily on the servlet container’s session; unit tests need to mock `HttpServletRequest`/`HttpSession`.  
   - Consider abstracting session access into a helper class to simplify testing.

### Future Enhancements

| Enhancement | Rationale | Suggested Approach |
|-------------|-----------|--------------------|
| **Dependency Injection** | Avoid ServiceLocator (`ServiceFactory`) and facilitate testing. | Switch to CDI/Spring and inject `MerchantService`. |
| **Role Caching** | Reduce DB load for static role sets. | Store role collection in a `WeakHashMap` keyed by user ID; invalidate on logout. |
| **Centralized Session Management** | Avoid magic strings and improve thread safety. | Create a `SessionContext` helper that wraps attribute access. |
| **Audit Trail** | Track login attempts and role checks. | Log successful/failed logins and role lookups with timestamps. |
| **Exception Granularity** | Provide more actionable error messages. | Define custom exceptions (`AuthenticationFailedException`, `RoleNotFoundException`, etc.). |
| **Logout Endpoint** | Properly terminate sessions. | Add `logout(HttpServletRequest)` that invalidates the session. |

---

**Conclusion**  
`CustomLogonImpl` implements a minimal yet functional authentication/authorization layer suitable for small to medium‑sized applications. With a few clean‑ups (unused imports, constants, tighter error handling) and optional refactoring toward dependency injection and session abstraction, the code can be made more robust, testable, and maintainable.

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
package com.salesmanager.core.module.impl.application.logon;

import java.security.Principal;
import java.util.Collection;
import java.util.Iterator;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.hibernate.Session;

import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.merchant.MerchantUserRole;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;

public class CustomLogonImpl extends AdministrationLogonModuleImpl {

	public boolean isUserInRole(HttpServletRequest request, String role)
			throws ServiceException {

		Session session = null;

		try {

			UserPrincipal principal = (UserPrincipal) request.getSession()
					.getAttribute("PRINCIPAL");

			if (principal == null) {
				throw new ServiceException("User Principal does not exist");
			}
			
			
			Collection roles = (Collection)request.getSession().getAttribute("roles");
			

			if(roles==null) {
			
				MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
				roles = mservice.getUserRoles(principal.getName());
			
			}
			
			//if(roles!=null && roles.contains(role)) {
			//	return true;
			//}

			if (roles != null && roles.size() > 0) {
				Iterator i = roles.iterator();
				while (i.hasNext()) {
					MerchantUserRole r = (MerchantUserRole) i.next();
					if (r.getRoleCode().equals(role)) {
						return true;
					}
				}
			}

			return false;

		} catch (Exception e) {

			throw new ServiceException(e);
		}

	}

	void validateUserNameAndPassword(HttpServletRequest request, String userName, String password)
			throws ServiceException {

		// Get username & password from merchant_information
		//String username = request.getParameter("username");
		//String password = request.getParameter("password");

		if (StringUtils.isBlank(userName) || StringUtils.isBlank(password)) {
			throw new ServiceException("Invalid username & password",
					ErrorConstants.INVALID_CREDENTIALS);
		}

		MerchantService service = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		MerchantUserInformation information = null;

		try {
			information = service.getMerchantInformationByUserNameAndPassword(
					userName, password);
		} catch (Exception e) {
			throw new ServiceException("technical problems",
					ErrorConstants.TECHNICAL_DIFFICULTIES);
		}

		if (information == null) {
			throw new ServiceException("Invalid username & password",
					ErrorConstants.INVALID_CREDENTIALS);
		}



		// Create a principal
		UserPrincipal principal = new UserPrincipal(userName);
		request.getSession().setAttribute("PRINCIPAL", principal);

		return;
	}

	@Override
	public String getUser(HttpServletRequest request) throws ServiceException {
		// TODO Auto-generated method stub

		HttpSession session = request.getSession();
		Principal p = (Principal) session.getAttribute("PRINCIPAL");

		if (p != null) {
			return p.getName();
		} else {
			throw new ServiceException("User does not exist");
		}

	}

}



```
