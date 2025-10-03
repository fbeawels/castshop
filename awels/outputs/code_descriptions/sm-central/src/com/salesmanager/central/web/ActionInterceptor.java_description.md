# ActionInterceptor.java

## Review

## 1. Summary
The **`ActionInterceptor`** is a custom Struts 2 interceptor that centralises error handling for all actions executed within the application.  
- **Purpose** – intercept each action invocation, catch any exception thrown by the action, and map specific exceptions (`ValidationException`, `AuthorizationException`) to appropriate Struts 2 result strings while logging unexpected errors and adding user‑friendly messages to the request.  
- **Key components**  
  - **`intercept(ActionInvocation)`** – core method where the action is invoked and exceptions are processed.  
  - **`init()` / `destroy()`** – lifecycle hooks required by the `Interceptor` interface (currently unused).  
  - **`Logger`** – used to record unhandled exceptions.  
- **Design patterns & libraries** – Implements the *Command*‑like pattern inherent to Struts 2 interceptors. Uses **Struts 2** (`org.apache.struts2.ServletActionContext`, `Interceptor`, `ActionInvocation`), **Apache Log4j** for logging, and a project‑specific `MessageUtil` for populating request‑level error messages.

## 2. Detailed Description
### Execution Flow
1. **Pre‑invocation** – `intercept()` retrieves the current `HttpServletRequest` and `HttpServletResponse` from the `ServletActionContext`.  
2. **Action execution** – `invoke.invoke()` runs the wrapped action.  
3. **Exception handling** – If any exception bubbles up:
   - `ValidationException` is silently swallowed, returning `SUCCESS`.  
   - `AuthorizationException` maps to a custom result string `"AUTHORIZATIONEXCEPTION"`.  
   - All other exceptions are logged, a generic error message is attached to the request via `MessageUtil`, and the method returns `ERROR`.  
4. **Post‑invocation** – The result string dictates which Struts 2 result page (JSP, JSON, etc.) will be rendered.

### Dependencies & Assumptions
- Relies on Struts 2’s static `ServletActionContext`; assumes a servlet‑based deployment.  
- Expects that the application has a result named `"AUTHORIZATIONEXCEPTION"` configured in `struts.xml`.  
- Uses a project‑specific `MessageUtil` to inject error messages; this utility is assumed to work with the request’s attribute map.  
- No thread‑local state is stored, so the interceptor is thread‑safe.

### Architecture & Design Choices
- **Centralised error handling**: By placing all exception logic in one interceptor, individual actions can remain lean and focused on business logic.  
- **Result mapping**: The use of string constants (e.g., `"ERROR"`) couples the interceptor to Struts 2’s result system, which is intentional.  
- **Extensibility**: Additional exception types can be handled by extending the `if` chain or by refactoring into a map of exception → result.

## 3. Functions/Methods
| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `String intercept(ActionInvocation invoke)` | Wraps action execution, handles exceptions, returns a Struts result name. | `ActionInvocation invoke` – the current action chain. | `String` – result name (`SUCCESS`, `"AUTHORIZATIONEXCEPTION"`, or `ERROR`). | Logs unhandled exceptions; populates request with error message. |
| `void destroy()` | Lifecycle hook called when interceptor is removed. | None | None | None (currently no logic). |
| `void init()` | Lifecycle hook called when interceptor is created. | None | None | None (currently no logic). |

### Reusable / Utility Methods
- **`MessageUtil.addErrorMessage(HttpServletRequest, String)`** – not defined here, but assumed to add a user‑friendly message to the request. It is reusable across the project wherever error messaging is required.

