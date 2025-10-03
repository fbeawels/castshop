# PaymentpsigateAction.java

## Review

## 1. Summary  
**Purpose** – The `PaymentpsigateAction` class is a Struts‑style action that manages the configuration of a payment gateway module named *psigate* for a merchant in the SalesManager platform. It handles CRUD‑style interactions (prepare, display, save, delete) by reading and writing `MerchantConfiguration` objects through the `MerchantService`.  

**Key components**  
| Component | Role |
|-----------|------|
| `ConfigurationResponse` | Container that holds the module’s configuration objects (`IntegrationKeys`, `IntegrationProperties`, `MerchantConfiguration`) |
| `MerchantService` | Service layer used to persist and fetch merchant‑specific configurations |
| `EncryptionUtil` | Utility to generate a key and encrypt sensitive credential strings |
| `IntegrationKeys` / `IntegrationProperties` | Simple POJOs holding the keys and property values used by the psigate module |

**Design patterns & frameworks**  
* The class follows the **Command / Action** pattern typical of Struts 1/2 actions (`PaymentModuleAction` as base).  
* It uses **Dependency Injection** in a very light‑weight form: `ServiceFactory.getService(...)` to obtain a `MerchantService` instance.  
* Configuration objects are stored in a generic `ConfigurationResponse` map rather than a typed domain object, which is a form of the **Facade** pattern for configuration storage.

---

## 2. Detailed Description  

### Execution Flow  

1. **`prepareModule()`**  
   * Retrieves the merchant id from the session `Context`.  
   * Calls `MerchantService.getConfigurationByModule(moduleid, merchantid)` to fetch existing configuration for the psigate module and stores it in `configurations`.

2. **`displayModule()`**  
   * Reads the stored configuration from `configurations`.  
   * Extracts `IntegrationKeys` (`keys`) and `IntegrationProperties` (`properties`) and assigns them to the action’s fields so they can be rendered on the UI.

3. **`saveModule()`**  
   * Validates that `keys.userid` and `keys.transactionKey` are present; otherwise throws `ValidationException`.  
   * Generates an encryption key based on the merchant id, builds a credentials string (`userid;N;transactionKey`), encrypts it, and stores it as `configurationValue` in a `MerchantConfiguration` object.  
   * Concatenates the three property values into a single string and stores it in `configurationValue2`.  
   * Persists the configuration via `MerchantService.saveOrUpdateMerchantConfiguration(conf)`.

4. **`deleteModule()`** – currently empty (the delete logic is commented out).  

### Assumptions & Constraints  

| Assumption | Impact |
|------------|--------|
| `Context` is present in the HTTP session under `ProfileConstants.context`. | The action will throw a `NullPointerException` if the session is empty or the key is wrong. |
| The `ConfigurationResponse` map contains entries keyed by `"keys"` and `"properties"` when `displayModule()` is called. | If the map is missing those keys, the action silently ignores the update. |
| The credentials string is constructed with a hard‑coded `"N"` separator. | Any change to the gateway’s expected format requires code changes. |
| `EncryptionUtil.generatekey` accepts the merchant id as a string and returns a key suitable for `encrypt`. | No validation that the key is of the required length or format. |

### Architecture & Design Choices  

* **Single Responsibility** – The action class focuses solely on orchestrating configuration persistence; the heavy lifting is delegated to services and utilities.  
* **Loose Coupling** – Using a generic `ConfigurationResponse` container reduces tight coupling between the action and the underlying configuration model.  
* **Centralized Encryption** – `EncryptionUtil` abstracts the encryption algorithm, which keeps the action free from cryptographic details.  
* **No transaction management** – All persistence is delegated to `MerchantService` without explicit transaction boundaries, which may rely on the service layer for transactional support.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs / Side‑Effects | Notes |
|--------|---------|--------|------------------------|-------|
| `deleteModule()` | Intended to delete the psigate configuration for a merchant. | None (retrieves merchant id from session) | Persists removal via `MerchantService` (currently commented out). | Unimplemented – potential bug if called. |
| `displayModule()` | Prepares data for the UI by extracting keys and properties from `configurations`. | None | Sets `keys` and `properties` fields. | Silently ignores missing configuration. |
| `prepareModule()` | Loads the existing configuration for the module. | None | Stores result in `configurations`. | Depends on `MerchantService`. |
| `saveModule()` | Validates user input, encrypts credentials, builds properties string, persists the configuration. | None | Persists `MerchantConfiguration`. | Throws `ValidationException` if required fields are missing. |
| `getConfigurations()` | Getter for the `configurations` field. | None | Returns `ConfigurationResponse`. | |
| `setConfigurations(ConfigurationResponse)` | Setter for the `configurations` field. | `ConfigurationResponse` | Sets field. | |
| `getKeys()` | Getter for `IntegrationKeys`. | None | Returns `keys`. | |
| `setKeys(IntegrationKeys)` | Setter for `keys`. | `IntegrationKeys` | Sets field. | |
| `getProperties()` | Getter for `IntegrationProperties`. | None | Returns `properties`. | |
| `setProperties(IntegrationProperties)` | Setter for `properties`. | `IntegrationProperties` | Sets field. | |

### Reusable / Utility Methods  

* `EncryptionUtil.generatekey(String)` – used for key creation.  
* `EncryptionUtil.encrypt(String, String)` – used for credential encryption.  
* `StringUtils.isBlank(String)` – null/empty check.  

