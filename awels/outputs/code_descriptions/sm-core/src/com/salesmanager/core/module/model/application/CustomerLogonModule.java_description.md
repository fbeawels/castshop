# CustomerLogonModule.java

## Review

## 1. Summary  

The file declares **`CustomerLogonModule`**, an interface that extends a generic `LogonModule` (not shown). It represents the contract for authentication‑related operations performed on `Customer` objects in a web application.  
Key responsibilities exposed by the interface:

| Method | Purpose |
|--------|---------|
| `logon(HttpServletRequest, int)` | Authenticates a customer for a particular merchant and returns the fully‑populated `Customer` entity. |
| `logout(HttpServletRequest)` | Terminates the current customer session. |
| `getUser(HttpServletRequest)` | Retrieves the username (or identifier) of the currently authenticated customer. |
| `getAuthToken(Customer, long)` | Generates a one‑time authentication token tied to a customer and an expiry value. |
| `isValidAuthToken(String)` | Validates a supplied token. |
| `resetPassword(Customer, String, String)` | Allows a customer to change his/her password after verifying the current one. |

The interface is designed for a *pluggable* authentication subsystem; implementations can vary (e.g., database‑backed, LDAP, OAuth, etc.) while the rest of the application depends only on this contract. The use of `HttpServletRequest` ties the module to the servlet API, implying that the implementation will extract session or cookie data directly from the request.

## 2. Detailed Description  

### Core Components  

1. **`CustomerLogonModule` (interface)**  
   - Extends `LogonModule`, inheriting generic logon operations (not provided).  
   - Declares customer‑specific authentication operations.

2. **`Customer` Entity**  
   - Represents a customer record; presumably contains fields like id, email, password hash, etc.  
   - Passed to several methods for identity and token operations.

3. **`HttpServletRequest`**  
   - Used to obtain request‑level data (session attributes, cookies, headers).  
   - The implementation will likely manipulate `HttpSession` or headers to store authentication state.

### Execution Flow (typical usage pattern)

1. **Login**  
   - `logon(request, merchantId)` validates credentials (implementation‑specific), establishes a session (e.g., storing a `Customer` object), and returns the logged‑in customer.

2. **Session Retrieval**  
   - `getUser(request)` reads the session or token to return the username of the authenticated user.

3. **Logout**  
   - `logout(request)` invalidates the session or removes authentication cookies.

4. **Token‑Based Authentication**  
   - `getAuthToken(customer, timeout)` generates a token (perhaps JWT or a custom signed string).  
   - `isValidAuthToken(token)` verifies its integrity and expiry.

5. **Password Management**  
   - `resetPassword(customer, currentPassword, newPassword)` verifies the current password and updates the stored hash.

### Assumptions & Constraints  

| Item | Assumption | Consequence |
|------|------------|-------------|
| **Password handling** | Implementation must perform secure hashing; interface exposes plain passwords. | Requires careful implementation to avoid accidental logging or storage. |
| **Session persistence** | Uses `HttpSession` or cookies; no explicit contract. | Implementers must agree on session strategy to remain compatible. |
| **Token format** | The interface does not dictate the token type. | Allows flexibility but mandates consistent behavior across implementations. |
| **Merchant context** | `logon` receives a merchant id, implying multi‑tenant support. | Implementation must filter users per merchant. |
| **Exception handling** | All methods throw `ServiceException`. | Clients must handle this checked exception. |

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side Effects | Notes |
|--------|-----------|---------|--------|---------|--------------|-------|
| `logon` | `Customer logon(HttpServletRequest request, int merchantId) throws ServiceException` | Authenticates and establishes a session for the customer. | `HttpServletRequest` (contains credentials), `merchantId` | Authenticated `Customer` instance | Likely stores customer in session, sets cookies | Must be thread‑safe (servlet environment). |
| `logout` | `void logout(HttpServletRequest request) throws ServiceException` | Terminates the customer's session. | `HttpServletRequest` (to locate session) | None | Invalidates session, clears cookies | Should be idempotent. |
| `getUser` | `String getUser(HttpServletRequest request) throws ServiceException` | Retrieves the username of the currently logged‑in customer. | `HttpServletRequest` | Username string or null if not authenticated | None | Should not modify session. |
| `getAuthToken` | `String getAuthToken(Customer customer, long timOutMillis)` | Generates an authentication token for stateless authentication. | `Customer` object, timeout in ms | Token string | None | Token must include expiration and signature. |
| `isValidAuthToken` | `boolean isValidAuthToken(String authToken)` | Validates the authenticity and expiry of a token. | Token string | Boolean | None | Implementation may throw `ServiceException` internally. |
| `resetPassword` | `void resetPassword(Customer customer, String currentPassword, String newPassword) throws ServiceException` | Changes the customer's password after verifying the current one. | `Customer`, current password, new password | None | Persists new password hash | Must hash new password securely. |

