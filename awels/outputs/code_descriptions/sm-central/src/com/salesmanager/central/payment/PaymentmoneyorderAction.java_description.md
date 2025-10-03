# PaymentmoneyorderAction.java

## Review

## 1. Summary

**Purpose**  
`PaymentmoneyorderAction` is a Struts‑like action class that implements CRUD operations for the “money‑order” payment module in the SalesManager e‑commerce platform.  
It handles:

| Operation | Method |
|-----------|--------|
| Display module configuration | `displayModule()` |
| Prepare module (load configuration) | `prepareModule()` |
| Save module configuration | `saveModule()` |
| Delete module configuration | `deleteModule()` |

**Key Components**

* **`moduleid`** – constant string `"moneyorder"` identifying the module.  
* **`MerchantService`** – service layer used to fetch and persist merchant configurations.  
* **`ConfigurationResponse`** – DTO holding a list of `MerchantConfiguration` objects.  
* **`MerchantConfiguration`** – entity representing a single key/value configuration pair.  

**Design Patterns / Libraries**

* *Service Factory* – `ServiceFactory.getService(...)` for obtaining the `MerchantService`.  
* *Data‑Transfer Object* – `ConfigurationResponse` for grouping configuration items.  
* *Validation* – `ValidationException` to signal missing required fields.  
* *Apache Commons Lang* – `StringUtils.isBlank()` for input validation.  
* *MVC/Action‑based architecture* – extends `PaymentModuleAction` (presumably a Struts action).

---

## 2. Detailed Description

### Execution Flow

1. **`prepareModule()`**  
   * Fetches the current merchant’s configuration for the module.  
   * Stores the `ConfigurationResponse` in the action’s `configurations` field.

2. **`displayModule()`**  
   * Retrieves the stored configuration (via `getConfigurations()`).  
   * Looks up the key `PaymentConstants.PAYMENT_MONEYORDERNAME`.  
   * If found, populates `payTo` (and would populate `address` if the commented code were active).  
   * Otherwise, falls back to the merchant store’s name/ address.

3. **`saveModule()`**  
   * Validates that `payTo` is not blank; adds a field error and throws `ValidationException` if it is.  
   * Retrieves or creates the `MerchantConfiguration` for the money‑order key.  
   * Persists the new/updated configuration via `MerchantService.saveOrUpdateMerchantConfiguration()`.

4. **`deleteModule()`**  
   * Retrieves all configurations for the module and deletes them in bulk.

### Assumptions & Constraints

* The action relies on a pre‑configured `ProfileConstants.context` attribute in the HTTP session.  
* It expects the `MerchantService` to be available via `ServiceFactory`.  
* Only a single configuration key (`PAYMENT_MONEYORDERNAME`) is handled; any additional keys (e.g., `address`) are currently ignored.  
* Error handling is rudimentary: any `Exception` propagates up the stack; the only specific exception is `ValidationException` for missing fields.

### Architecture & Design Choices

* **Loose Coupling via Service Factory** – avoids direct DAO usage but sacrifices compile‑time type safety.  
* **DTO‑Based Configuration** – grouping all merchant configurations in a single response object simplifies retrieval but can be inefficient if the merchant has many unrelated configs.  
* **No Dependency Injection** – services are pulled from a static factory each time; this makes unit testing harder and hides dependencies.  
* **Manual Field Validation** – uses `StringUtils.isBlank()` and manual error handling rather than framework validation annotations.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `deleteModule()` | Deletes all configurations for the money‑order module. | None (uses merchant ID from context). | None | Invokes `MerchantService.deleteMerchantConfigurations` |
| `displayModule()` | Loads current configuration for display. | None (reads from session). | Sets `payTo` (and potentially `address`). | None |
| `prepareModule()` | Loads configuration into `configurations`. | None. | Sets `configurations` field. | None |
| `saveModule()` | Persists user‑entered configuration. | Uses `payTo` field from form. | None | Calls `MerchantService.saveOrUpdateMerchantConfiguration` |
| `getConfigurations()` | Getter for `configurations`. | None. | `ConfigurationResponse` | None |
| `setConfigurations(ConfigurationResponse)` | Setter. | `ConfigurationResponse`. | None | None |
| `getPayTo()` / `setPayTo(String)` | Accessors for the pay‑to value. | `String`. | `String` | None |
| `getAddress()` / `setAddress(String)` | Accessors for address (currently unused). | `String`. | `String` | None |

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Commons Lang – used for null/blank checks. |
| `com.salesmanager.central.profile.Context` | Internal | Holds merchant ID, stored in session. |
| `com.salesmanager.central.profile.ProfileConstants` | Internal | Contains session key `context`. |
| `com.salesmanager.central.util.ValidationException` | Internal | Custom exception for validation failures. |
| `com.salesmanager.core.constants.PaymentConstants` | Internal | Constants for payment configuration keys. |
| `com.salesmanager.core.entity.merchant.MerchantConfiguration` | Internal | Entity for config key/value. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Internal | Entity representing the merchant store. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Factory for obtaining services. |
| `com.salesmanager.core.service.merchant.ConfigurationRequest` | Internal | Request DTO for configuration queries. |
| `com.salesmanager.core.service.merchant.ConfigurationResponse` | Internal | Response DTO containing a list of `MerchantConfiguration`. |
| `com.salesmanager.core.service.merchant.MerchantService` | Internal | Service layer for merchant operations. |

