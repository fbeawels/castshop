# RoleInterceptor.java

## Review

## 1. Summary
The file defines an **abstract Struts2 interceptor** (`RoleInterceptor`) that enforces role‑based authorization.  
- **Purpose**: Intercepts every action call, checks whether the current user (stored in the HTTP session as `PRINCIPAL`) is allowed to execute the action, and short‑circuits execution if the user lacks the required role.  
- **Key components**:
  - `intercept(ActionInvocation)` – the core Struts2 method that performs the role check and either continues execution or returns an error result.
  - `isUserInRole(Principal, HttpServletRequest, HttpServletResponse)` – an abstract method that subclasses must implement to define the actual role logic.  
- **Design patterns / frameworks**:
  - **Interceptor pattern** (via Struts2 `Interceptor` interface).
  - **Template method** – `intercept` provides the common workflow, delegating role logic to `isUserInRole`.
  - Uses **Struts2** (`ServletActionContext`, `ActionInvocation`) and **Apache Log4j** for logging.

---

## 2. Detailed Description
1. **Initialization**  
   - `destroy()` and `init()` are no‑ops; the interceptor is stateless, so no expensive resources need initialization or cleanup.

2. **Execution flow** (`intercept` method)  
   - Retrieve the current `HttpServletRequest`/`HttpServletResponse` from `ServletActionContext`.  
   - Obtain the HTTP session (`req.getSession()`).  
   - Extract the `Principal` object stored under the key `"PRINCIPAL"`.  
   - Call `isUserInRole(...)`.  
     - **If `false`**: use `MessageUtil`/`LabelUtil` to attach an “authorization” error to the request and return the result code `"AUTHORIZATIONEXCEPTION"`.  
     - **If `true`**: delegate to `invoke.invoke()` to continue normal action execution.  
   - Any `Exception` thrown during the above steps is caught, logged, and results in a generic error message and the result `"GENERICERROR"`.

3. **Assumptions & Constraints**  
   - The principal must be stored in the session under the exact attribute name `"PRINCIPAL"`.  
   - The `isUserInRole` method must not throw; it should return `false` for unauthorized users.  
   - The code presumes that the interceptor is configured in `struts.xml` (or annotations) and that the result names `"AUTHORIZATIONEXCEPTION"` and `"GENERICERROR"` are defined.

4. **Architecture & Design Choices**  
   - **Stateless interceptor**: no instance fields that change per request, simplifying thread safety.  
   - **Abstract role check**: allows various concrete interceptors (e.g., AdminInterceptor, UserInterceptor) to reuse the common flow while supplying specific role logic.  
   - **Error handling**: logs exceptions and presents user‑friendly error messages.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `destroy()` | Lifecycle hook (no action). | – | – | – |
| `init()` | Lifecycle hook (no action). | – | – | – |
| `intercept(ActionInvocation invoke)` | Main interceptor logic. Checks role, handles errors, and forwards the action invocation. | `ActionInvocation invoke` – the current action context. | `String` – result code (`"AUTHORIZATIONEXCEPTION"`, `"GENERICERROR"`, or result from `invoke.invoke()`). | Adds error messages to request, logs exceptions. |
| `isUserInRole(Principal principal, HttpServletRequest req, HttpServletResponse resp)` | **Abstract** – subclasses implement specific role validation. | `Principal principal`, `HttpServletRequest req`, `HttpServletResponse resp` | `boolean` – `true` if the user has the required role. | None. |

*Utility methods used*: `MessageUtil.addErrorMessage`, `LabelUtil.getInstance().getText` – these add localized error messages to the request.

---

## 4. Dependencies

| Library / API | Type | Notes |
|---------------|------|-------|
| `java.security.Principal` | Standard Java | Represents the authenticated user. |
| `javax.servlet.http.*` | Servlet API | For request/response/session handling. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework. |
| `org.apache.struts2.ServletActionContext` | Struts2 | Provides access to the current request/response. |
| `com.opensymphony.xwork2.*` | Struts2 / XWork2 | Core interceptor and action invocation interfaces. |
| `com.salesmanager.central.AuthorizationException` | Custom | Imported but unused. |
| `com.salesmanager.core.util.LabelUtil` | Custom | For i18n text retrieval. |
| `com.salesmanager.core.util.MessageUtil` | Custom | For adding messages to the request. |

