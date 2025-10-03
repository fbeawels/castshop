# SecurityUtil.java

## Review

## 1. Summary  
**Purpose**  
`SecurityUtil` is a small helper that checks whether the current HTTP user (either a *customer* or a *merchant*) is in a particular role.  The checks are delegated to Spring‑managed `LogonModule` beans (`customerLogon` or `merchantLogon`).  

**Key components**  
| Component | Role |
|-----------|------|
| `isCustomerInRole(HttpServletRequest, String)` | Returns `true` if the current customer has the requested role. |
| `isUserInRole(HttpServletRequest, String)` | Returns `true` if the current merchant has the requested role. |
| `SpringUtil.getBean(...)` | Retrieves the corresponding `LogonModule` bean from the Spring context. |

**Design patterns / frameworks**  
* **Spring dependency injection** – beans are obtained via `SpringUtil.getBean`.  
* **Singleton service façade** – all methods are static; the class acts as a utility façade.  
* **Logging** – uses `org.apache.log4j.Logger`.  

The class is deliberately simple; its responsibilities are limited to role checking.

---

## 2. Detailed Description  
During a web request, one of the static methods is invoked with the `HttpServletRequest` and the role name.  
1. The method obtains the appropriate `LogonModule` bean from Spring.  
2. It calls `logon.isUserInRole(request, role)` – the actual authority check is implemented inside the bean.  
3. If anything goes wrong (bean not found, NullPointer, etc.) an exception is caught, logged, and a default value (`false`) is returned.  

No resources are held across method calls, so there is no explicit cleanup logic. The class is stateless; thread‑safety is guaranteed by the JVM for the static fields (`log`).  

### Assumptions & Constraints  
* **Spring context is already initialized** – otherwise `SpringUtil.getBean` will throw an exception.  
* **Role information is stored on the HTTP session** – implied by the commented caching logic.  
* **No multi‑tenant or hierarchical role support** – the bean encapsulates that logic.  

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `isCustomerInRole` | `public static boolean isCustomerInRole(HttpServletRequest request, String role)` | Checks if the current *customer* user has the given role. | `HttpServletRequest` – current request; `String` – role name | `boolean` – `true` if the role is granted | Logs error if an exception occurs; otherwise no state change. |
| `isUserInRole` | `public static boolean isUserInRole(HttpServletRequest request, String role)` | Checks if the current *merchant* user has the given role. | `HttpServletRequest` – current request; `String` – role name | `boolean` – `true` if the role is granted | Logs error if an exception occurs; otherwise no state change. |

**Utility considerations**  
Both methods perform identical logic except for the bean name.  The duplication could be refactored into a private helper:

```java
private static boolean isInRole(HttpServletRequest request, String beanName, String role) { … }
```

The commented block in `isUserInRole` suggests a session‑based caching strategy that could reduce bean calls.  If performance becomes a concern, that logic could be re‑enabled or replaced with a proper cache.

---

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Classic Log4j 1.x logging. |
| `javax.servlet.http.HttpServletRequest` | Standard | Servlet API. |
| `com.salesmanager.core.module.model.application.LogonModule` | Internal | Interface that performs role checks. |
| `com.salesmanager.core.service.ServiceFactory` & `com.salesmanager.core.service.merchant.MerchantService` | Internal | Imported but **unused** – should be removed. |
| `SpringUtil` (in same package or elsewhere) | Internal | Provides `getBean`. |

No platform‑specific constraints beyond the servlet container. The code relies on Spring’s bean lifecycle; if the context is not available the methods silently fail and return `false`.

---

## 5. Additional Notes  

### Strengths  
* Very small and focused – easy to read.  
* Centralised role checks – other code can call `SecurityUtil.isCustomerInRole(...)`.  
* Uses Spring to decouple role logic from the utility class.  

### Weaknesses & Edge Cases  
1. **Exception swallowing** – generic `catch (Exception)` hides all errors.  Clients receive `false` even if the failure is due to mis‑configuration.  
2. **Logging messages** – `log.error("Customer " + e)` is uninformative; better to log the stack trace.  
3. **Redundant imports** – `ServiceFactory`, `MerchantService`, and `Collection` are unused.  
4. **Duplicate code** – both methods are identical except for the bean name.  
5. **Potential NPE** – if `SpringUtil.getBean` returns `null`, a `NullPointerException` will be thrown when invoking `isUserInRole`.  
6. **No caching** – each call pulls the bean and delegates to the module.  While Spring bean lookup is cheap, repeated calls could be expensive in high‑traffic scenarios.  
7. **Hard‑coded bean names** – if the bean names change, the code must be updated.  

### Suggested Improvements  
* **Refactor**: create a private helper that accepts the bean name to eliminate duplication.  
* **Specific exception handling**: catch `BeansException` or `NullPointerException` and rethrow a custom runtime exception or return a more descriptive status.  
* **Enhanced logging**: use `log.error("Failed to check role for user", e);`.  
* **Remove unused imports** and add `SpringUtil` import if not in the same package.  
* **Optional caching**: uncomment and re‑implement the session‑based caching logic, or use a dedicated cache (e.g., Guava Cache).  
* **Dependency injection**: consider injecting the `LogonModule` beans rather than pulling them statically; this would make unit‑testing easier.  

### Future Extensions  
* Add methods for checking multiple roles at once or for checking permissions instead of roles.  
* Support role hierarchy or tenant‑specific role checks.  
* Integrate with Spring Security to replace this custom mechanism if feasible.  

Overall, the class is functional but could benefit from a modest refactor and better error handling to improve maintainability and robustness.

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
package com.salesmanager.core.util;

import java.util.Collection;
import java.util.HashMap;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.apache.log4j.Logger;

import com.salesmanager.core.module.model.application.LogonModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;

public class SecurityUtil {

	private static Logger log = Logger.getLogger(SecurityUtil.class);

	/**
	 * Check if the customer is in the appropriate role for a given action
	 * 
	 * @param request
	 * @param role
	 * @return
	 */
	public static boolean isCustomerInRole(HttpServletRequest request,
			String role) {

		try {

			LogonModule logon = (LogonModule) SpringUtil
					.getBean("customerLogon");
			return logon.isUserInRole(request, role);

		} catch (Exception e) {
			log.error("Customer " + e);
		}

		return false;
	}

	/**
	 * Check if the merchant is in the appropriate role for a given action
	 * 
	 * @param request
	 * @param role
	 * @return
	 */
	public static boolean isUserInRole(HttpServletRequest request,
			String role) {
		
		boolean rl = false;
		
		try {
			
			LogonModule logon = (LogonModule) SpringUtil.getBean("merchantLogon");
			rl = logon.isUserInRole(request, role);
			
		} catch (Exception e) {
			log.error(e);
		}
		
		return rl;
		


/*		try {

			Map roles = (Map) request.getSession().getAttribute("roles");

			if (roles == null || !roles.containsKey(role)) {
				LogonModule logon = (LogonModule) SpringUtil
						.getBean("merchantLogon");
				boolean rl = logon.isUserInRole(request, role);
				if (roles == null) {
					roles = new HashMap();
				}
				roles.put(role, rl);
			}
			boolean aRole = (Boolean) roles.get(role);
			return aRole;

		} catch (Exception e) {
			log.error("Customer " + e);
		}*/
	}

}



```