All dependencies are internal to the SalesManager codebase, except for the Apache Commons Lang library.

---

## 5. Additional Notes & Recommendations

### Code Quality & Maintainability

* **Raw Types** – `List confs` should use generics (`List<MerchantConfiguration>`).  
* **Unnecessary Variable** – `ConfigurationRequest requestvo` is created in `deleteModule()` but never used.  
* **Commented‑out Code** – The `address` handling is commented out. If not needed, remove it to avoid confusion.  
* **Null Handling** – `getConfigurations()` can return `null` (e.g., if `prepareModule()` never called). Methods that dereference it should guard against `NullPointerException`.  
* **Error Reporting** – `addFieldError` suggests integration with Struts. Ensure that the error messages are properly internationalised.  

### Performance & Scalability

* Deleting all configurations in a single bulk operation (`deleteMerchantConfigurations`) is efficient.  
* The configuration retrieval loads **all** merchant configurations for the module; if the module grows more complex (multiple keys), consider filtering at the service level.

### Security

* No obvious vulnerabilities, but ensure that `merchantid` cannot be spoofed via session manipulation.  
* Validate the `payTo` value against potential injection or formatting attacks before persisting.

### Extensibility

* **Address Field** – If required in future, uncomment and implement logic consistently.  
* **Multiple Config Keys** – Refactor to handle an arbitrary set of keys rather than a hard‑coded constant.  
* **Dependency Injection** – Switch to constructor/field injection (e.g., Spring) for easier unit testing.  
* **Logging** – Add SLF4J/Log4j logging to aid troubleshooting.  

### Testing

* Unit tests should mock `MerchantService` and validate:
  * Correct service method calls.
  * Proper handling of missing `payTo`.
  * Creation of new configuration versus update of existing.  
* Integration tests can verify end‑to‑end persistence in a test database.

---

**Overall**  
The class provides the essential CRUD functionality for a single payment module but exhibits several opportunities for improvement in type safety, code clarity, and testability. Refactoring along the lines suggested above would make the component more robust, maintainable, and easier to extend.

## Code Critique



## Code Preview

```java
/*
 * Provided by CSTI Consulting 
 * Following GNU LESSER GENERAL PUBLIC LICENSE
 * You may obtain more details at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.central.payment;

import java.util.List;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.ValidationException;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;

public class PaymentmoneyorderAction extends PaymentModuleAction {

	private final static String moduleid = "moneyorder";

	private ConfigurationResponse configurations;
	private String payTo;
	private String address;

	@Override
	public void deleteModule() throws Exception {

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		ConfigurationRequest requestvo = new ConfigurationRequest(super
				.getContext().getMerchantid().intValue());
		ConfigurationResponse responsevo = mservice.getConfigurationByModule(
				moduleid, super.getContext().getMerchantid());

		List confs = responsevo.getMerchantConfigurationList();
		if (confs != null) {
			mservice.deleteMerchantConfigurations(confs);
		}

	}

	@Override
	public void displayModule() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		// get payto / address
		ConfigurationResponse vo = this.getConfigurations();
		MerchantConfiguration conf = (MerchantConfiguration) vo
				.getConfiguration(PaymentConstants.PAYMENT_MONEYORDERNAME);

		if (conf != null) {
			this.setPayTo(conf.getConfigurationValue());
			// this.setAddress(conf.getConfigurationValue1());
		} else { // get store information
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice.getMerchantStore(merchantid);
			if (store != null) {
				this.setPayTo(store.getStorename());
				// this.setAddress(store.getStoreaddress());
			}
		}
	}

	@Override
	public void prepareModule() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		ConfigurationResponse config = mservice.getConfigurationByModule(
				moduleid, merchantid);
		this.setConfigurations(config);

	}

	@Override
	public void saveModule() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		boolean fielderror = false;

		if (StringUtils.isBlank(this.getPayTo())) {
			addFieldError("payTo", getText("error.payment.payto.required"));
			fielderror = true;
		}



		if (fielderror) {
			throw new ValidationException("Missing fields");
		}

		ConfigurationResponse vo = this.getConfigurations();
		MerchantConfiguration conf = null;
		if (vo != null) {
			conf = (MerchantConfiguration) vo
					.getConfiguration(PaymentConstants.PAYMENT_MONEYORDERNAME);
		}
		if (conf == null) {
			conf = new MerchantConfiguration();
			conf.setMerchantId(merchantid);
			conf.setConfigurationModule(moduleid);
			conf.setConfigurationKey(PaymentConstants.MODULE_PAYMENT
					+ PaymentConstants.PAYMENT_MONEYORDERNAME);

		}

		conf.setConfigurationValue(this.getPayTo());
		// conf.setConfigurationValue1(this.getAddress());

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		mservice.saveOrUpdateMerchantConfiguration(conf);

	}

	public ConfigurationResponse getConfigurations() {
		return configurations;
	}

	public void setConfigurations(ConfigurationResponse configurations) {
		this.configurations = configurations;
	}

	public String getPayTo() {
		return payTo;
	}

	public void setPayTo(String payTo) {
		this.payTo = payTo;
	}

	public String getAddress() {
		return address;
	}

	public void setAddress(String address) {
		this.address = address;
	}

}



```
