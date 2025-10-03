# PaymentauthorizenetAction.java

## Review

## 1. Summary

| Item | Description |
|------|-------------|
| **Purpose** | Implements the Authorize.Net payment module integration for the SalesManager central application. It handles CRUD‑like operations on the module configuration, validates user input, and persists encrypted credentials and gateway properties in the merchant configuration store. |
| **Key Components** | *`PaymentauthorizenetAction`* – Extends a generic `PaymentModuleAction` and overrides lifecycle hooks (`displayModule`, `prepareModule`, `saveModule`, `deleteModule`).<br>*`IntegrationKeys`* – Holds the API user ID and transaction key.<br>*`IntegrationProperties`* – Holds three custom property strings used by the Authorize.Net gateway.<br>*`MerchantService`* – Service layer that reads and writes `MerchantConfiguration` entities.<br>*`EncryptionUtil`* – Encrypts the concatenated credentials with a key derived from the merchant ID. |
| **Frameworks/Libraries** | • **Apache Commons Lang** (StringUtils)<br>• Custom **SalesManager** core services (ServiceFactory, MerchantService, EncryptionUtil, etc.)<br>• Likely a web MVC framework (e.g., Struts/JSF) inferred from the `getServletRequest()` call. |

## 2. Detailed Description

1. **Initialization (`prepareModule`)**  
   * Retrieves the current merchant context from the HTTP session.<br>* Calls `MerchantService.getConfigurationByModule(moduleid, merchantid)` to obtain the configuration bundle for Authorize.Net.<br>* Stores the returned `ConfigurationResponse` for later use.

2. **Display (`displayModule`)**  
   * Extracts `keys` and `properties` from the `ConfigurationResponse` (if present) and populates the action’s fields.

3. **Save (`saveModule`)**  
   * Validates that the user ID and transaction key are not blank, raising a `ValidationException` on error.<br>* Generates an encryption key based on the merchant ID, concatenates the credentials (`userid;N;transactionKey`), encrypts the string, and prepares the property string (`props1;props2;props3`).<br>* Loads an existing `MerchantConfiguration` for the module; if none exists, creates a new one.<br>* Persists the configuration via `MerchantService.saveOrUpdateMerchantConfiguration`.

4. **Delete (`deleteModule`)** – Not implemented; currently a no‑op.

5. **Lifecycle & Cleanup** – No explicit cleanup; relies on the framework’s request‑scoped lifecycle. The action’s instance fields (`keys`, `properties`, `configurations`) are reused per request.

**Assumptions & Constraints**

* The action is assumed to be **request‑scoped**; otherwise, concurrent requests could overwrite shared fields (`keys`, `properties`).
* `EncryptionUtil.generatekey()` and `EncryptionUtil.encrypt()` are deterministic and safe for use with the merchant ID.
* The configuration store accepts semicolon‑separated strings for `configurationValue` and `configurationValue1/2/3`.
* The merchant ID is always present in the session (`Context`).

**Architectural Notes**

* The action follows a **template method** pattern: the superclass `PaymentModuleAction` defines abstract lifecycle hooks that concrete modules override.
* The code mixes **validation** (field checks) and **business logic** (configuration persistence) in a single method; separation could improve testability.
* The service layer (`MerchantService`) abstracts persistence, suggesting a **Service/DAO** separation.

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `deleteModule()` | Intended to delete the Authorize.Net module configuration. Currently empty. | None | None | Should remove config from DB. |
| `displayModule()` | Populate `keys` and `properties` from existing configuration for rendering. | None | None | Sets internal fields. |
| `prepareModule()` | Retrieve the current merchant’s configuration for the module. | None | None | Stores `ConfigurationResponse`. |
| `saveModule()` | Validate user input, encrypt credentials, persist configuration. | None (uses internal fields) | None | Throws `ValidationException` on missing fields, persists via `MerchantService`. |
| `getConfigurations()` | Getter for the `ConfigurationResponse`. | None | `ConfigurationResponse` | None |
| `setConfigurations(ConfigurationResponse)` | Setter for the configuration response. | `ConfigurationResponse` | None | Sets internal field. |
| `getKeys()` | Getter for `IntegrationKeys`. | None | `IntegrationKeys` | None |
| `setKeys(IntegrationKeys)` | Setter for keys. | `IntegrationKeys` | None | Sets internal field. |
| `getProperties()` | Getter for `IntegrationProperties`. | None | `IntegrationProperties` | None |
| `setProperties(IntegrationProperties)` | Setter for properties. | `IntegrationProperties` | None | Sets internal field. |

