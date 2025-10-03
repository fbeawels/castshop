# CustomerJAASLogonImpl.java

## Review

## 1. Summary

**Purpose & Functionality**  
`CustomerJAASLogonImpl` is an implementation of the `CustomerLogonModule` interface that handles customer authentication, logout, password reset, and token‑based validation for a web application.  
It uses Java Authentication and Authorization Service (JAAS) to perform login/logout and relies on a custom `JAASSecurityCustomerLoginModule` for the actual credential checks.  

**Key Components**

| Component | Responsibility |
|-----------|----------------|
| `logout` | Destroys the JAAS `LoginContext` and removes session attributes. |
| `logon` | Validates credentials, authenticates via JAAS, retrieves the `Customer` entity, and returns it. |
| `isValidLogin` | Creates a `LoginContext` programmatically, logs in, and stores the resulting `Subject` & `UserPrincipal` in the HTTP session. |
| `getAuthToken` / `isValidAuthToken` | Generates a time‑bounded, encrypted authentication token and validates it. |
| `resetPassword` | Delegates password change to `CustomerService`. |

**Design Patterns & Libraries**

* **Singleton / Service Locator** – `ServiceFactory.getService` is used to obtain `CustomerService`.  
* **Factory Method** – `CustomerService` is injected via setter for testability.  
* **JAAS** – The module uses `LoginContext` and a custom callback handler (`CustomerLoginCallBackHandler`).  
* **EncryptionUtil** – Provides symmetric key generation and encryption/decryption (likely AES).  
* **Apache Commons Lang** – For `StringUtils.isBlank`.  
* **Log4j** – For logging.

---

## 2. Detailed Description

### Execution Flow

1. **Login (`logon`)**  
   * Extracts `username` and `password` from the request.  
   * Calls `logout` to clean any existing session.  
   * Calls `isValidLogin` which:
     * Builds a `LoginContext` with the custom callback handler.  
     * Performs `context.login()`.  
     * On success, places a `UserPrincipal` and the `LoginContext` in the session.  
   * If login succeeds, the service encrypts the supplied password, retrieves the customer record via `CustomerService`, and returns it.  
   * On failure, it throws a `ServiceException`.

2. **Logout (`logout`)**  
   * Retrieves the `LoginContext` from the session and invokes `context.logout()`.  
   * Cleans session attributes.

3. **Token Generation / Validation**  
   * `getAuthToken` concatenates an AES key, a separator, and an encrypted payload (email + timeout).  
   * `isValidAuthToken` reverses the process, decrypts the payload, checks the timeout, and confirms that the customer exists.

4. **Password Reset**  
   * Calls `CustomerService.changeCustomerPassword`.  
   * Throws `ServiceException` if the password change fails.

### Dependencies & Assumptions

* Assumes the presence of a JAAS configuration (`jaas.conf`) and the custom login module `JAASSecurityCustomerLoginModule`.  
* Expects `EncryptionUtil.generatekey` and `EncryptionUtil.encrypt/decrypt` to be deterministic and symmetric.  
* Relies on `CustomerService` to be thread‑safe as it may be shared across requests.  
* Uses `HttpSession` for state; the code is not stateless, which may not scale in clustered environments unless session replication is configured.

### Architectural Notes

