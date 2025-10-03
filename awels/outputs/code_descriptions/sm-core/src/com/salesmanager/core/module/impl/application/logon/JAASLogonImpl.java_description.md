# JAASLogonImpl.java

## Review

## 1. Summary  
**Purpose & Scope**  
`JAASLogonImpl` is a concrete implementation of an administration‑login module that relies on **JAAS** (Java Authentication and Authorization Service) for authentication and role checks. It sits inside the `com.salesmanager.core.module.impl.application.logon` package and extends `AdministrationLogonModuleImpl`.  

**Key Components**  
| Component | Role |
|-----------|------|
| `isUserInRole` | Delegates to the servlet container’s `HttpServletRequest.isUserInRole()` to verify a user’s role. |
| `validateUserNameAndPassword` | Stub method – intended to be overridden by subclasses or to contain JAAS‑specific validation logic. |
| `getUser` | Retrieves the authenticated username from the request (`HttpServletRequest.getRemoteUser()`); throws a `ServiceException` if the username is missing. |

**Design Patterns / Libraries**  
* The class follows a **Template Method**‑style pattern where the base class defines the overall workflow and subclasses provide concrete behavior.  
* It relies on the **Servlet API** (`HttpServletRequest`) and a custom `ServiceException`. No heavyweight frameworks are involved.

---

## 2. Detailed Description  
### Flow of Execution  
1. **Initialization** – The module is instantiated by the container (or a factory) during application startup.  
2. **Runtime** –  
   * When a request arrives, `isUserInRole(request, role)` is called to check permissions.  
   * `getUser(request)` extracts the authenticated username.  
   * `validateUserNameAndPassword(request, userName, password)` is supposed to confirm credentials; currently it does nothing.  
3. **Cleanup** – There is no explicit cleanup logic; the module relies on the container’s lifecycle.

### Assumptions & Constraints  
* The servlet container must be configured for **JAAS** authentication; otherwise `request.getRemoteUser()` will return `null`.  
* The base class `AdministrationLogonModuleImpl` is expected to define other required methods; otherwise the class might be abstract at runtime.  
* `validateUserNameAndPassword` is currently a no‑op; any call will silently succeed, which could be a security risk.

### Architecture Choices  
* **Delegation to the Servlet API** keeps the class lightweight and lets the container handle authentication.  
* Extending a common base (`AdministrationLogonModuleImpl`) promotes code reuse across different login strategies.

---

## 3. Functions/Methods  

| Method | Visibility | Parameters | Returns | Purpose & Notes |
|--------|------------|------------|---------|-----------------|
| `public boolean isUserInRole(HttpServletRequest request, String role)` | Public | `HttpServletRequest`, `String` | `boolean` | Delegates to `request.isUserInRole(role)`; marked `@Override` would clarify intent. |
| `void validateUserNameAndPassword(HttpServletRequest request, String userName, String password)` | Package‑private | `HttpServletRequest`, `String`, `String` | `void` | Stub – should either throw `UnsupportedOperationException` or contain JAAS validation logic. |
| `public String getUser(HttpServletRequest request)` | Public | `HttpServletRequest` | `String` | Retrieves `request.getRemoteUser()`. Throws `ServiceException` if `null`. |
| *Inherited* | — | — | — | The class inherits abstract methods from `AdministrationLogonModuleImpl` (not shown). |

#### Side‑Effects & Exceptions  
* `isUserInRole` may throw a `ServiceException` if the underlying container throws one (unlikely).  
* `getUser` throws `ServiceException` on missing username.  
* `validateUserNameAndPassword` currently has no side effects; any future implementation should handle credential validation securely.

---

## 4. Dependencies  

| Dependency | Type | Remarks |
|------------|------|---------|
| `javax.servlet.http.HttpServletRequest` | **Standard** (Servlet API) | Requires a servlet container. |
| `com.salesmanager.core.service.ServiceException` | **Third‑party** (application specific) | Custom exception used across the project. |
| `AdministrationLogonModuleImpl` | **Internal** | Base class; not part of this snippet but mandatory. |
| Optional: JOSSO configuration | **External** | The comment refers to JOSSO, suggesting that the application may run behind a JOSSO single‑sign‑on gateway. |

No external frameworks (Spring, CDI, etc.) are directly referenced.

---

## 5. Additional Notes  

### Edge Cases / Potential Issues  
1. **Empty `validateUserNameAndPassword`** – This method silently accepts any credentials. If the base class calls it during login, the module effectively bypasses authentication.  
2. **`getRemoteUser()` Nullability** – The method throws a `ServiceException` when the username is `null`. In production, it would be safer to log a warning and perhaps redirect to a login page instead of throwing.  
3. **Thread‑Safety** – All methods are stateless; no concurrency concerns.  
4. **Missing `@Override` Annotations** – Adding `@Override` to methods that override the base class would improve readability and compiler checks.  
5. **Logging** – No logging is performed; adding logs (e.g., on failed role checks) would aid troubleshooting.  

### Suggested Enhancements  
* **Implement `validateUserNameAndPassword`** – Hook into JAAS (`LoginContext`, `Subject`) to verify credentials.  
* **Add JavaDoc** – Document the contract of each method, especially the assumptions about container configuration.  
* **Refactor Access Modifiers** – Make `validateUserNameAndPassword` `protected` or `abstract` if it’s intended to be overridden.  
* **Unit Tests** – Mock `HttpServletRequest` to test each method’s behavior.  
* **Configuration Centralization** – The comment suggests moving logic into `sm-core`; consider extracting configuration details (e.g., JOSSO checks) into a dedicated properties file or service.  
* **Error Handling** – Replace generic `ServiceException` messages with more descriptive, user‑friendly responses or error codes.  

---

**Overall Assessment**  
`JAASLogonImpl` is a lightweight bridge between the servlet container’s JAAS support and the application’s authentication workflow. While the core delegation logic is sound, the incomplete `validateUserNameAndPassword` method and lack of documentation are the primary areas needing attention before this module can be safely deployed in a production environment.

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

import javax.servlet.http.HttpServletRequest;

import com.salesmanager.core.service.ServiceException;

public class JAASLogonImpl extends AdministrationLogonModuleImpl {

	public boolean isUserInRole(HttpServletRequest request, String role)
			throws ServiceException {
		return request.isUserInRole(role);
	}

	void validateUserNameAndPassword(HttpServletRequest request, String userName, String password)
			throws ServiceException {
		// implemented by JAAS
		return;
	}

	public String getUser(HttpServletRequest request) throws ServiceException {
		// TODO Auto-generated method stub

		// @todo put in sm-core as this is a mechanism choice over many
		// possibilities
		String username = request.getRemoteUser();

		if (username == null) {
			throw new ServiceException(
					"username is null in HttpServletRequest, check JOSSO configuration");
		}



		return username;

	}

}



```