*Utility methods:*  
`addFieldError(String field, String message)` – inherited from `PaymentModuleAction` (presumably adds validation errors to the framework’s error map).  
`getText(String key)` – fetches localized error messages.

## 4. Dependencies

| Library/Framework | Usage | Type |
|-------------------|-------|------|
| `org.apache.commons.lang.StringUtils` | Blank/empty checks | Third‑party (Apache Commons Lang) |
| `com.salesmanager.central.profile.Context` | Session context retrieval | Internal |
| `com.salesmanager.central.profile.ProfileConstants` | Session attribute key | Internal |
| `com.salesmanager.central.util.ValidationException` | Validation error propagation | Internal |
| `com.salesmanager.core.constants.PaymentConstants` | Constant strings for config keys | Internal |
| `com.salesmanager.core.entity.merchant.MerchantConfiguration` | Configuration entity | Internal |
| `com.salesmanager.core.service.ServiceFactory` | Service lookup | Internal |
| `com.salesmanager.core.service.common.model.IntegrationKeys` | Holds API credentials | Internal |
| `com.salesmanager.core.service.common.model.IntegrationProperties` | Holds gateway properties | Internal |
| `com.salesmanager.core.service.merchant.ConfigurationResponse` | Composite response wrapper | Internal |
| `com.salesmanager.core.service.merchant.MerchantService` | CRUD for merchant config | Internal |
| `com.salesmanager.core.util.EncryptionUtil` | Key generation / encryption | Internal |
| Web MVC framework (e.g., Struts) | `getServletRequest()` | Platform‑specific |

All dependencies are either internal to the SalesManager codebase or well‑known third‑party libraries. No native or platform‑specific APIs are invoked directly.

## 5. Additional Notes

### 5.1 Edge Cases & Missing Functionality
* **`deleteModule()`**: No implementation; attempts to delete the configuration will silently fail, potentially leaving stale data.
* **Concurrency**: The action stores mutable state (`keys`, `properties`, `configurations`) as instance fields. If the action bean is shared between requests, concurrent access could corrupt data. Verify that the framework creates a fresh instance per request.
* **NPE Risks**: `this.getConfigurations()` can return `null`; subsequent casts (`vo.getConfiguration(...)`) would throw an NPE. Defensive checks are missing.
* **Credential Separation**: Uses semicolons (`;`) as separators in the credential string. If any credential contains a semicolon, parsing will break. A more robust format (e.g., JSON, Base64) would be safer.
* **Encryption Key**: Derives the key from `String.valueOf(merchantid)`. If the merchant ID changes or is reused elsewhere, this could create key collisions or weaken security.
* **Property Handling**: `configurationValue1` is always set to an empty string, yet the code appends properties to `configurationValue2`. The purpose of `configurationValue1` is unclear.
* **Validation Messages**: `addFieldError` references localization keys (`error.payment.loginid.required`, etc.). If those keys are missing from the resource bundle, the error message will be blank.

### 5.2 Security Considerations
* Credentials are stored in the database encrypted with a key derived solely from the merchant ID. If the encryption algorithm or key derivation is weak, an attacker who obtains the database could decrypt the credentials.
* The code never logs or displays the raw credentials, which is good, but it also never masks them when re‑displaying the form for editing. This could expose them in logs or browser history if the form action is mishandled.
* Using `StringBuffer` for building the credential string is unnecessary; `StringBuilder` (non‑synchronized) would be more efficient.

