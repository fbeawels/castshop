# PaymentmonerisAction.java

## Review

## 1. Summary
`PaymentmonerisAction` is a Struts‑style action that handles the lifecycle of a **Moneris payment gateway** configuration for a merchant.  
* **Purpose** – Create, update, display and (intended) delete the Moneris configuration stored in the database.  
* **Key components**  
  * `moduleid` – identifies the gateway module.  
  * `configurations` – the current configuration set for the merchant.  
  * `properties` / `keys` – helper objects that hold the raw data entered by the user.  
  * `MerchantService` – the DAO layer that persists `MerchantConfiguration` objects.  
* **Design pattern** – MVC (Model‑View‑Controller) with an action controller; uses dependency injection via a factory (`ServiceFactory`).  
* **Libraries** – Apache Commons Lang (`StringUtils`), custom Salesmanager core/util classes, and an `EncryptionUtil` for simple credential protection.

## 2. Detailed Description
### Execution Flow
| Phase | Method | What it does |
|-------|--------|--------------|
| **Preparation** | `prepareModule()` | Pulls the merchant id from the session, retrieves the current configuration for the Moneris module, and stores it in `configurations`. |
| **Display** | `displayModule()` | Reads keys/properties from `configurations` and exposes them to the view layer via getters/setters. |
| **Save** | `saveModule()` | Validates required key fields, encrypts the credential string, builds a `MerchantConfiguration` (creating one if necessary), and persists it. |
| **Delete** | `deleteModule()` | Intended to delete the configuration, but the deletion logic is commented out. |

### Assumptions & Constraints
* The session must contain a `Context` bean that exposes the merchant id.  
* The `MerchantService` is thread‑safe and can be shared across actions.  
* Credentials are stored in a single configuration value, encrypted with a key derived from the merchant id.  
* The action is tightly coupled to the Salesmanager framework – it uses `ServiceFactory`, `PaymentConstants`, and `ProfileConstants`.

### Architecture Choices
* **Separation of Concerns** – The action only deals with request handling; all business logic is delegated to `MerchantService`.  
* **Encapsulation of Raw Data** – `IntegrationKeys` and `IntegrationProperties` keep the raw form data separate from the persistence model.  
* **Encryption** – Simple encryption is used instead of storing plain text, but the key management strategy (derived from merchant id) is minimalistic.

## 3. Functions/Methods
| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `deleteModule()` | Intended to remove the Moneris configuration. | None | None | (currently none – code is commented out) |
| `displayModule()` | Exposes current configuration values to the view. | None | Sets `keys` and `properties` fields | None |
| `prepareModule()` | Loads current config for the merchant. | None | Sets `configurations` field | None |
| `saveModule()` | Validates, encrypts, and persists configuration. | None | Throws `ValidationException` on error | Calls `MerchantService.saveOrUpdateMerchantConfiguration` |
| `getConfigurations()` | Getter for `configurations`. | None | `ConfigurationResponse` | None |
| `setConfigurations(ConfigurationResponse)` | Setter for `configurations`. | `ConfigurationResponse` | None | None |
| `getKeys()` | Getter for `keys`. | None | `IntegrationKeys` | None |
| `setKeys(IntegrationKeys)` | Setter for `keys`. | `IntegrationKeys` | None | None |
| `getProperties()` | Getter for `properties`. | None | `IntegrationProperties` | None |
| `setProperties(IntegrationProperties)` | Setter for `properties`. | `IntegrationProperties` | None | None |

### Reusable / Utility Methods
* `EncryptionUtil.generatekey()` and `EncryptionUtil.encrypt()` – used only within `saveModule()`; could be extracted into a dedicated helper class for clarity.

## 4. Dependencies
| Library | Type | Notes |
|---------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String validation. |
| `com.salesmanager.central.profile.*` | Internal | Session context handling. |
| `com.salesmanager.central.util.ValidationException` | Internal | Validation exception type. |
| `com.salesmanager.core.constants.PaymentConstants` | Internal | Constant keys for payment modules. |
| `com.salesmanager.core.entity.merchant.MerchantConfiguration` | Internal | Persistence entity. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Factory for service lookup. |
| `com.salesmanager.core.service.common.model.IntegrationKeys/Properties` | Internal | DTOs for form data. |
| `com.salesmanager.core.service.merchant.ConfigurationResponse` | Internal | Wrapper for configuration sets. |
| `com.salesmanager.core.service.merchant.MerchantService` | Internal | CRUD operations on merchant data. |
| `com.salesmanager.core.util.EncryptionUtil` | Internal | Simple encryption. |

All dependencies are internal to the Salesmanager platform except Apache Commons Lang, which is widely used and stable.

## 5. Additional Notes
### Edge Cases / Missing Handling
* **Null Configurations** – `displayModule()` and `saveModule()` assume `configurations` is non‑null. If `prepareModule()` fails or is omitted, a `NullPointerException` will be thrown. A defensive null check or a default empty `ConfigurationResponse` should be added.  
* **Delete Logic** – The method is effectively a no‑op; the commented code should either be removed or implemented to actually delete the configuration.  
* **Property Validation** – Only `userid` and `transactionKey` are validated. If properties are required, additional checks should be added.  
* **Encryption Key Reuse** – The encryption key is derived from the merchant id and may change if the id changes, causing data loss. A dedicated key‑management strategy would be safer.  
* **Error Messaging** – The error messages rely on `getText()` which assumes i18n resources are present. Ensure that keys `error.payment.storeid.required` and `error.payment.transactionkey.required` exist.  

### Potential Enhancements
1. **Refactor Encryption** – Extract credential handling into a separate service (`CredentialEncryptor`) to simplify the action and facilitate unit testing.  
2. **Unit Tests** – Add tests for validation, encryption, and DAO interactions using a mock `MerchantService`.  
3. **Logging** – Introduce a logger (e.g., SLF4J) to trace actions, especially on exceptions and encryption failures.  
4. **Graceful Deletion** – Implement `deleteModule()` to remove the configuration from the database and clean up related session data.  
5. **Form Binding** – Use a dedicated form object that encapsulates `IntegrationKeys` and `IntegrationProperties` to simplify data transfer between view and controller.  
6. **Error Handling** – Replace the generic `throws Exception` with specific checked exceptions and provide user‑friendly messages.  

Overall, the code performs its core responsibilities but would benefit from defensive programming, clearer separation of concerns, and a complete implementation of the delete functionality.

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

public class PaymentmonerisAction extends PaymentModuleAction {

	private final static String moduleid = "moneris";

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
				.getConfiguration(PaymentConstants.PAYMENT_MONERIS);

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
					.getConfiguration(PaymentConstants.PAYMENT_MONERIS);
		}
		if (conf == null) {

			conf = new MerchantConfiguration();
			conf.setMerchantId(merchantid);

			conf.setConfigurationModule(moduleid);
			conf.setConfigurationKey(PaymentConstants.MODULE_PAYMENT_GATEWAY
					+ PaymentConstants.PAYMENT_MONERIS);

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
