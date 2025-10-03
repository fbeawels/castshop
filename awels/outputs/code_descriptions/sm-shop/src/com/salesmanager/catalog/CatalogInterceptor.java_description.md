# CatalogInterceptor.java

## Review

## 1. Summary  
**Purpose** – `CatalogInterceptor` is a Struts‑2 interceptor that runs before every catalog‑related action. It ensures that:

1. The user’s profile URL (stored in the HTTP session) is cleared so that stale data is not reused.
2. The *mini shopping cart* (a lightweight representation of the user’s cart) is synchronised between the HTTP session and a cookie.  
   * If the session does not contain a cart, the interceptor looks for a cookie named  
     `CatalogConstants.SKU_COOKIE + <merchantId>`.  
   * The cookie value (JSON‑encoded) is deserialised into a `ShoppingCart`, totals are recalculated and the cart is stored back in the session.  
   * If deserialization fails the cookie is expired.

**Key Components**

| Class | Role |
|-------|------|
| `CatalogInterceptor` | Extends `ShopInterceptor`; contains the interception logic. |
| `SessionUtil` | Retrieves/sets session attributes (mini‑cart, store). |
| `MiniShoppingCartSerializationUtil` | Handles JSON <-> `ShoppingCart` conversion. |
| `MiniShoppingCartUtil` | Recalculates totals for a cart. |
| `CatalogConstants` | Holds constants such as the cookie name prefix. |

**Notable Design Patterns / Frameworks**

* **Interceptor Pattern** – part of the Struts‑2 framework (`com.opensymphony.xwork2.ActionInvocation`).
* **Utility / Facade Pattern** – `SessionUtil`, `MiniShoppingCartUtil`, etc. expose simple, static methods that hide implementation details.
* **Serialization / Deserialization** – JSON handling of shopping‑cart state.

---

## 2. Detailed Description  

### Execution Flow
1. **Session Clean‑up**  
   The interceptor removes the `"profileUrl"` attribute from the current HTTP session. This is likely a safety measure to avoid carrying over a URL from a previous session.

2. **Mini‑Cart Synchronisation**  
   * Retrieve the current `ShoppingCart` and `MerchantStore` from the session.  
   * If the cart is `null` (i.e. the user has not yet created a cart in the current session), iterate over all request cookies.  
   * For each cookie whose name matches the expected pattern,  
     * Determine the request locale.  
     * Un‑escape the cookie value (to handle any URL‑encoding).  
     * Deserialize the JSON string into a `ShoppingCart`.  
     * If deserialization succeeds, recalculate the cart’s totals and store the cart in the session.  
     * If it fails, mark the cookie for deletion (`maxAge=0`) and add it back to the response.

3. **Return**  
   The method returns `null`. In the parent `ShopInterceptor` (not shown) this likely allows the invocation to continue; otherwise the interceptor would need to call `invoke.invoke()`.

### Assumptions & Constraints
* The `MerchantStore` is always present in the session when this interceptor runs.  
* Cookie values are JSON and unescaped safely.  
* The interceptor is executed before any action that requires a shopping cart.  
* Only one cookie per merchant store is expected; if multiple exist the last one found will be used.

### Architecture & Design Choices
* **Separation of Concerns** – The interceptor focuses only on session/cookie state, delegating cart calculation and JSON conversion to dedicated utilities.
* **Statelessness** – The interceptor does not maintain state; all data is read from the request/response/session.
* **Extensibility** – New cart synchronisation logic can be added by extending or replacing the utilities, without changing the interceptor.

---

## 3. Functions/Methods  

| Method | Parameters | Returns | Description |
|--------|------------|---------|-------------|
| `doIntercept(ActionInvocation invoke, HttpServletRequest req, HttpServletResponse resp)` | *invoke*: the current action invocation.<br>*req*: HTTP request.<br>*resp*: HTTP response. | `String` (always `null` here) | Core interception logic. Removes session attribute, synchronises mini‑cart from cookie to session, expires cookie on failure. |
| `SessionUtil.getMiniShoppingCart(HttpServletRequest)` | *req*: HTTP request. | `ShoppingCart` | Static helper that pulls the cart from the session. |
| `SessionUtil.setMiniShoppingCart(ShoppingCart, HttpServletRequest)` | *cart*: cart to store.<br>*req*: HTTP request. | `void` | Static helper that stores the cart in the session. |
| `SessionUtil.getMerchantStore(HttpServletRequest)` | *req*: HTTP request. | `MerchantStore` | Retrieves the merchant store context from the session. |
| `MiniShoppingCartSerializationUtil.deserializeJSON(String, MerchantStore, Locale)` | *value*: cookie JSON string.<br>*mStore*: merchant store.<br>*locale*: locale. | `ShoppingCart` | Converts JSON into a `ShoppingCart` instance. |
| `MiniShoppingCartUtil.calculateTotal(ShoppingCart, MerchantStore)` | *sc*: cart to update.<br>*mStore*: merchant store. | `void` | Calculates totals (subtotal, tax, etc.) for the cart. |
| `LocaleUtil.getLocale(HttpServletRequest)` | *req*: HTTP request. | `Locale` | Derives the locale from request/session. |
| `StringUtil.unescape(String)` | *value*: string to un‑escape. | `String` | Decodes URL‑encoded or escaped strings. |