These utilities are used only within `saveModule()` but could be extracted to a helper class for clarity.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.service.ServiceFactory` | Third‑party (SalesManager core) | Provides static service lookup. |
| `com.salesmanager.core.service.merchant.MerchantService` | Third‑party | Service for CRUD of merchant configuration. |
| `com.salesmanager.core.service.common.model.IntegrationKeys` | Third‑party | Simple POJO for credentials. |
| `com.salesmanager.core.service.common.model.IntegrationProperties` | Third‑party | Simple POJO for gateway properties. |
| `com.salesmanager.core.util.EncryptionUtil` | Third‑party | Provides key generation & encryption. |
| `org.apache.commons.lang.StringUtils` | Third‑party (Apache Commons Lang) | Utility for string handling. |
| `com.salesmanager.core.constants.PaymentConstants` | Third‑party | Constants for payment module names. |
| `com.salesmanager.central.profile.Context` | Third‑party | Holds session data (merchant id). |
| `com.salesmanager.central.profile.ProfileConstants` | Third‑party | Holds session key constants. |
| `com.salesmanager.central.util.ValidationException` | Third‑party | Custom exception for validation errors. |
| Java EE (Servlet API) | Standard | For `getServletRequest()` and session handling. |

No platform‑specific APIs are used; the code is portable across servlet containers.

---

## 5. Additional Notes & Recommendations  

### Edge Cases & Potential Issues  

1. **Unimplemented Delete Logic** – Calling `deleteModule()` will currently do nothing; this may lead to orphaned configurations.  
2. **Missing Session Context** – If the `Context` object is not in the session, a `NullPointerException` will occur. A null‑check with a user‑friendly error message would be safer.  
3. **Hard‑coded “N” Separator** – The credentials string uses `"N"` as a flag. If the gateway changes the flag, the code must be modified. Consider using a configuration property instead.  
4. **Encryption Key Generation** – The method `EncryptionUtil.generatekey(String)` is opaque; if it changes (e.g., to use a different algorithm), the rest of the code may break. Document the expected key format.  
5. **Property Concatenation** – `properties` are concatenated into a single string without delimiters that escape semicolons. If any property contains a semicolon, parsing will break.  
6. **Null Handling for `ConfigurationResponse`** – The code assumes that `vo` and the inner configuration map will not be `null`. Defensive checks would prevent `NullPointerException`.  
7. **Thread‑Safety** – The action class fields (`configurations`, `keys`, `properties`) are instance fields. In a typical servlet container, a new instance is created per request, so this is safe; however, if the framework re‑uses instances, this could lead to data leakage between requests.  

### Suggested Enhancements  

| Area | Suggestion |
|------|------------|
| **Validation** | Move validation logic to a separate validator class or use a framework‑level validation (e.g., Struts 2 `Validator`). |
| **Configuration Storage** | Replace the concatenated string for `configurationValue2` with a JSON object or a dedicated map to improve readability and robustness. |
| **Encryption** | Expose a higher‑level `CredentialEncryptor` that hides the string format and provides `encryptCredentials`/`decryptCredentials` methods. |
| **Constants** | Externalize hard‑coded strings (`"N"`, semicolons) into `PaymentConstants` or a properties file. |
| **Error Handling** | Wrap service calls in try/catch blocks and translate low‑level exceptions into user‑friendly messages. |
| **Unit Tests** | Add tests for `saveModule()` that mock `MerchantService` and verify encryption, property handling, and persistence. |
| **Delete Logic** | Implement `deleteModule()` to call `MerchantService.deleteMerchantConfiguration(conf)` when `conf` is not null. |
| **Documentation** | Add Javadoc to each public method and a high‑level description of the credential format. |

### Future Extensions  

* **Multi‑merchant support** – If the platform needs to support shared or multi‑tenant configurations, the current approach (storing a single configuration per merchant) is adequate but may need to be extended.  
* **Audit Trail** – Log every configuration change with timestamp and user id for compliance.  
* **Plugin Architecture** – Expose the payment module as a pluggable component so other gateways can be integrated without modifying this action class.  

---

### Final Verdict  

The `PaymentpsigateAction` class is functional and follows a clear, service‑oriented approach to persisting payment configuration. However, it contains several hard‑coded values, lacks defensive coding, and has an unimplemented delete method. Addressing these concerns will improve maintainability, security, and robustness of the module.

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

public class PaymentpsigateAction extends PaymentModuleAction {

	private final static String moduleid = "psigate";

	private ConfigurationResponse configurations;

	private IntegrationProperties properties = new IntegrationProperties();
	private IntegrationKeys keys = new IntegrationKeys();

	@Override
	public void deleteModule() throws Exception {
		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

/*		ConfigurationResponse vo = this.getConfigurations();

		MerchantConfiguration conf = (MerchantConfiguration) vo
				.getConfiguration(PaymentConstants.PAYMENT_PSIGATENAME);

		if (conf != null) {
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			mservice.deleteMerchantConfiguration(conf);
		}*/

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
					getText("error.payment.storeid.required"));
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
					.getConfiguration(PaymentConstants.PAYMENT_PSIGATENAME);
		}
		if (conf == null) {

			conf = new MerchantConfiguration();
			conf.setMerchantId(merchantid);

			conf.setConfigurationModule(moduleid);
			conf.setConfigurationKey(PaymentConstants.MODULE_PAYMENT_GATEWAY
					+ PaymentConstants.PAYMENT_PSIGATENAME);

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