No platform‑specific or heavyweight dependencies; everything is standard for a Struts2 MVC application.

---

## 5. Additional Notes

### Strengths
- Clear separation of concerns: common flow is in the abstract class, role logic in subclasses.
- Thread‑safe: no mutable state.
- Uses i18n utilities for user messages.

### Weaknesses / Edge Cases
1. **Null Handling**  
   - `session.getAttribute("PRINCIPAL")` may return `null`. The code passes `null` to `isUserInRole` without checking; subclasses need to guard against it.  
   - `req.getSession()` can throw an exception if the session is invalid; consider `req.getSession(false)` to avoid creating a new session inadvertently.

2. **Hard‑coded Result Names**  
   - `"AUTHORIZATIONEXCEPTION"` and `"GENERICERROR"` are magic strings. Using constants or Struts2 result configurations would improve maintainability.

3. **Unused Imports**  
   - `ValidationException` and `AuthorizationException` are imported but never used; they can be removed.

4. **Logging**  
   - `log.error(e);` prints stack trace but not the message. Consider `log.error("Authorization check failed", e);` for clarity.

5. **Error Message Construction**  
   - The generic error string concatenates a static text with the exception message. This could expose sensitive information if `e.getMessage()` contains details. A more secure approach would be to log the exception and show a generic message to the user.

### Potential Enhancements
- **Add a `boolean allowNullPrincipal()`** default to `false` to control behaviour when the principal is missing.
- **Configure result names via constants** or properties file.
- **Provide a default implementation of `isUserInRole`** that simply returns `false`, so subclasses only override when needed.
- **Add unit tests** for the interceptor logic, mocking `ActionInvocation` and session attributes.
- **Centralize exception handling**: create a dedicated exception handler that intercepts `Exception` and maps to appropriate result codes.

Overall, the interceptor is a concise and reusable component that cleanly delegates role checking to subclasses while handling errors and user messaging in a consistent way. Addressing the above edge cases would make the component more robust and easier to maintain.

## Code Critique



## Code Preview

```java
/*
 * Provided by CSTI Consulting 
 * Following GNU LESSER GENERAL PUBLIC LICENSE
 * You may obtain more details at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-2011 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.central.security;

import java.security.Principal;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.apache.log4j.Logger;
import org.apache.struts2.ServletActionContext;

import com.opensymphony.xwork2.ActionInvocation;
import com.opensymphony.xwork2.interceptor.Interceptor;
import com.opensymphony.xwork2.validator.ValidationException;
import com.salesmanager.central.AuthorizationException;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;

public abstract class RoleInterceptor implements Interceptor {
	
	private Logger log = Logger
	.getLogger(RoleInterceptor.class);


	public void destroy() {
		// TODO Auto-generated method stub

	}

	public void init() {
		// TODO Auto-generated method stub

	}

	public String intercept(ActionInvocation invoke) throws Exception {
		// TODO Auto-generated method stub
		HttpServletRequest req = (HttpServletRequest) ServletActionContext
		.getRequest();
		HttpServletResponse resp = (HttpServletResponse) ServletActionContext
		.getResponse();
	
		try {
			
			HttpSession session = req.getSession();
			Principal p = (Principal) session.getAttribute("PRINCIPAL");
			
			if(!isUserInRole(p, req, resp)) {
				MessageUtil.addErrorMessage(req, LabelUtil
						.getInstance().getText("messages.authorization"));
					return "AUTHORIZATIONEXCEPTION";
			}
			
			return invoke.invoke();
		} catch (Exception e) {
		

			log.error(e);
			MessageUtil.addErrorMessage(req, LabelUtil
					.getInstance().getText("errors.technical") + " " + e.getMessage());
			return "GENERICERROR";
		}
	}
	
	protected abstract boolean isUserInRole(Principal principal, HttpServletRequest req, HttpServletResponse resp);

}



```
