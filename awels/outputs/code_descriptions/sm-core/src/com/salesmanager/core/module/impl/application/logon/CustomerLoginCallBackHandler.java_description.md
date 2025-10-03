# CustomerLoginCallBackHandler.java

## Review

## 1. Summary
This class, `CustomerLoginCallBackHandler`, implements the `javax.security.auth.callback.CallbackHandler` interface to supply authentication data (username, password, and merchant ID) to an authentication mechanism that expects `NameCallback`, `PasswordCallback`, and `TextInputCallback`. It is a small, self‑contained helper used in the *Sales Manager* core module.

**Key components**

| Class | Purpose |
|-------|---------|
| `CustomerLoginCallBackHandler` | Stores login credentials and translates them into callback objects for the underlying authentication framework. |

**Design patterns / libraries**

* Implements the **CallbackHandler** pattern from JAAS (Java Authentication and Authorization Service).  
* Uses only JDK APIs – no third‑party dependencies.

---

## 2. Detailed Description
The handler is constructed with a username, password, and an integer `merchantId`. The `handle(Callback[])` method iterates over the provided callbacks, detects their concrete type, and sets the corresponding value:

1. **NameCallback** → `setName(userName)`  
2. **PasswordCallback** → `setPassword(password.toCharArray())`  
3. **TextInputCallback** → `setText(String.valueOf(merchantId))`

The method declares `throws IOException, UnsupportedCallbackException` as required by the interface but never throws them explicitly – the only potential exception would be a `NullPointerException` if a callback were `null`, which is unlikely in normal JAAS usage.

The class is lightweight, stateless after construction, and can be reused for multiple authentication attempts as long as the credentials remain unchanged.

---

## 3. Functions/Methods

| Method | Signature | Purpose | Inputs | Outputs | Side Effects |
|--------|-----------|---------|--------|---------|--------------|
| `CustomerLoginCallBackHandler(String userName, String password, int merchantId)` | Constructor | Stores credentials for later use. | `userName`, `password`, `merchantId` | none (initializes fields) | none |
| `void handle(Callback[] callbacks)` | Implements `CallbackHandler.handle` | Fills provided callbacks with the stored credentials. | `callbacks` – array of `Callback` objects | None – callbacks are mutated | May throw `IOException` or `UnsupportedCallbackException` (declared, but not actually thrown) |

**Reusable utilities**: None – the class is a simple data holder.

---

## 4. Dependencies

| Library | Type | Notes |
|---------|------|-------|
| `javax.security.auth.callback.*` | JDK (standard) | JAAS callbacks. |
| `java.io.*` | JDK (standard) | For exception types. |

No external or platform‑specific dependencies.

---

## 5. Additional Notes

### Strengths
* **Simplicity** – clear one‑to‑one mapping between credentials and callbacks.  
* **Thread‑safety** – immutable after construction (no mutable state accessed concurrently).  
* **Compliance** – follows JAAS contract.

### Potential Improvements / Edge Cases
| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Unnecessary imports** | Minor bloat | Remove unused `CallbackHandler`, `Callback`, and `UnsupportedCallbackException` imports that are already covered by the used classes. |
| **No null checks** | NPE if `userName`, `password`, or callbacks contain nulls | Validate constructor arguments and skip null callbacks in `handle`. |
| **Password handling** | The password is kept as a plain `String`; copying to a `char[]` only occurs when setting the callback, but the original string remains in memory. | Store password in a `char[]` initially, and clear it after use. |
| **Unsupported callback types** | Silent no‑op for unexpected callbacks; could hide bugs. | Throw `UnsupportedCallbackException` if an unknown callback type is encountered. |
| **Encoding assumptions** | `String.valueOf(merchantId)` always uses decimal; if the authentication mechanism expects a different format, this will fail. | Document the expected format or allow a formatter strategy. |
| **Exception contract** | Declares `IOException` and `UnsupportedCallbackException` but never throws them; callers may misinterpret. | Either remove the declarations or actually throw them for unsupported types. |

### Future Enhancements
1. **Builder pattern** – for more flexible credential construction (e.g., optional merchant ID).  
2. **Credential encryption** – store password securely in memory (char array, zeroed after use).  
3. **Logging / metrics** – track how many callbacks are handled for debugging.  
4. **Unit tests** – cover all callback types, null handling, and exception paths.

Overall, the class fulfills its intended role with minimal code, but small defensive programming adjustments would improve robustness and security.

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

import javax.security.auth.callback.Callback;
import javax.security.auth.callback.CallbackHandler;
import javax.security.auth.callback.NameCallback;
import javax.security.auth.callback.PasswordCallback;
import javax.security.auth.callback.TextInputCallback;
import javax.security.auth.callback.UnsupportedCallbackException;

public class CustomerLoginCallBackHandler implements CallbackHandler {

	private String userName;
	private String password;
	private int merchantId;

	public CustomerLoginCallBackHandler(String userName, String password,
			int merchantId) {
		this.userName = userName;
		this.password = password;
		this.merchantId = merchantId;
	}

	public void handle(Callback[] callbacks) throws IOException,
			UnsupportedCallbackException {
		for (Callback callBack : callbacks) {
			if (callBack instanceof NameCallback) {
				((NameCallback) callBack).setName(userName);
			}
			if (callBack instanceof PasswordCallback) {
				((PasswordCallback) callBack).setPassword(password
						.toCharArray());
			}
			if (callBack instanceof TextInputCallback) {
				((TextInputCallback) callBack).setText(String
						.valueOf(merchantId));
			}
		}
	}

}



```