* **Coupling** – The class couples closely to the `EncryptionUtil` and the JAAS configuration string. A refactor to inject a `TokenService` would reduce tight coupling.  
* **Error Handling** – The code mixes unchecked `RuntimeException`s (e.g., during logout) with checked `ServiceException`s, which can be confusing.  
* **Logging** – Minimal logging; many catch blocks only call `e.printStackTrace()` instead of proper log statements.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects | Comments |
|--------|---------|------------|---------|--------------|----------|
| `logout(HttpServletRequest)` | Clears JAAS context from session. | `HttpServletRequest` | `void` | Removes `PRINCIPAL` and `LOGINCONTEXT` from session. | Throws `RuntimeException` if context creation fails. |
| `logon(HttpServletRequest, int)` | Authenticates user and fetches `Customer`. | `HttpServletRequest`, `merchantId` | `Customer` | Throws `ServiceException` on failure. | Calls `isValidLogin`; encrypts password using `SecurityConstants.idConstant`. |
| `isValidLogin(HttpServletRequest, String, String, int)` | Creates JAAS context, logs in, stores principal. | Request, username, password, merchantId | `boolean` | Stores `UserPrincipal` & `LoginContext` in session. | Returns `true` if JAAS login succeeds. |
| `getUser(HttpServletRequest)` | (Not implemented) | Request | `String` | `null` | Placeholder. |
| `isUserInRole(HttpServletRequest, String)` | (Not implemented) | Request, role | `boolean` | `false` | Placeholder. |
| `getAuthToken(Customer, long)` | Generates encrypted token with timeout. | Customer, timeoutMillis | `String` | `null` on error. | Uses `SecurityConstants.idConstant` as key. |
| `resetPassword(Customer, String, String)` | Changes password. | Customer, currentPwd, newPwd | `void` | Throws `ServiceException` on failure. | Delegates to `CustomerService`. |
| `isValidAuthToken(String)` | Validates token’s authenticity & expiry. | authToken | `boolean` | `false` if invalid. | Decrypts using the key prefix. |
| `getCustomerService()` / `setCustomerService(CustomerService)` | Accessor / Mutator. | `CustomerService` | `CustomerService` | None | Allows dependency injection. |

---

## 4. Dependencies

| External Library | Type | Notes |
|------------------|------|-------|
| `javax.servlet.http.HttpServletRequest / HttpSession` | Standard Java EE | Handles HTTP context. |
| `javax.security.auth.*` | Standard Java SE | JAAS API. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Utility for string checks. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework. |
| `com.salesmanager.core.util.EncryptionUtil` | Internal | Symmetric encryption helpers. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Service locator. |
| `com.salesmanager.core.service.customer.CustomerService` | Internal | CRUD for `Customer`. |
| `com.salesmanager.core.constants.*` | Internal | Constants for errors and security. |

No platform‑specific dependencies beyond standard servlet/J2EE containers are present.

---

## 5. Additional Notes & Recommendations

### Strengths
* **Clear separation** of concerns between authentication (JAAS) and business logic (`CustomerService`).  
* **Reusable token mechanism** that can be used for “remember‑me” or password‑reset flows.  
* **Dependency injection** via setter for `CustomerService` supports unit testing.

### Weaknesses & Edge Cases
1. **Hardcoded JAAS module name** (`com.salesmanager.core.module.impl.application.logon.JAASSecurityCustomerLoginModule`). If the package changes, the code breaks.  
2. **Error handling inconsistency** – `RuntimeException` used in logout, but `ServiceException` in other methods.  
3. **Logging** – Exceptions are printed to standard error rather than logged.  
4. **Session fixation** – The same `LoginContext` may be reused across requests if the user logs in again without logout.  
5. **Token security** – The key used for encryption is derived from a constant; if that constant is compromised, all tokens can be decrypted.  
6. **Thread‑safety** – `customerService` is a mutable field; if this instance is shared, concurrent access could lead to stale references.  
7. **No validation of `merchantId` in `logon`** – The merchant ID is passed to `isValidLogin` but not used elsewhere.  
8. **Missing implementations** (`getUser`, `isUserInRole`) – Potential API contract violation.  