## 4. Dependencies
| Library | Type | Notes |
|---------|------|-------|
| `javax.servlet.http.HttpServletRequest` / `HttpServletResponse` | Standard API | Servlet container only. |
| `org.apache.log4j.Logger` | Third‑party (Log4j) | Standard logging framework; ensure Log4j 2 is not used inadvertently. |
| `org.apache.struts2.ServletActionContext` | Struts 2 | Static holder for servlet objects; less testable. |
| `com.opensymphony.xwork2.ActionInvocation`, `Interceptor` | Struts 2 | Core interceptor interfaces. |
| `com.opensymphony.xwork2.validator.ValidationException` | Struts 2 | Validation error. |
| `com.salesmanager.central.AuthorizationException` | Project‑specific | Custom auth exception. |
| `com.salesmanager.central.BaseAction` | Project‑specific | Not directly used in this class but indicates the action hierarchy. |
| `com.salesmanager.core.util.MessageUtil` | Project‑specific | Utility for request messaging. |

No platform‑specific or non‑standard dependencies beyond the Struts 2 framework and Log4j.

## 5. Additional Notes & Recommendations
### Strengths
- **Centralised error handling** simplifies individual actions.  
- **Clear mapping** of known exception types to result names.  
- **Thread‑safe** implementation (no mutable shared state).  

### Weaknesses / Edge Cases
1. **Catching `Exception`** – Swallows all checked and unchecked exceptions, potentially masking programming errors (e.g., `NullPointerException`). Consider catching `RuntimeException` separately or re‑throwing fatal ones.  
2. **Hardcoded result names** – `"AUTHORIZATIONEXCEPTION"` is a magic string. If the result name changes in `struts.xml`, the interceptor will break silently.  
3. **Using `ServletActionContext`** – Direct access to static context reduces testability. Prefer `ActionContext.getContext().get(ServletContext)` or dependency injection.  
4. **Empty lifecycle methods** – `init()` and `destroy()` are stubbed; if the interceptor grows, these may need implementation.  
5. **No status code handling** – For APIs, you might want to set HTTP status codes (`403`, `500`, etc.) instead of just a result name.

### Suggested Enhancements
- **Refactor exception handling** into a `Map<Class<? extends Exception>, String>` to avoid lengthy `if` chains.  
- **Add status code support**: use `HttpServletResponse` to set status codes for specific exceptions.  
- **Parameterise result names** via interceptor configuration (`<interceptor class="..."> <param name="authorizationResult">dashboard</param> </interceptor>`).  
- **Implement `init()` and `destroy()`** if any resource allocation is required later.  
- **Use Struts 2 `ActionContext`** for request/response retrieval to improve testability.  
- **Logging best practices** – include stack traces and correlation IDs where possible.

Overall, the interceptor is functional and aligns with typical Struts 2 error‑handling patterns, but a few refactorings could make it more robust, maintainable, and testable.

## Code Critique



## Code Preview

```java
package com.salesmanager.central.web;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.log4j.Logger;
import org.apache.struts2.ServletActionContext;

import com.opensymphony.xwork2.ActionInvocation;
import com.opensymphony.xwork2.interceptor.Interceptor;
import com.opensymphony.xwork2.validator.ValidationException;
import com.salesmanager.central.AuthorizationException;
import com.salesmanager.central.BaseAction;
import com.salesmanager.core.util.MessageUtil;

public class ActionInterceptor implements Interceptor {

	
	private Logger log = Logger
	.getLogger(ActionInterceptor.class);
	

	public String intercept(ActionInvocation invoke) throws Exception {
		HttpServletRequest req = (HttpServletRequest) ServletActionContext
		.getRequest();
		
		try {

			
			HttpServletResponse resp = (HttpServletResponse) ServletActionContext
					.getResponse();
			return invoke.invoke();
		} catch (Exception e) {
			if (e instanceof ValidationException) {// do nothing
				return com.opensymphony.xwork2.Action.SUCCESS;
			}
			if (e instanceof AuthorizationException) {// return to dashborad
				return "AUTHORIZATIONEXCEPTION";
			}
			log.error(e);
			
			MessageUtil.addErrorMessage(req,e.getMessage());
			
			
			return com.opensymphony.xwork2.Action.ERROR;
		}
		
	}


	public void destroy() {
		// TODO Auto-generated method stub
		
	}


	public void init() {
		// TODO Auto-generated method stub
		
	}

}



```
