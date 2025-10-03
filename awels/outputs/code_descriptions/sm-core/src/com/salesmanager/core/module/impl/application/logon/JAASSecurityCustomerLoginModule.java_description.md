# JAASSecurityCustomerLoginModule.java

## Review

## 1. Summary

| Item | Description |
|------|-------------|
| **Purpose** | Implements a JAAS `LoginModule` that authenticates a customer against the local database. It verifies the supplied username, password and merchant‑id, and, if successful, attaches a `UserPrincipal` to the authenticated `Subject`. |
| **Key Components** | <ul><li>`JAASSecurityCustomerLoginModule` – the login module itself.</li><li>`CustomerService` – a service façade used to query customer data.</li><li>`EncryptionUtil` – a custom utility for encrypting passwords before comparison.</li><li>`UserPrincipal` – a simple `Principal` implementation that holds the authenticated username.</li></ul> |
| **Frameworks / Libraries** | • Java Authentication and Authorization Service (JAAS) <br>• Apache Log4j for logging (though not fully used)<br>• Custom domain classes (`Customer`, `CustomerService`) and utilities (`EncryptionUtil`). |

---

## 2. Detailed Description

### Flow of Execution

1. **Initialization (`initialize`)**  
   - Receives the `Subject`, a `CallbackHandler`, shared state, and options.  
   - Stores the callback handler and subject.  
   - Instantiates `CustomerService` via `ServiceFactory`.  

2. **Login (`login`)**  
   - Creates three callbacks (`NameCallback`, `PasswordCallback`, `TextInputCallback`) for username, password, and merchant‑id.  
   - Delegates to the supplied `CallbackHandler` to collect the values.  
   - Validates that none of the fields are null.  
   - Calls `isValidUser(name, password, merchantId)` which encrypts the supplied password and queries `CustomerService`.  
   - Sets `verification` flag accordingly.  

3. **Commit (`commit`)**  
   - If `verification` is true, a `UserPrincipal` is created and added to the `Subject`.  

4. **Abort / Logout**  
   - `abort` simply clears the username if authentication had not succeeded.  
   - `logout` clears internal state and returns true.  

### Assumptions & Constraints

| Assumption | Implication |
|------------|-------------|
| `ServiceFactory.getService(ServiceFactory.CustomerService)` returns a thread‑safe singleton. | The module can be reused across requests. |
| `EncryptionUtil.encrypt()` produces a reversible representation of the password. | The module performs *encryption* rather than hashing, meaning passwords can be decrypted (potential security risk). |
| The supplied `CallbackHandler` will provide all three callbacks. | If the handler returns unexpected values, the module will throw `LoginException`. |
| Merchant‑id is always numeric and present. | No validation for non‑numeric merchant ids; leads to `NumberFormatException` if malformed. |

### Architecture & Design Choices

* **JAAS Pattern** – The module follows the classic `LoginModule` contract (`initialize`, `login`, `commit`, `abort`, `logout`).  
* **Separation of Concerns** – Password validation logic is isolated in `isValidUser`.  
* **Stateless Service Layer** – Delegates persistence logic to `CustomerService`, keeping the module thin.  

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `initialize(Subject, CallbackHandler, Map, Map)` | Set up internal state, obtain services. | `subject`, `callbackHandler`, `sharedState`, `options` | None | Stores `subject` and `callbackHandler`; creates `customerService`. |
| `login()` | Collect credentials via callbacks, authenticate. | None | `boolean` – true if credentials are valid | May throw `LoginException`; sets `verification` and `username`. |
| `isValidUser(String, String, String)` | Encrypt password and query DB. | `user`, `password`, `merchantId` | `boolean` – true if customer exists | None |
| `commit()` | Attach `UserPrincipal` to the `Subject`. | None | `boolean` – true if commit succeeds | Adds principal to `subject`. |
| `abort()` | Undo login on failure. | None | `boolean` – true if aborted | Clears `username`. |
| `logout()` | Clean up module state. | None | `boolean` – always true | Clears `verification`, `subject`. |
| `assignPrincipal(Principal)` | Adds a principal to the `Subject` if absent. | `Principal p` | None | Mutates `subject`’s principals set. |

### Reusable / Utility Methods

* `assignPrincipal` is generic and could be reused by other modules that need to attach principals.  
* `isValidUser` encapsulates the encryption & lookup logic; however, it currently returns `false` on any exception without distinguishing error types.

---

## 4. Dependencies

| Dependency | Type | Comments |
|------------|------|----------|
| `javax.security.auth.*` | Standard Java | JAAS API. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework (not heavily used). |
| `com.salesmanager.core.constants.SecurityConstants` | Internal | Provides a key constant. |
| `com.salesmanager.core.entity.customer.Customer` | Internal | Domain model. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Factory for obtaining services. |
| `com.salesmanager.core.service.customer.CustomerService` | Internal | Service for customer queries. |
| `com.salesmanager.core.util.EncryptionUtil` | Internal | Custom encryption utility. |
| `java.io.IOException`, `UnsupportedCallbackException` | Standard | Exception handling. |

No platform‑specific dependencies; the code should run on any Java EE/J2EE container that supports JAAS.

---

## 5. Additional Notes

### Strengths

* Follows JAAS contract cleanly.  
* Keeps database logic in a separate service layer.  
* Uses `Principal` to represent authenticated identity.  
* Minimal external configuration.

### Weaknesses / Risks

1. **Password Handling**  
   * The module *encrypts* passwords using a reversible algorithm.  
   * Proper authentication should use a **one‑way hash** (e.g., PBKDF2, bcrypt).  
   * Storing encrypted passwords exposes the system to potential decryption if the key is compromised.

