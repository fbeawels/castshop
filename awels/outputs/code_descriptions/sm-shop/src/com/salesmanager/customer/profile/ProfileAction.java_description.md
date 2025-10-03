# ProfileAction.java

## Review

## 1. Summary

**Purpose & Functionality**  
`ProfileAction` is a Struts‑based controller that handles three main customer‑profile related workflows:

1. **Display Profile** – retrieves the logged‑in `Customer` and its `CustomerInfo` from the session and a DAO, then forwards to the view.
2. **Show Change‑Password Form** – simply returns a success view where the user can input a new password.
3. **Change Password** – validates user input, enforces a 6‑8 character policy, then delegates to a `CustomerLogonModule` to perform the password reset. On success the new customer object is re‑stored in the session and a confirmation message is set.

**Key Components**

| Class | Responsibility |
|-------|----------------|
| `ProfileAction` | Struts action handling profile UI |
| `CustomerService` | DAO layer for fetching `CustomerInfo` |
| `CustomerLogonModule` | Encapsulates authentication & password‑reset logic |
| `SessionUtil` | Helper for getting/setting the `Customer` in the HTTP session |
| `SalesManagerBaseAction` | Base action providing helper methods (`addFieldMessage`, `setMessage`, etc.) |
| `ServiceFactory` | Simple service locator used to obtain `CustomerService` |

The code relies on the **Struts 1.x** framework (e.g., `SUCCESS`, `ERROR`, `INPUT` constants) and on **Apache Commons Lang** (`StringUtils`). It also uses **log4j** for logging.

---

## 2. Detailed Description

### Flow of Execution

| Step | Action | Notes |
|------|--------|-------|
| 1 | `displayProfile` is invoked after a successful login. | Retrieves the customer from the session via `SessionUtil.getCustomer`. |
| 2 | Calls `CustomerService.findCustomerInfoById` to fetch personal data. | Service is acquired via `ServiceFactory`. |
| 3 | If any exception occurs, logs the error and returns `ERROR`. | Otherwise returns `SUCCESS`. |
| 4 | `changePasswordForm` simply forwards to the password‑change UI. |
| 5 | `changePassword` validates fields: not blank, matches, length 6‑8. | Adds field errors to the Struts context. |
| 6 | Calls `CustomerLogonModule.resetPassword` passing old & new passwords. | Catches `ServiceException` for invalid credentials. |
| 7 | On success, re‑stores the updated customer in the session and sets a success message. | Returns `SUCCESS`; on any error returns `INPUT`. |

### Assumptions & Constraints

- **Thread Safety** – Struts 1 actions are typically request‑scoped, but the class contains mutable fields (`customer`, `customerInfo`, passwords). If the action is pooled, these fields could leak between requests. Current design assumes a per‑request instance.
- **Password Policy** – Hard‑coded to 6‑8 characters; no complexity checks (uppercase, digits, symbols).
- **Service Locator** – Uses a static `ServiceFactory` rather than dependency injection; reduces testability.
- **Exception Handling** – All generic `Exception` blocks swallow stack traces after logging, which might hide root causes.

### Architecture & Design Choices

- **Action‑based MVC** – Classic Struts 1 pattern with a dedicated action per page.
- **Service Locator** – Keeps code tight but makes unit testing harder. A DI framework (Spring) could inject services instead.
- **Utility Logging** – Uses log4j directly; could abstract via a logging facade for easier replacement.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `displayProfile()` | Load customer data for profile view. | None | `String` (SUCCESS/ERROR) | Sets `customer` & `customerInfo` fields |
| `changePasswordForm()` | Show password change form. | None | `String` (SUCCESS) | None |
| `changePassword()` | Validate and update customer password. | None | `String` (SUCCESS/INPUT) | Updates session, adds messages, logs |
| `getCustomer()` / `setCustomer(Customer)` | Getter/Setter for `customer`. | - | `Customer` / void | - |
| `getCustomerInfo()` / `setCustomerInfo(CustomerInfo)` | Getter/Setter for `customerInfo`. | - | `CustomerInfo` / void | - |
| `getCurrentPassword()` / `setCurrentPassword(String)` | Getter/Setter for current password. | - | `String` / void | - |
| `getNewPassword()` / `setNewPassword(String)` | Getter/Setter for new password. | - | `String` / void | - |
| `getRepeatNewPassword()` / `setRepeatNewPassword(String)` | Getter/Setter for repeat new password. | - | `String` / void | - |