### 5.3 Potential Enhancements
1. **Implement `deleteModule()`** – Remove the Authorize.Net configuration from the store and handle orphaned references.
2. **Improve Thread Safety** – Mark the action as request‑scoped or make all mutable fields local to the method to avoid cross‑request contamination.
3. **Robust Serialization** – Replace semicolon‑separated strings with JSON or Base64‑encoded bytes for both credentials and properties.
4. **Validation Layer** – Separate input validation from persistence logic (e.g., use a validator class or annotations) to improve testability.
5. **Exception Handling** – Wrap service calls in try/catch blocks and provide user‑friendly error messages for persistence failures.
6. **Logging** – Add proper logging (without leaking secrets) for debugging and audit trails.
7. **Unit Tests** – Write tests for `saveModule()` to cover successful persistence, validation failures, and encryption correctness.

### 5.4 Code Style & Readability
* Variable names are mostly descriptive, but the class name `PaymentauthorizenetAction` would be clearer as `AuthorizeNetPaymentAction` (Java naming convention: capitalized words).
* The code uses `new StringBuffer()` where `StringBuilder` would suffice; `String` concatenation is even simpler in modern Java.
* Magic strings (e.g., `"N"`, semicolons) could be constants for maintainability.
* Inline comments are minimal; adding brief comments above each method would aid future maintainers.

---

**Overall Assessment**  
The action implements the core CRUD functionality for Authorize.Net integration in a concise manner, leveraging existing service layers and utilities. However, it exhibits several design gaps (missing deletion logic, potential concurrency issues, fragile credential serialization) and could benefit from stronger validation, error handling, and security hardening. Addressing these points would make the module more robust, maintainable, and secure.

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

import org.apache.commons.lang.StringUtils;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.ValidationException;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.EncryptionUtil;

public class PaymentauthorizenetAction extends PaymentModuleAction {

	private final static String moduleid = "authorizenet";

	private IntegrationProperties properties = new IntegrationProperties();
	private IntegrationKeys keys = new IntegrationKeys();

	private ConfigurationResponse configurations;

	@Override
	public void deleteModule() throws Exception {
		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();



	}

	@Override
	public void displayModule() throws Exception {
		ConfigurationResponse vo = this.getConfigurations();
		IntegrationKeys k = (IntegrationKeys) vo.getConfiguration("keys");
		if (k != null) {
			this.setKeys(k);
		}

		IntegrationProperties p = (IntegrationProperties) vo
				.getConfiguration("properties");
		if (p != null) {
			this.setProperties(p);
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

		boolean hasError = false;

		if (StringUtils.isBlank(this.getKeys().getUserid())) {
			addFieldError("keys.userid",
					getText("error.payment.loginid.required"));
			hasError = true;
		}

		if (StringUtils.isBlank(this.getKeys().getTransactionKey())) {
			addFieldError("keys.transactionKey",
					getText("error.payment.transactionkey.required"));
			hasError = true;
		}

		if (hasError) {
			throw new ValidationException("Missing fields");
		}

		String key = EncryptionUtil.generatekey(String.valueOf(merchantid));

		String credentials = new StringBuffer().append(getKeys().getUserid())
				.append(";").append("N").append(";").append(
						getKeys().getTransactionKey()).toString();

		String encrypted = EncryptionUtil.encrypt(key, credentials);

		String props = new StringBuffer().append(
				this.getProperties().getProperties1()).append(";").append(
				this.getProperties().getProperties2()).append(";").append(
				this.getProperties().getProperties3()).toString();

		ConfigurationResponse vo = this.getConfigurations();
		MerchantConfiguration conf = null;
		if (vo != null) {
			conf = (MerchantConfiguration) vo
					.getConfiguration(PaymentConstants.PAYMENT_AUTHORIZENETNAME);
		}
		if (conf == null) {

			conf = new MerchantConfiguration();
			conf.setMerchantId(merchantid);

			conf.setConfigurationModule(moduleid);
			conf.setConfigurationKey(PaymentConstants.MODULE_PAYMENT_GATEWAY
					+ PaymentConstants.PAYMENT_AUTHORIZENETNAME);

		}

		conf.setConfigurationValue(encrypted);
		conf.setConfigurationValue1("");
		conf.setConfigurationValue2(props);

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

	public IntegrationKeys getKeys() {
		return keys;
	}

	public void setKeys(IntegrationKeys keys) {
		this.keys = keys;
	}

	public IntegrationProperties getProperties() {
		return properties;
	}

	public void setProperties(IntegrationProperties properties) {
		this.properties = properties;
	}

}



```