### Reusable/Utility Methods  

None are defined directly in this interface. However, implementations may expose helper methods such as token generation utilities or password hashing wrappers.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard (Servlet API) | Ties the module to Java EE / Jakarta EE web containers. |
| `com.salesmanager.core.entity.customer.Customer` | Internal | Represents customer domain object. |
| `com.salesmanager.core.service.ServiceException` | Internal | Custom checked exception for service layer errors. |
| `com.salesmanager.core.module.model.application.LogonModule` | Internal | Super‑interface; likely defines common login/logout methods for non‑customer entities. |

No external third‑party libraries are referenced directly; however, implementations may rely on cryptographic libraries (e.g., JCA, BouncyCastle) or persistence frameworks (JPA, Hibernate) to fulfill the contract.

## 5. Additional Notes  

### Design Strengths  

- **Clear separation of concerns**: Authentication logic is isolated in an interface, facilitating multiple back‑ends.  
- **Pluggability**: Swapping authentication providers (DB, LDAP, OAuth) only requires a new implementation of this interface.  
- **Multi‑tenant awareness**: `merchantId` parameter in `logon` anticipates multi‑merchant contexts.

### Potential Weaknesses & Edge Cases  

1. **Password Exposure**  
   - `resetPassword` receives passwords in clear text; the interface does not mandate any protection (e.g., HTTPS only).  
   - Risk of accidental logging or memory leaks if implementations are careless.

2. **Session Management Ambiguity**  
   - No explicit contract on how session data is stored.  
   - Different implementations may store different session attributes, leading to incompatibilities.

3. **Token Security**  
   - The interface does not specify token format (JWT, opaque, signed).  
   - Without clear contract, one implementation might use a simple random string, another a JWT, causing confusion in downstream consumers.

4. **Error Handling**  
   - All methods throw a generic `ServiceException`.  
   - Clients cannot distinguish between authentication failures, token expiry, or system errors without inspecting the exception’s cause.

5. **Thread‑safety**  
   - Implementations must be safe for concurrent servlet requests; the interface does not provide guarantees or guidelines.

### Future Enhancements  

- **Explicit Token Interface**  
  - Define a `AuthToken` type (with fields like `value`, `expiry`, `issuedAt`) to standardise token handling.  
- **Password Policy Enforcement**  
  - Add methods for checking password strength or policy compliance before resetting.  
- **Multi‑Factor Support**  
  - Provide hooks for 2FA tokens or biometric verification.  
- **Logout Notification**  
  - Return a status or callback to notify other services when a user logs out (e.g., invalidate cached tokens).  
- **Session Invalidation Strategy**  
  - Add methods to invalidate all sessions for a user, useful in account‑compromise scenarios.

Overall, `CustomerLogonModule` offers a solid, extensible foundation for customer authentication in a Java web application. Careful implementation of the contract, especially around security concerns, will be critical to maintain robustness and protect user data.

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
package com.salesmanager.core.module.model.application;

import javax.servlet.http.HttpServletRequest;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.service.ServiceException;

public interface CustomerLogonModule extends LogonModule {

	public Customer logon(HttpServletRequest request, int merchantId)
			throws ServiceException;

	public void logout(HttpServletRequest request) throws ServiceException;

	public String getUser(HttpServletRequest request) throws ServiceException;

	public String getAuthToken(Customer customer, long timOutMillis);

	public boolean isValidAuthToken(String authToken);

	public void resetPassword(Customer customer, String currentPassword,
			String newPassword) throws ServiceException;

}



```