The getter/setter pairs are used by Struts to bind form fields to action properties.

---

## 4. Dependencies

| Library | Type | Purpose |
|---------|------|---------|
| `javax.servlet.http.HttpSession` | Standard | HTTP session handling |
| `org.apache.commons.lang.StringUtils` | Third‑party | String utility methods |
| `org.apache.log4j.Logger` | Third‑party | Logging |
| `com.salesmanager.common.SalesManagerBaseAction` | Internal | Base action providing message helpers |
| `com.salesmanager.core.constants.ErrorConstants` | Internal | Error code constants |
| `com.salesmanager.core.entity.customer.Customer` | Internal | Domain model |
| `com.salesmanager.core.entity.customer.CustomerInfo` | Internal | Domain model |
| `com.salesmanager.core.module.model.application.CustomerLogonModule` | Internal | Auth / password logic |
| `com.salesmanager.core.service.ServiceException` | Internal | Service‑layer exception |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Service locator |
| `com.salesmanager.core.service.customer.CustomerService` | Internal | DAO for customer data |
| `com.salesmanager.core.util.www.SessionUtil` | Internal | Session helper |
| `org.apache.struts.action.Action` (implied via `SalesManagerBaseAction`) | Framework | Struts action base |

All dependencies are either part of the Java EE stack or the custom `salesmanager` codebase, with no external open‑source libraries beyond Commons Lang and log4j.

---

## 5. Additional Notes

### Edge Cases & Potential Issues

1. **Thread Safety** – Mutable fields may survive across requests if Struts pools actions. Using `ThreadLocal` or request‑scoped beans would be safer.
2. **Password Strength** – The 6‑8 character rule is weak and may not satisfy modern security standards. Consider configurable complexity checks.
3. **Password Exposure** – Storing `newPassword` & `repeatNewPassword` in action fields exposes them to accidental logging or reflection. Prefer transient fields or clear them after use.
4. **Logging Sensitive Data** – The current code does not log passwords, which is good, but any future debugging that logs the action instance might inadvertently expose them.
5. **Service Locator** – Using `ServiceFactory` hampers unit testing. Replacing with Spring DI (e.g., `@Autowired` fields) would allow mocking services.
6. **Error Handling** – A generic `catch (Exception)` block masks the specific exception type, making debugging harder. Prefer narrower catch clauses.
7. **Internationalization** – Hard‑coded message keys (`messages.required.currentpassword`) rely on a proper message bundle. Missing keys could cause `NullPointerException` when retrieving texts.
8. **Session Synchronization** – After resetting the password, `SessionUtil.setCustomer(customer, request)` re‑writes the whole `Customer` object. If other concurrent requests modify the session, a race condition could occur.

### Suggested Enhancements

- **Use Struts 2 or Spring MVC** – Modern frameworks provide built‑in validation, dependency injection, and better action-scoped handling.
- **Add Validation Framework** – Move password checks into a dedicated validator (e.g., Apache Commons Validator or JSR‑303 Bean Validation).
- **Externalize Configuration** – Move password length, complexity rules, and error messages to a configuration file or database.
- **Improve Logging** – Use structured logging (e.g., SLF4J + Logback) and ensure sensitive data is never logged.
- **Unit Tests** – With DI, create mocks for `CustomerService` and `CustomerLogonModule` and write JUnit tests for each action method.
- **Security Review** – Verify that `resetPassword` uses secure hashing and salting; audit the module for potential timing attacks.

Overall, the code is straightforward and functional for a legacy Struts 1 application but would benefit from modernization, stronger security policies, and better testability.

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
package com.salesmanager.customer.profile;

import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.common.SalesManagerBaseAction;
import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.customer.CustomerInfo;
import com.salesmanager.core.module.model.application.CustomerLogonModule;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.www.SessionUtil;