### Suggested Enhancements
1. **Inject JAAS Configuration** – Pass `jaasConfig` or a `Configuration` instance via constructor or setter.  
2. **Use a dedicated TokenService** – Encapsulate token generation/validation logic, parameterize key and separator.  
3. **Improve Logging** – Replace `e.printStackTrace()` with `log.error(e)`.  
4. **Add Session Regeneration** on successful login to prevent fixation.  
5. **Unit Tests** – With Mockito or JUnit to verify interaction with `CustomerService`.  
6. **Handle Null Cases Gracefully** – Return proper error codes rather than generic `ServiceException`.  
7. **Implement `getUser` & `isUserInRole`** or throw `UnsupportedOperationException`.  
8. **Validate `merchantId`** – Ensure that the user belongs to the specified merchant before authentication.  

---

### Final Verdict
The implementation provides a solid foundation for JAAS‑based authentication in a servlet environment. However, the code would benefit from refactoring to separate concerns, improve exception handling, strengthen security (especially around key management), and enhance testability. Addressing the identified weaknesses will increase robustness, maintainability, and scalability.

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

import java.util.Set;

import javax.security.auth.Subject;
import javax.security.auth.login.LoginContext;
import javax.security.auth.login.LoginException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.constants.SecurityConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.module.model.application.CustomerLogonModule;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.EncryptionUtil;

public class CustomerJAASLogonImpl implements CustomerLogonModule {

	private static final String LOGIN_CONTEXT_CONFIG_NAME = "Login";
	private static final String AUTH_TOKEN_SEPERATOR = ":";

	private Logger log = Logger.getLogger(CustomerJAASLogonImpl.class);

	private CustomerService customerService = null;

	public void logout(HttpServletRequest request) throws ServiceException {
		LoginContext context = null;
		SalesManagerJAASConfiguration jaasc = new SalesManagerJAASConfiguration(
				"com.salesmanager.core.module.impl.application.logon.JAASSecurityCustomerLoginModule");

		try {
			HttpSession session = request.getSession();
			context = (LoginContext) session.getAttribute("LOGINCONTEXT");
			if (context != null) {
				context.logout();
			}

			session.removeAttribute("PRINCIPAL");
			session.removeAttribute("LOGINCONTEXT");

		} catch (Exception e) {
			throw new RuntimeException(
					"Unable to Create Logout Context, configuration file may be missing",
					e);
		}

	}

	public Customer logon(HttpServletRequest request, int merchantId)
			throws ServiceException {

		String username = request.getParameter("username");
		String password = request.getParameter("password");
		if (StringUtils.isBlank(username) || StringUtils.isBlank(password)) {
			throw new ServiceException("Invalid username & password",
					ErrorConstants.INVALID_CREDENTIALS);
		}
		logout(request);
		if (isValidLogin(request, username, password, merchantId)) {
			CustomerService customerService = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			Customer customer = null;
			try {
				// retreive a Customer
				String encPassword = EncryptionUtil.encrypt(EncryptionUtil
						.generatekey(String
								.valueOf(SecurityConstants.idConstant)),
						password);
				customer = customerService.findCustomerbyUserNameAndPassword(
						username, encPassword, merchantId);
			} catch (Exception e) {
				logout(request);
				throw new ServiceException("Exception while getting Customer "
						+ e);
			}
			if (customer == null) {
				logout(request);
				throw new ServiceException("Invalid username & password",
						ErrorConstants.INVALID_CREDENTIALS);
				
			} else {
				return customer;
			}

		} else {
			throw new ServiceException("Invalid username & password",
					ErrorConstants.INVALID_CREDENTIALS);
		}
	}

