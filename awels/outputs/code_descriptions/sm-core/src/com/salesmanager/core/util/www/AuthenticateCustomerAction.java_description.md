# AuthenticateCustomerAction.java

## Review

## 1. Summary  

**Purpose**  
`AuthenticateCustomerAction` is a server‑side action class that handles customer authentication (log‑in, log‑out, and retrieval of customer data). It interacts with a custom `CustomerLogonModule` (responsible for the actual credential verification) and updates `CustomerInfo` records (last login time, login count).  

**Key Components**  
| Component | Role |
|-----------|------|
| `CustomerLogonModule` | Authenticates username/password and provides an authentication token |
| `CustomerService` | Persists and retrieves `CustomerInfo` (meta‑data about the customer) |
| `SessionUtil` | Stores the authenticated `Customer` in the HTTP session |
| `SpringUtil` | Bean lookup for the `CustomerLogonModule` |
| `PropertiesUtil` | Loads configuration (e.g., login timeout) |
| `BaseAction` | Inherits common action functionality (e.g., request/locale helpers) |

**Design Patterns & Frameworks**  
- **Action Pattern** (likely part of a Struts‑like MVC framework).  
- **Dependency Injection (manual)**: `SpringUtil.getBean` is used to obtain the `CustomerLogonModule`.  
- **Singleton Service Factory** (`ServiceFactory.getService`) for retrieving `CustomerService`.  
- **Internationalization** via `getText(...)` (assumed to be provided by `BaseAction`).  

The code is tightly coupled to servlet APIs and to a proprietary framework (Sales Manager Core).

---

## 2. Detailed Description  

### Execution Flow  

| Step | Method | What Happens | Notes |
|------|--------|--------------|-------|
| **Login** | `sendCustomerInformation()` → `logon()` | 1. Calls `logon()`. 2. If a `Customer` is set, attaches it to the request. | Returned value is a string used by the framework (`SUCCESS` or exception). |
| **Logon()** | `logon()` | 1. Validates presence of `username` & `password`. 2. Obtains the `CustomerLogonModule`. 3. Retrieves the merchant ID from the session (default 1). 4. Calls `logon()` on the module to authenticate. 5. Updates `CustomerInfo` (last login date, login count). 6. Stores the `Customer` in session & request. 7. Builds a JSON‑style response (returnCode, authToken, messages). | Throws `ServiceException` for invalid credentials; generic `Exception` for other errors. |
| **Logout** | `logout()` | 1. Calls `logout()` on the `CustomerLogonModule`. 2. Updates `CustomerInfo` (last login date). | Catches and logs all exceptions, but never propagates them. |
| **Retrieve Customer** | `logonCustomer()` | Calls `logon()` and returns the authenticated `Customer`. | Unused in this snippet; presumably used by other actions. |

### Initialization & Cleanup  

- **Initialization**: The class relies on inherited `BaseAction` for request/session handling. No explicit constructor logic.  
- **Cleanup**: No explicit resource cleanup. Exceptions are logged but not always propagated (especially in `logout()`).

### Assumptions & Constraints  

- The servlet environment is already configured with a Spring context.  
- `CustomerLogonModule` is a singleton bean.  
- `CustomerService` is thread‑safe (retrieved from a factory).  
- `getText()` provides localized messages; missing messages are commented out.  
- `customer` is assumed non‑null after successful login.

### Architectural Choices  

