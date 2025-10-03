# UrlUtil.java

## Review

## 1. Summary  
**Purpose** – `UrlUtil` provides two helper methods for extracting the unsecured or secured domain for a `MerchantStore` associated with a current HTTP request.  

**Key Components**  
- `getUnsecuredDomain(HttpServletRequest)` – returns the non‑HTTPS domain string.  
- `getSecuredDomain(HttpServletRequest)` – returns the HTTPS domain string.  
- Relies on `SessionUtil` to fetch the `MerchantStore` from the session.  
- Falls back to the request attribute `"STORE"` if the session lookup fails.  
- Delegates the actual domain construction to `ReferenceUtil.getUnSecureDomain()` and `ReferenceUtil.getSecureDomain()`.

**Design Patterns / Libraries**  
- Uses a **utility class** pattern (`static` methods only).  
- Depends on the standard `javax.servlet.http.HttpServletRequest`.  
- No frameworks or third‑party libraries are used directly.

---

## 2. Detailed Description  

### Flow of Execution  
1. **Method call** – A controller or servlet invokes either `getUnsecuredDomain` or `getSecuredDomain`.  
2. **MerchantStore resolution** –  
   - `SessionUtil.getMerchantStore(request)` attempts to read the `MerchantStore` from the HTTP session.  
   - If the result is `null`, the code checks for a request attribute named `"STORE"`.  
3. **Domain lookup** – The resolved `MerchantStore` is passed to `ReferenceUtil.getUnSecureDomain()` or `ReferenceUtil.getSecureDomain()` which produce the string to be returned.  

### Assumptions & Constraints  
- It assumes that `MerchantStore` will always be present either in the session or request attribute.  
- It assumes that `ReferenceUtil` provides the domain logic and that the methods accept a `MerchantStore` instance.  
- No null‑check or error handling is performed on the returned domain; an empty or null string could propagate to callers.  

### Architecture & Design Choices  
- A **simple static utility** is chosen to avoid instance state; this is appropriate for stateless request‑based logic.  
- The fallback to a request attribute provides resilience when the session is not populated (e.g., during certain redirects or API calls).  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `public static String getUnsecuredDomain(HttpServletRequest request)` | Retrieve the non‑HTTPS domain for the current store. | `HttpServletRequest request` – the current HTTP request. | `String` – domain, may be `null`. | None. |
| `public static String getSecuredDomain(HttpServletRequest request)` | Retrieve the HTTPS domain for the current store. | `HttpServletRequest request` – the current HTTP request. | `String` – domain, may be `null`. | None. |

**Reusable Utility** – The two methods are nearly identical; the only difference is the call to `ReferenceUtil`. A single private helper could reduce duplication.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard Java EE | Needed for request handling. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Project‑specific | Domain entity representing a merchant. |
| `com.salesmanager.core.util.www.SessionUtil` | Project‑specific | Session helper. |
| `com.salesmanager.core.util.ReferenceUtil` | Project‑specific | Provides domain strings. |

All dependencies are internal to the `com.salesmanager` code base; no external libraries are imported.

---

## 5. Additional Notes  

### Edge Cases & Potential Issues  
1. **Null Store** – If neither the session nor the request contains a `MerchantStore`, the methods will pass `null` to `ReferenceUtil`. If `ReferenceUtil` doesn’t handle `null`, a `NullPointerException` may occur.  
2. **Missing Request Attribute** – The hard‑coded attribute key `"STORE"` is brittle; if the key changes elsewhere, this util will silently fail.  
3. **Redundant Logic** – Both methods repeat the same store resolution logic; extracting that into a private helper would improve maintainability.  
4. **Thread‑Safety** – Not an issue; no mutable state.  

### Suggested Enhancements  
- **Centralized Store Retrieval**  
  ```java
  private static MerchantStore resolveStore(HttpServletRequest request) {
      MerchantStore store = SessionUtil.getMerchantStore(request);
      return (store != null) ? store : (MerchantStore)request.getAttribute("STORE");
  }
  ```
  Use this in both public methods.

- **Null‑Guarding** – Return an empty string or throw a descriptive exception if the store is `null`.  
- **Configurable Attribute Key** – Expose the request attribute key as a constant or config value to avoid hard‑coding.  

- **Logging** – Log a warning when the store cannot be resolved to aid debugging.  

- **Unit Tests** – Add tests that cover scenarios: store in session, store in request, store missing, and verify the correct domain string is returned.  

Implementing these changes would make the utility more robust, easier to maintain, and clearer for future developers.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 25, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.util;

import javax.servlet.http.HttpServletRequest;

import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.util.www.SessionUtil;

public class UrlUtil {

	public static String getUnsecuredDomain(HttpServletRequest request) {

		MerchantStore store = SessionUtil.getMerchantStore(request);
		
		if(store==null) {
			
			store = (MerchantStore)request.getAttribute("STORE");
			
		}

		return ReferenceUtil.getUnSecureDomain(store);

	}

	public static String getSecuredDomain(HttpServletRequest request) {

		MerchantStore store = SessionUtil.getMerchantStore(request);
		
		if(store==null) {
			
			store = (MerchantStore)request.getAttribute("STORE");
			
		}

		return ReferenceUtil.getSecureDomain(store);

	}

}



```