	private boolean isValidLogin(HttpServletRequest req, String username,
			String password, int merchantId) {
		LoginContext context = null;
		try {

			// 1) using jaas.conf
			// context = new LoginContext(LOGIN_CONTEXT_CONFIG_NAME,new
			// CustomerLoginCallBackHandler(username,password));

			// 2) programaticaly created jaas.conf equivalent
			SalesManagerJAASConfiguration jaasc = new SalesManagerJAASConfiguration(
					"com.salesmanager.core.module.impl.application.logon.JAASSecurityCustomerLoginModule");
			context = new LoginContext(LOGIN_CONTEXT_CONFIG_NAME, null,
					new CustomerLoginCallBackHandler(username, password,
							merchantId), jaasc);

		} catch (Exception e) {
			e.printStackTrace();
			throw new RuntimeException(
					"Unable to Create Login Context, configuration file may be missing",
					e);
			/**
			 * needs a jaas.conf file in the startup script Logon {
			 * com.salesmanager.core.module.impl.application.logon.
			 * JAASSecurityCustomerLoginModule required; }; and this parameter
			 * -Djava.security.auth.login.config=jaas.conf
			 */
		}
		if (context != null) {
			try {
				context.login();

				Subject s = context.getSubject();

				if (s != null) {
					Set principals = s.getPrincipals();
				}

				// Create a principal
				UserPrincipal principal = new UserPrincipal(username);

				HttpSession session = req.getSession();
				session.setAttribute("PRINCIPAL", principal);
				session.setAttribute("LOGINCONTEXT", context);

				return true;
			} catch (LoginException e) {
				e.printStackTrace();
				return false;
			}
		}
		return false;
	}

	public String getUser(HttpServletRequest request) throws ServiceException {
		return null;
	}

	public boolean isUserInRole(HttpServletRequest request, String role)
			throws ServiceException {
		return false;
	}

	public String getAuthToken(Customer customer, long timeOutMillis) {
		String authToken = null;
		try {
			// Generate Key and Auth Token which has a timeout interval and Auth
			// token is encrypted.
			// AUTH TOKEN = GENERATED KEY + ENCRYPED (USER EMAIL + SEPERATOR +
			// TIMEOUTMILLIS)
			String key = EncryptionUtil.generatekey(String
					.valueOf(SecurityConstants.idConstant));
			authToken = key
					+ AUTH_TOKEN_SEPERATOR
					+ EncryptionUtil.encrypt(key, customer.getEmail()
							+ AUTH_TOKEN_SEPERATOR
							+ (System.currentTimeMillis() + timeOutMillis));
		} catch (Exception e) {
			e.printStackTrace();
		}
		return authToken;
	}

	public void resetPassword(Customer customer, String currentPassword,
			String newPassword) throws ServiceException {
		CustomerService customerService = (CustomerService) ServiceFactory
				.getService(ServiceFactory.CustomerService);
		try {

			boolean change = customerService.changeCustomerPassword(customer,
					currentPassword, newPassword);

			if (!change) {
				throw new ServiceException("Password do not match ",
						ErrorConstants.INVALID_CREDENTIALS);
			}

		} catch (Exception e) {
			throw new ServiceException("Exception while getting Customer " + e);
		}
	}

	public boolean isValidAuthToken(String authToken) {
		if (!StringUtils.isBlank(authToken)) {
			if (authToken.indexOf(AUTH_TOKEN_SEPERATOR) != -1) {
				String key = authToken.substring(0, authToken
						.indexOf(AUTH_TOKEN_SEPERATOR));
				String value = authToken.substring(authToken
						.indexOf(AUTH_TOKEN_SEPERATOR) + 1, authToken.length());
				try {
					String decryptedToken = EncryptionUtil.decrypt(key, value)
							.trim();
					if (decryptedToken.indexOf(AUTH_TOKEN_SEPERATOR) != -1) {
						String[] strArr = decryptedToken
								.split(AUTH_TOKEN_SEPERATOR);
						String username = strArr[0];
						long timeout = Long.parseLong(strArr[1]);
						if (customerService.findCustomerByEmail(username) != null) {
							if ((System.currentTimeMillis()) < timeout) {
								return true;
							} else {
								return false;
							}
						}
					}
				} catch (Exception e) {
					e.printStackTrace();
				}
			}
		}
		return false;
	}

	public CustomerService getCustomerService() {
		return customerService;
	}

	public void setCustomerService(CustomerService customerService) {
		this.customerService = customerService;
	}
}



```
