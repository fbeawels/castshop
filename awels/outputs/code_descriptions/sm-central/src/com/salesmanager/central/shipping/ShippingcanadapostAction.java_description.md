# ShippingcanadapostAction.java

## Review

## 1. Summary  
`ShippingcanadapostAction` is a Struts‑style action that manages the configuration of the Canada Post shipping module for a merchant.  
- **Core responsibilities**  
  - **Prepare** the module (load available packages and existing merchant settings).  
  - **Display** the current settings to the UI.  
  - **Save** the submitted credentials, properties and package selection.  
  - **Delete** all Canada Post related settings.  
- **Key components**  
  - `MerchantService` – persistence layer for `MerchantConfiguration` entities.  
  - `ShippingUtil` – helper for building key/property strings and a package map.  
  - `ConfigurationResponse` – DTO that holds the merchant’s current settings.  
  - `IntegrationKeys` / `IntegrationProperties` – value objects that hold credentials and other integration settings.  
- **Design patterns / frameworks**  
  - Struts‑like MVC action (`ShippingModuleAction` is the base).  
  - DAO/service abstraction (`MerchantService`).  
  - Simple DTO/VO pattern for keys & properties.  
  - No heavy frameworks; mainly custom business logic.

---

## 2. Detailed Description  

### Flow of execution  

| Phase | Action | Key Objects | Notes |
|-------|--------|-------------|-------|
| **Prepare (`prepareModule`)** | • Load merchant id from `Context`.<br>• Build a locale‑aware package map via `ShippingUtil.buildPackageMap`.<br>• Retrieve existing configuration via `MerchantService.getConfigurationByModule`. | `Context`, `MerchantService`, `ConfigurationResponse` | Pre‑populate UI controls. |
| **Display (`displayModule`)** | • If a config exists, pull “canadapost‑keys”, “canadapost‑properties”, and the chosen package.<br>• Populate class fields (`keys`, `properties`, `packageSelection`). | `ConfigurationResponse` | Called after `prepareModule` to render the form. |
| **Save (`saveModule`)** | • Validate required fields (userid and package).<br>• Build credential/property strings.<br>• Either update existing `MerchantConfiguration` entries or create new ones.<br>• Persist via `MerchantService.saveOrUpdateMerchantConfigurations`. | `MerchantConfiguration`, `MerchantService` | Persists credentials, properties and selected package. |
| **Delete (`deleteModule`)** | • Clean all configuration entries matching key prefix `SHP_RT_`. | `MerchantService` | Removes all Canada Post settings for the merchant. |

### Assumptions & Constraints  

| Item | Description |
|------|-------------|
| **Thread safety** | The action is instantiated per request, so instance fields are request‑local. |
| **Null handling** | Most getters assume the retrieved configuration is non‑null; a `NullPointerException` may surface if a missing key is encountered. |
| **Date handling** | Uses legacy `java.util.Date`; a new `Date()` is redundant (`new Date(new Date().getTime())`). |
| **Error handling** | Validation errors are aggregated into a `ValidationException`. No granular field‑specific messaging beyond the hard‑coded strings. |
| **Persistence** | `MerchantService` abstracts DB operations but the code directly manipulates `MerchantConfiguration` values (1‑to‑1 mapping). |
| **Security** | Credentials are stored as a concatenated string via `ShippingUtil.buildShippingKeyLine`; no encryption or hashing is evident. |

### Architectural Observations  

- **Thin controller** – Most business logic resides here rather than in a dedicated service.  
- **Hard‑coded strings** – Magic values (e.g., `moduleid = "canadapost"`, key names) are sprinkled throughout.  
- **Raw types** – `List modulestosave = new ArrayList();` and `Map packages = ShippingUtil.buildPackageMap(...)` use non‑parameterized collections, losing type safety.  
- **Logging** – A `Logger` is declared but never used.  
- **Naming** – Class name uses camel‑case (`ShippingcanadapostAction`) rather than the conventional `ShippingCanadaPostAction`.  
- **Duplicated code** – The block that creates or updates `MerchantConfiguration` entries is duplicated in both the `if (configurations != null)` and `else` branches.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs / Side Effects | Comments |
|--------|---------|--------|------------------------|----------|
| `deleteModule()` | Deletes all Canada Post related configurations for the current merchant. | None (merchant id fetched from session) | Throws `Exception`; uses `MerchantService.cleanConfigurationLikeKeyModule`. | Simple one‑liner; could return a status. |
| `displayModule()` | Populates action fields from the existing configuration. | None | Sets `keys`, `properties`, `packageSelection`. | No validation; could throw if keys missing. |
| `prepareModule()` | Loads package options and current configuration. | None | Sets `packageMap`, `configurations`. | Uses `Locale` (from inherited `getLocale()`), may be null. |
| `saveModule()` | Validates input, builds configuration objects, persists them. | None (fields are populated by the framework) | Persists via `MerchantService`; throws `ValidationException` if required fields missing. | Large method; repeated code for create/update logic. |
| `getPackageMap()` / `setPackageMap(Map<String, String>)` | Getter/Setter for package options. | `Map<String, String>` | None. | No defensive copy. |
| `getConfigurations()` / `setConfigurations(ConfigurationResponse)` | Getter/Setter for current config. | `ConfigurationResponse` | None. | Same. |
| `getPackageSelection()` / `setPackageSelection(String)` | Getter/Setter for selected package. | `String` | None. | Same. |
| `getKeys()` / `setKeys(IntegrationKeys)` | Getter/Setter for integration keys. | `IntegrationKeys` | None. | Same. |
| `getProperties()` / `setProperties(IntegrationProperties)` | Getter/Setter for integration properties. | `IntegrationProperties` | None. | Same. |