- **Coupling to Servlet**: The action directly manipulates `HttpServletRequest` & `HttpSession`.  
- **Manual DI**: Uses static lookup (`SpringUtil.getBean`) rather than constructor/setter injection.  
- **Response Building**: Sets attributes on the request instead of writing JSON/XML directly.  
- **Error Handling**: Uses custom `ServiceException` for validation failures; generic `Exception` for other errors.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects | Notes |
|--------|---------|------------|--------|--------------|-------|
| `sendCustomerInformation()` | Entry point for customer login. Calls `logon()` and exposes the customer to the view. | None | `String` (framework result) | Sets request attribute `"customer"`. | May throw checked exceptions. |
| `logout()` | Logs the customer out and updates last login metadata. | None | `String` (framework result) | Calls `CustomerLogonModule.logout`; updates `CustomerInfo`. | Swallows all exceptions – could hide errors. |
| `logonCustomer()` | Convenience method returning the logged‑in `Customer`. | None | `Customer` | Calls `logon()`. | Declared but not used within the class. |
| `logon()` | Core login routine: validation, authentication, metadata update, session handling, response preparation. | None | `String` (framework result) | Modifies request attributes, session, `CustomerInfo`. | Throws `ServiceException` for invalid credentials. |
| `getAuthenticatedToken(Customer)` | Delegates to `CustomerLogonModule` to obtain a token with a configurable timeout. | `Customer` | `String` (token) | None | Uses a hardcoded default timeout if property missing. |
| `prepareResponse(HttpServletRequest, String, String, String)` | Populates the request with response data (returnCode, token, messages). | `HttpServletRequest`, `returnCode`, `authenticationToken`, `messages` | None | Sets request attributes. | No validation of arguments. |
| `getStrMessages(List<String>)` | Concatenates a list of messages using `", "` separator. | `List<String>` | `String` | None | Returns empty string if list is null/empty. |
| `validateCustomerLogon(List<String>, HttpServletRequest)` | Checks presence of `username` and `password` parameters. | `List<String>`, `HttpServletRequest` | `boolean` | Adds messages to list (currently commented out). | No trimming or further validation (e.g., length, pattern). |

---

## 4. Dependencies  