*Reusable/Utility methods* are those in `SessionUtil`, `MiniShoppingCartSerializationUtil`, `MiniShoppingCartUtil`, `LocaleUtil`, and `StringUtil`. These encapsulate common operations and can be used elsewhere in the project.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.*` | J2EE Servlet API | Standard library. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Provides string utilities. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework. |
| `com.opensymphony.xwork2.*` | Struts‑2 | Core framework for actions and interceptors. |
| `com.salesmanager.*` | Internal | Project‑specific utilities, constants, entities. |

All dependencies are either part of the standard Java EE stack or internal to the *salesmanager* codebase. No external network services or platform‑specific features are used.

---

## 5. Additional Notes  

### Strengths  
* **Clear separation** of concerns through utility classes.  
* **Robustness**: handles missing carts, cookie absence, and deserialization failures gracefully.  
* **Simplicity**: the interceptor is concise and focused on a single responsibility.

### Potential Issues & Edge Cases  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Null `MerchantStore`** | `getMerchantStore(req)` could return `null`; subsequent code would NPE‑fail when accessing `mStore.getMerchantId()`. | Add a null‑check and either log an error or throw a controlled exception. |
| **Returning `null` from `doIntercept`** | In Struts‑2, interceptors usually call `invoke.invoke()` or return a result string. Returning `null` may inadvertently skip the action execution. | Ensure that the parent `ShopInterceptor` handles a `null` return correctly; otherwise call `invoke.invoke()`. |
| **Cookie Size Limits** | If the mini‑cart grows large, the JSON cookie may exceed typical browser limits (~4 KB). | Consider persisting large carts server‑side (e.g., session or database) and use a smaller token cookie. |
| **Security** | The cookie contains cart data; it is unencrypted and could be tampered with. | Sign or encrypt the cookie value to prevent tampering. |
| **Locale Resolution** | `LocaleUtil.getLocale(req)` may not always match the user's preferences if the request lacks proper headers. | Validate locale against supported values or fallback to store default. |
| **Concurrency** | Two concurrent requests could race when writing the cart to the session. | Use proper synchronization if necessary, though session attributes are typically isolated per request. |

### Future Enhancements  

1. **Cookie Validation** – Add HMAC or JWT signing to guard against tampering.  
2. **Optimised Cart Persistence** – Store a lightweight cart ID in the cookie and keep full cart data server‑side.  
3. **Logging** – Add more granular logging (e.g., cart size, deserialization failures).  
4. **Unit Tests** – Mock `HttpServletRequest`/`HttpServletResponse` and test the cookie sync logic.  
5. **Configurable Cookie Name** – Externalise the cookie prefix to a configuration property.  

---

### Final Verdict  
`CatalogInterceptor` is a well‑structured, focused piece of code that neatly bridges the session and cookie realms for mini‑cart handling. With a few defensive checks (null handling, cookie size, security) and clearer return logic, it would be robust enough for production use in a Struts‑2 e‑commerce application.

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
package com.salesmanager.catalog;

import java.util.Locale;

import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.ActionInvocation;
import com.salesmanager.checkout.util.MiniShoppingCartUtil;
import com.salesmanager.common.ShopInterceptor;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.ShoppingCart;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.MiniShoppingCartSerializationUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.StringUtil;
import com.salesmanager.core.util.www.BaseActionAware;
import com.salesmanager.core.util.www.SessionUtil;

/**
 * Information on the store, request parameters, Locale
 * 
 * @author Carl Samson
 * 
 */
public class CatalogInterceptor extends ShopInterceptor {

	private static Logger log = Logger.getLogger(CatalogInterceptor.class);

	@Override
	protected String doIntercept(ActionInvocation invoke,
			HttpServletRequest req, HttpServletResponse resp) throws Exception {

		/** remove profile url **/
		req.getSession().removeAttribute("profileUrl");
		
				
		
		
		/** synchronize mini shopping cart**/
		
		//get http session shopping cart
		ShoppingCart cart = SessionUtil.getMiniShoppingCart(req);
		MerchantStore mStore = SessionUtil.getMerchantStore(req);
		
		if(cart==null) {//synch only when the cart is null or empty
		
			Cookie[] cookies = req.getCookies();
			if (cookies != null) {
				for (int i = 0; i < cookies.length; i++) {
					Cookie cookie = cookies[i];
					if(cookie.getName().equals(CatalogConstants.SKU_COOKIE + mStore.getMerchantId())) {
						

							Locale locale = LocaleUtil.getLocale(req);
							
							String cookieValue = StringUtil.unescape(cookie.getValue());
							
							ShoppingCart sc = MiniShoppingCartSerializationUtil.deserializeJSON(cookieValue, mStore, locale);
							if(sc!=null) {
							
								MiniShoppingCartUtil.calculateTotal(sc,mStore);
								SessionUtil.setMiniShoppingCart(sc, req);
								
							} else {//expire cookie
								cookie.setValue(null);
								cookie.setMaxAge(0);
								resp.addCookie(cookie);
							}
					}
				}
			}
		
		}
		

		return null;

	}

}



```