2. **Error Handling**  
   * Exceptions are caught and printed to `stderr` via `e.printStackTrace()`.  
   * This leaks stack traces and may reveal sensitive information.  
   * A proper logging strategy (e.g., `log.error(...)`) should replace the prints.

3. **Merchant‑ID Validation**  
   * The code assumes the merchant‑id callback always returns a numeric string.  
   * Passing a non‑numeric value will trigger `NumberFormatException`, swallowed by `isValidUser` and simply returning `false`.  
   * Explicit validation and meaningful error messages would improve UX.

4. **Thread Safety**  
   * The instance fields (`username`, `verification`, `customerService`) are shared across potential concurrent calls if the same module instance is reused.  
   * JAAS typically creates a new module instance per authentication request, but clarifying this contract in comments or using local variables would be safer.

5. **Principal Assignment**  
   * The `assignPrincipal` method adds a principal only if it is not already present.  
   * It does not handle roles or other attributes that might be needed for fine‑grained authorization.

6. **Logging**  
   * The logger is defined but never used.  
   * Adding logs for success/failure, user IDs, and errors would aid debugging and auditing.

### Suggested Enhancements

| Enhancement | Rationale |
|-------------|-----------|
| Replace encryption with a secure hash function. | Protects passwords even if the database is compromised. |
| Use a dedicated `LoginException` subclass for each failure type. | Allows callers to react differently to user‑not‑found vs. invalid credentials. |
| Validate merchant‑id and provide clear error messages. | Improves user experience and prevents silent failures. |
| Add role extraction and assignment. | Enables fine‑grained authorization downstream. |
| Replace `printStackTrace()` with `log.error(...)`. | Avoids leaking stack traces to console, supports configurable log levels. |
| Consider making the module stateless or ensuring instance fields are thread‑safe. | Prevents accidental data leakage between authentication attempts. |
| Add unit tests covering each callback path and error scenario. | Guarantees reliability and eases future refactoring. |

Overall, the module is a solid foundation but would benefit from tightening security practices and improving error handling to meet production‑grade standards.

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

import java.io.IOException;
import java.security.Principal;
import java.util.Map;

import javax.security.auth.Subject;
import javax.security.auth.callback.Callback;
import javax.security.auth.callback.CallbackHandler;
import javax.security.auth.callback.NameCallback;
import javax.security.auth.callback.PasswordCallback;
import javax.security.auth.callback.TextInputCallback;
import javax.security.auth.callback.UnsupportedCallbackException;
import javax.security.auth.login.LoginException;
import javax.security.auth.spi.LoginModule;

import org.apache.log4j.Logger;

import com.salesmanager.core.constants.SecurityConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.EncryptionUtil;

public class JAASSecurityCustomerLoginModule implements LoginModule {

	private Logger log = Logger
			.getLogger(JAASSecurityCustomerLoginModule.class);

	private CallbackHandler callbackHandler = null;
	private boolean verification = false;
	private Subject subject = null;
	private String username = null;

	private CustomerService customerService = null;

	public boolean abort() throws LoginException {
		if (!verification) {
			username = null;
			return true;
		}

		return false;
	}

	public boolean commit() throws LoginException {
		if (verification) {
			assignPrincipal(new UserPrincipal(username));
			return true;
		} else {
			username = null;
			return false;
		}
	}

	public void initialize(Subject subject, CallbackHandler callbackHandler,
			Map<String, ?> sharedState, Map<String, ?> options) {
		this.callbackHandler = callbackHandler;
		this.subject = subject;
		customerService = (CustomerService) ServiceFactory
				.getService(ServiceFactory.CustomerService);
	}

	public boolean login() throws LoginException {
		Callback[] callBacks = new Callback[3];
		callBacks[0] = new NameCallback("name");
		callBacks[1] = new PasswordCallback("password", false);
		callBacks[2] = new TextInputCallback("merchantId");
		try {
			callbackHandler.handle(callBacks);
		} catch (IOException e) {
			e.printStackTrace();
			throw new LoginException(e.getMessage());
		} catch (UnsupportedCallbackException e) {
			e.printStackTrace();
			throw new LoginException(e.getMessage());
		}
		if (((NameCallback) callBacks[0]).getName() == null) {
			throw new LoginException("UserName cannot be Null");
		}
		if (((PasswordCallback) callBacks[1]).getPassword() == null) {
			throw new LoginException("Password cannot be Null");
		}

		String name = ((NameCallback) callBacks[0]).getName();
		String password = String.valueOf(((PasswordCallback) callBacks[1])
				.getPassword());
		String merchantId = ((TextInputCallback) callBacks[2]).getText();

		if (isValidUser(name, password, merchantId)) {
			username = ((NameCallback) callBacks[0]).getName();
			verification = true;
		} else {
			verification = false;
		}
		return verification;
	}

	public boolean logout() throws LoginException {
		verification = false;
		subject = null;
		return true;
	}

	private boolean isValidUser(String user, String password, String merchantId) {
		// we should check merchantId
		try {
			String encPassword = EncryptionUtil.encrypt(EncryptionUtil
					.generatekey(String.valueOf(SecurityConstants.idConstant)),
					password);
			Customer customer = customerService
					.findCustomerbyUserNameAndPassword(user, encPassword,
							Integer.parseInt(merchantId));
			return customer != null;
		} catch (Exception e) {
			e.printStackTrace();
		}
		return false;
	}

	// set roles here
	private void assignPrincipal(Principal p) {
		if (!subject.getPrincipals().contains(p)) {
			subject.getPrincipals().add(p);
		}
	}

}



```