---

## 4. Dependencies  

| External / Internal | Type | Notes |
|---------------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party (Apache Commons Lang) | For null/empty checks. |
| `org.apache.log4j.Logger` | Third‑party (Log4j 1.x) | Declared but unused. |
| `com.salesmanager.central.profile.*` | Internal | Context holder, profile constants. |
| `com.salesmanager.core.*` | Internal | Service factory, constants, entity (`MerchantConfiguration`), service (`MerchantService`), util (`ShippingUtil`), DTO (`ConfigurationResponse`). |
| `java.util.*` | Standard | Collections, Date, Locale. |
| `java.lang.*` | Standard | Basic classes. |

Platform‑specific: The code assumes a servlet environment (`getServletRequest()`), Struts‑like action lifecycle, and a session‑based context. No native OS dependencies.

---

## 5. Additional Notes  

### Edge Cases & Potential Issues  

1. **Null Pointer Exceptions** –  
   - `configurations.getConfiguration("canadapost‑keys")` will throw if the key is absent.  
   - `ShippingUtil.buildPackageMap(moduleid, locale)` may return `null` but the code still checks for null; fine.  
2. **Duplicate Configuration Entries** – If `saveModule` is called twice without intervening changes, duplicate entries might be created because the code only checks for null at the start, not for existing keys in the `configurations` map.  
3. **Hard‑coded Defaults** – Default package `"01"` is arbitrary; if the package list changes, this may break.  
4. **Security** – Credentials are stored as plain strings; consider encrypting them in the database.  
5. **Thread‑Safety of Logging** – `Logger` is never used; if added, ensure thread‑safe usage.  
6. **Legacy Date API** – Replace `new Date(new Date().getTime())` with `new Date()` or better `Instant.now()` from `java.time`.  

### Suggested Enhancements  

| Area | Recommendation |
|------|----------------|
| **Generics & Type Safety** | Replace raw types (`List modulestosave = new ArrayList();`, `Map packages = ...`) with parameterized types (`List<MerchantConfiguration>`, `Map<String, String>`). |
| **Code Reuse** | Extract configuration creation/update logic into a helper method (`buildMerchantConfiguration`). |
| **Validation** | Use a dedicated validation framework (e.g., Struts 2 validation or Bean Validation) instead of manual `addFieldError` checks. |
| **Error Reporting** | Return a structured status (e.g., success flag + message) instead of throwing a generic `ValidationException`. |
| **Security** | Store encrypted credentials; use a dedicated encryption service. |
| **Logging** | Log important steps (e.g., configuration load, save, delete) to aid troubleshooting. |
| **Naming** | Rename class to `ShippingCanadaPostAction` and `moduleid` to `MODULE_ID`. |
| **Use Java 8+** | Replace `Date` with `LocalDateTime` / `Instant`. Use streams for list construction. |
| **Configuration Constants** | Move hard‑coded strings (`"SHP_RT_"`, `ShippingConstants.MODULE_SHIPPING_RT_CRED`) to constants or enums. |
| **Null‑safe Access** | Use `Optional` or guard against missing keys. |
| **Unit Tests** | Write tests for each method, especially `saveModule` logic, using mock `MerchantService`. |

### Final Thoughts  

The class accomplishes its basic goal of persisting Canada Post shipping settings, but it does so with a mix of legacy patterns and manual error handling. Refactoring toward a cleaner service layer, type‑safe collections, and modern Java APIs would improve maintainability, testability, and security. The current implementation is functional but could benefit from the enhancements above to better meet modern development standards.

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
package com.salesmanager.central.shipping;

import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.ValidationException;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.ShippingUtil;

public class ShippingcanadapostAction extends ShippingModuleAction {

	private final static String moduleid = "canadapost";

	private Logger log = Logger.getLogger(ShippingcanadapostAction.class);