| Dependency | Type | Role |
|------------|------|------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String utility (blank check). |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.core.*` | Proprietary | Core domain entities, services, constants, utilities. |
| `com.salesmanager.core.module.model.application.CustomerLogonModule` | Proprietary | Authentication service. |
| `com.salesmanager.core.service.ServiceFactory` | Proprietary | Factory for retrieving services. |
| `com.salesmanager.core.util.SpringUtil` | Proprietary | Spring bean lookup. |
| `javax.servlet.http.*` | Standard | Servlet API. |
| `java.util.*` | Standard | Collections, Date, Locale. |

All external dependencies are either standard Java EE libraries or proprietary components of the Sales Manager Core platform. No platform‑specific assumptions beyond servlet containers and a Spring application context.

---

## 5. Additional Notes & Recommendations  

### 5.1 Potential Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **NPE on `customerInfoNumberOfLogon`** | `login = login + 1;` will throw `NullPointerException` if the field is null. | Initialize to `0` if null: `int login = Optional.ofNullable(customerInfo.getCustomerInfoNumberOfLogon()).orElse(0);` |
| **Unvalidated Input** | No length or pattern checks on `username`/`password`. | Add validation (e.g., regex, max length) and return meaningful error messages. |
| **Hard‑coded Merchant ID** | Uses `1` when no store is in session; may mask misconfiguration. | Throw an exception or log a warning if `store` is null. |
| **Thread Safety** | `Customer` field is an instance variable; actions are usually singletons. | Remove instance field; use local variable or store in request/session. |
| **Exception Handling in `logout()`** | Swallows all exceptions, potentially hiding real problems. | Log at error level and rethrow or return an error result. |
| **Manual Bean Lookup** | Tight coupling to Spring; harder to test. | Inject `CustomerLogonModule` via constructor/setter or use Spring MVC. |
| **Token Expiry Hard‑coded** | 360000 ms (6 min) may not be appropriate. | Make timeout a configurable property and expose via `@Value`/environment. |
| **No CSRF Protection** | Authentication endpoint likely vulnerable. | Add CSRF tokens or use framework‑provided protection. |
| **Error Messages Are Commented Out** | Users receive no detailed feedback on missing fields. | Un-comment or replace with proper i18n messages. |
| **`prepareResponse` Uses Request Attributes** | Could leak data into the view layer; not ideal for RESTful services. | Consider returning a JSON object or using `HttpServletResponse`. |
| **Locale Handling** | `customer.setLocale(locale);` but locale may not match user’s preferences. | Store locale in `Customer` only if user explicitly selects it. |

### 5.2 Code Quality & Style  

- **Redundant Casting**: `CustomerLogonModule logon = (CustomerLogonModule) SpringUtil.getBean("customerLogon");` – casting can be avoided by defining the bean’s type in `SpringUtil`.
- **Magic Strings**: `"STORE"`, `"CUSTOMER"`, `"CUSTOMER_PARAM"` – consider constants or enums.
- **Logging**: Use parameterized logs (e.g., `log.error("Error logging out customer {}", customer.getCustomerId(), e);`).
- **Resource Leaks**: None detected, but consider closing any external resources if added later.
- **Documentation**: Javadoc comments are absent; add brief method descriptions and param/return notes.

### 5.3 Future Enhancements  

1. **Dependency Injection** – Switch to Spring MVC’s `@Controller` and autowire `CustomerLogonModule` & `CustomerService`.  
2. **RESTful API** – Replace request/response attribute pattern with a JSON REST endpoint (`@RestController`).  
3. **Unit Tests** – With DI, unit tests become easier; mock services and test validation logic.  
4. **Security Improvements** – Add rate limiting, account lockout after multiple failed attempts, and secure token handling (JWT).  
5. **Internationalization** – Ensure all messages are localized and loaded via a resource bundle.  
6. **Centralized Error Handling** – Use an exception‑handling filter or controller advice to manage `ServiceException`.  
7. **Logging Enhancements** – Add correlation IDs to trace requests across services.

---

### Verdict  

The class implements the basic login/logout flow required by the application and demonstrates a clear separation of concerns between authentication logic and persistence. However, it suffers from several design shortcomings—particularly tight coupling to the servlet environment, manual bean lookups, and potential null‑pointer pitfalls. Refactoring towards a more modern, dependency‑injection‑friendly architecture (Spring MVC / REST) would greatly improve maintainability, testability, and security.

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
package com.salesmanager.core.util.www;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Locale;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.customer.CustomerInfo;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.module.model.application.CustomerLogonModule;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.PropertiesUtil;

public class AuthenticateCustomerAction extends BaseAction {

	private static final long serialVersionUID = 1L;
	private static final int ERROR_CODE = -1;
	private static final int SUCCESS_CODE = 1;
	private static final String STR_SEPERATOR = ", ";
	private static final String CUSTOMER_PARAM = "customer";

	private Customer customer = null;
	private Logger log = Logger.getLogger(AuthenticateCustomerAction.class);

	public String sendCustomerInformation() throws Exception {

		String returnStr = logon();
		if (customer != null) {
			getServletRequest().setAttribute(CUSTOMER_PARAM, customer);
		}
		return returnStr;
	}

	public String logout() throws Exception {

		try {

			CustomerLogonModule logon = (CustomerLogonModule) com.salesmanager.core.util.SpringUtil
					.getBean("customerLogon");

			logon.logout(getServletRequest());

			// get CustomerInfo
			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			CustomerInfo customerInfo = cservice.findCustomerInfoById(customer
					.getCustomerId());

			if (customerInfo == null) {
				customerInfo = new CustomerInfo();
				customerInfo.setCustomerInfoId(customer.getCustomerId());
			}

			customerInfo.setCustomerInfoDateOfLastLogon(new Date());
			cservice.saveOrUpdateCustomerInfo(customerInfo);

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	protected Customer logonCustomer() throws ServiceException, Exception {

		this.logon();
		return customer;
	}

	public String logon() throws ServiceException, Exception {
		List<String> messages = new ArrayList<String>();
		if (!validateCustomerLogon(messages, getServletRequest())) {
			// prepareResponse(getServletRequest(),String.valueOf(ERROR_CODE),"",
			// getStrMessages(messages));
			ServiceException sException = new ServiceException(getText("login.invalid"));
			sException.setReason(ErrorConstants.INVALID_CREDENTIALS);
			throw sException;
			// return SUCCESS;
		}
		try {

			CustomerLogonModule logon = (CustomerLogonModule) com.salesmanager.core.util.SpringUtil
					.getBean("customerLogon");

			// get merchantId
			int merchantId = 1;
			HttpSession session = getServletRequest().getSession();
			MerchantStore store = (MerchantStore) session.getAttribute("STORE");
			if (store != null) {
				merchantId = store.getMerchantId();
			}

			customer = logon.logon(getServletRequest(), merchantId);
			Locale locale = super.getLocale();
			customer.setLocale(locale);

			// get CustomerInfo
			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			CustomerInfo customerInfo = cservice.findCustomerInfoById(customer
					.getCustomerId());

			if (customerInfo == null) {
				customerInfo = new CustomerInfo();
				customerInfo.setCustomerInfoId(customer.getCustomerId());
			}

			Integer login = customerInfo.getCustomerInfoNumberOfLogon();
			login = login + 1;
			customerInfo.setCustomerInfoNumberOfLogon(login);
			cservice.saveOrUpdateCustomerInfo(customerInfo);

			SessionUtil.setCustomer(customer, getServletRequest());
			getServletRequest().setAttribute("CUSTOMER", customer);
		} catch (ServiceException e) {
			messages.add(getText("login.invalid"));
			prepareResponse(getServletRequest(), String.valueOf(ERROR_CODE),
					"", getStrMessages(messages));
			// return SUCCESS;
			throw e;
		} catch (Exception ex) {
			log.error(ex);
			messages.add(getText("errors.technical"));
			prepareResponse(getServletRequest(), String.valueOf(ERROR_CODE),
					"", getStrMessages(messages));
			// return SUCCESS;
			throw ex;
		}

		if (customer != null) {
			messages.add(getText("login.successfull"));
			prepareResponse(getServletRequest(), String.valueOf(SUCCESS_CODE),
					getAuthenticatedToken(customer), getStrMessages(messages));

		} else {
			messages.add(getText("login.invalid"));
			prepareResponse(getServletRequest(), String.valueOf(ERROR_CODE),
					"", getStrMessages(messages));
		}
		return SUCCESS;
	}

	private String getAuthenticatedToken(Customer customer) {
		CustomerLogonModule logon = (CustomerLogonModule) com.salesmanager.core.util.SpringUtil
				.getBean("customerLogon");
		return logon.getAuthToken(customer, PropertiesUtil.getConfiguration()
				.getLong("core.login.timeout", 360000));
	}

	private void prepareResponse(HttpServletRequest request, String returnCode,
			String authenticationToken, String messages) {
		request.setAttribute("returnCode", returnCode);
		request.setAttribute("authenticationToken", authenticationToken);
		request.setAttribute("messages", messages);
	}

	private static String getStrMessages(List<String> messages) {
		StringBuilder builder = null;
		for (String message : messages) {
			if (builder == null) {
				builder = new StringBuilder();
				builder.append(message);
			} else {
				builder.append(STR_SEPERATOR).append(message);
			}
		}
		return (builder != null) ? builder.toString() : "";
	}

	private boolean validateCustomerLogon(List<String> messages,
			HttpServletRequest request) {
		boolean isValid = true;
		String username = request.getParameter("username");
		String password = request.getParameter("password");
		if (StringUtils.isBlank(username)) {
			// messages.add(getText("login.empty.username"));
			isValid = false;
		}
		if (StringUtils.isBlank(password)) {
			// messages.add(getText("login.empty.password"));
			isValid = false;
		}
		return isValid;
	}

}



```