public class ProfileAction extends SalesManagerBaseAction {

	private Logger log = Logger.getLogger(ProfileAction.class);
	private Customer customer;
	private CustomerInfo customerInfo;

	/** change password **/
	private String currentPassword;
	private String newPassword;
	private String repeatNewPassword;

	/**
	 * Displays Customer profile
	 * 
	 * @return
	 */
	public String displayProfile() {

		try {

			// get customer from HttpSession (login putted Customer in
			// HttpSession)
			customer = SessionUtil.getCustomer(super.getServletRequest());

			// get CustomerInfo
			CustomerService cservice = (CustomerService) ServiceFactory
					.getService(ServiceFactory.CustomerService);
			customerInfo = cservice.findCustomerInfoById(customer
					.getCustomerId());

		} catch (Exception e) {
			log.error(e);
			return ERROR;
		}

		return SUCCESS;

	}

	/**
	 * Displays change password form
	 * 
	 * @return
	 */
	public String changePasswordForm() {
		return SUCCESS;
	}

	/**
	 * Changes customer password
	 * 
	 * @return
	 */
	public String changePassword() {
		try {

			CustomerLogonModule logon = (CustomerLogonModule) com.salesmanager.core.util.SpringUtil
					.getBean("customerLogon");
			HttpSession session = getServletRequest().getSession();

			Customer customer = SessionUtil.getCustomer(super
					.getServletRequest());

			if (customer == null) {
				super.setTechnicalMessage();
				log.error("Customer does not exist in http session");
				return INPUT;
			}

			// new paswords match
			if (StringUtils.isBlank(this.getCurrentPassword())) {
				super.addFieldMessage("currentPassword",
						"messages.required.currentpassword");
				return INPUT;
			}

			if (StringUtils.isBlank(this.getNewPassword())) {
				super.addFieldMessage("newPassword",
						"messages.required.newpassword");
				return INPUT;
			}

			if (StringUtils.isBlank(this.getRepeatNewPassword())) {
				super.addFieldMessage("repeatNewPassword",
						"messages.required.repeatnewpassword");
				return INPUT;
			}

			if (!this.getNewPassword().equals(this.getRepeatNewPassword())) {
				super.addFieldMessage("repeatNewPassword",
						"messages.password.match");
				return INPUT;
			}

			// 6 to 8 characters
			if (this.getNewPassword().length() < 6
					|| this.getNewPassword().length() > 8) {
				super.addErrorMessage("messages.password.length");
				return INPUT;
			}

			logon.resetPassword(customer, getCurrentPassword(),
					getNewPassword());

			SessionUtil.setCustomer(customer, super.getServletRequest());

			super.setMessage("customer.changepassword.success.message");

		} catch (ServiceException e) {

			if (e.getReason() == ErrorConstants.INVALID_CREDENTIALS) {
				addActionError(getText("customer.changepassword.validation.invalid"));
			} else {
				log.error(e);
				addActionError(getText("errors.technical"));
			}
			return INPUT;
		} catch (Exception ex) {
			log.error(ex);
			super.setTechnicalMessage();
			return INPUT;
		}

		return SUCCESS;
	}

	public Customer getCustomer() {
		return customer;
	}

	public void setCustomer(Customer customer) {
		this.customer = customer;
	}

	public CustomerInfo getCustomerInfo() {
		return customerInfo;
	}

	public void setCustomerInfo(CustomerInfo customerInfo) {
		this.customerInfo = customerInfo;
	}

	public String getCurrentPassword() {
		return currentPassword;
	}

	public void setCurrentPassword(String currentPassword) {
		this.currentPassword = currentPassword;
	}

	public String getNewPassword() {
		return newPassword;
	}

	public void setNewPassword(String newPassword) {
		this.newPassword = newPassword;
	}

	public String getRepeatNewPassword() {
		return repeatNewPassword;
	}

	public void setRepeatNewPassword(String repeatNewPassword) {
		this.repeatNewPassword = repeatNewPassword;
	}

}



```