	private String packageSelection = null;
	private IntegrationKeys keys;
	private IntegrationProperties properties;

	private ConfigurationResponse configurations;
	private Map<String, String> packageMap;// available packages options options

	public void deleteModule() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		mservice.cleanConfigurationLikeKeyModule("SHP_RT_", moduleid,
				merchantid);

	}

	public void displayModule() throws Exception {

		if (configurations != null) {
			IntegrationKeys keys = (IntegrationKeys) configurations
					.getConfiguration("canadapost-keys");
			setKeys(keys);

			IntegrationProperties props = (IntegrationProperties) configurations
					.getConfiguration("canadapost-properties");
			setProperties(props);

			// choosen package [1 package allowed]
			String packageoption = (String) configurations
					.getConfiguration("package-canadapost");
			if (!StringUtils.isBlank(packageoption)) {
				setPackageSelection(packageoption);
			} else {// default selection
				setPackageSelection("01");
			}

		}

	}

	public void prepareModule() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		Locale locale = getLocale();

		Map packages = ShippingUtil.buildPackageMap(moduleid, locale);
		if (packages != null) {
			setPackageMap(packages);
		}

		// get merchant configs
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		ConfigurationResponse config = mservice.getConfigurationByModule(
				moduleid, merchantid);
		this.setConfigurations(config);

	}

	public void saveModule() throws Exception {

		boolean fielderror = false;

		if (StringUtils.isBlank(this.getKeys().getUserid())) {
			addFieldError("keys.userid", getText("errors.required.userid"));
			fielderror = true;
		}

		if (StringUtils.isBlank(this.getPackageSelection())) {
			addFieldError("packageSelection",
					getText("message.error.packageoption"));
			fielderror = true;
		}

		if (fielderror) {
			throw new ValidationException("Missing fields");
		}

		Date date = new Date(new Date().getTime());

		List modulestosave = new ArrayList();

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		String submitedcredentials = ShippingUtil.buildShippingKeyLine(this
				.getKeys());
		String submitedproperties = ShippingUtil
				.buildShippingPropertiesLine(this.getProperties());
		String packageOption = getPackageSelection();
		// first get the entry
		if (configurations != null) {
			// get credentials
			MerchantConfiguration credentials = configurations
					.getMerchantConfiguration(moduleid,
							ShippingConstants.MODULE_SHIPPING_RT_CRED);
			if (credentials != null) {
				credentials.setConfigurationValue1(submitedcredentials);
				credentials.setConfigurationValue2(submitedproperties);
			} else {
				credentials = new MerchantConfiguration();
				credentials
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_CRED);
				credentials.setConfigurationModule(moduleid);
				credentials.setDateAdded(date);
				credentials.setMerchantId(merchantid);
				credentials.setConfigurationValue1(submitedcredentials);
				credentials.setConfigurationValue2(submitedproperties);
			}
			credentials.setLastModified(date);
			modulestosave.add(credentials);

			// get packages
			MerchantConfiguration pack = configurations
					.getMerchantConfiguration(moduleid,
							ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
			if (pack != null) {
				pack.setConfigurationValue(getPackageSelection());
			} else {
				pack = new MerchantConfiguration();
				pack
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
				pack.setConfigurationModule(moduleid);
				pack.setDateAdded(date);
				pack.setConfigurationValue(packageOption);
				pack.setMerchantId(merchantid);
			}
			pack.setLastModified(date);
			modulestosave.add(pack);

		} else {// create both entries
			MerchantConfiguration credentials = new MerchantConfiguration();
			credentials
					.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_CRED);
			credentials.setConfigurationModule(moduleid);
			credentials.setDateAdded(date);
			credentials.setMerchantId(merchantid);
			credentials.setLastModified(date);
			credentials.setConfigurationValue1(submitedcredentials);
			credentials.setConfigurationValue2(submitedproperties);
			modulestosave.add(credentials);

			MerchantConfiguration pack = new MerchantConfiguration();
			pack
					.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
			pack.setConfigurationModule(moduleid);
			pack.setDateAdded(date);
			pack.setLastModified(date);
			pack.setMerchantId(merchantid);
			pack.setConfigurationValue(packageOption);
			modulestosave.add(pack);
		}

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		mservice.saveOrUpdateMerchantConfigurations(modulestosave);

	}

	public Map<String, String> getPackageMap() {
		return packageMap;
	}

	public void setPackageMap(Map<String, String> packageMap) {
		this.packageMap = packageMap;
	}

	public ConfigurationResponse getConfigurations() {
		return configurations;
	}

	public void setConfigurations(ConfigurationResponse configurations) {
		this.configurations = configurations;
	}

	public String getPackageSelection() {
		return packageSelection;
	}

	public void setPackageSelection(String packageSelection) {
		this.packageSelection = packageSelection;
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
