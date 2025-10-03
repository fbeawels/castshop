# ShippingfedexgroundAction.java

## Review

## 1. Summary  
**Purpose** – The `ShippingfedexgroundAction` class implements the CRUD logic for a “FedEx Ground” shipping module within the SalesManager web application. It exposes the following key operations:

| Operation | Responsibility |
|-----------|----------------|
| `prepareModule()` | Loads locale‑specific package definitions and merchant configuration data. |
| `displayModule()` | Populates the UI with any previously stored keys, properties and selected package. |
| `saveModule()` | Validates input, constructs configuration key/value lines, and persists them via `MerchantService`. |
| `deleteModule()` | Removes all shipping‑related configuration entries for the current merchant. |

**Key Components**  
* `MerchantService` – Handles persistence of configuration records.  
* `ReferenceService` – Currently unused (possibly for future reference data).  
* `ShippingUtil` – Provides helper methods to format configuration strings and build maps.  
* `IntegrationKeys` / `IntegrationProperties` – Value objects representing the FedEx credentials and optional properties.

**Design Patterns / Frameworks**  
* The class follows a typical *Action* pattern used in MVC frameworks (e.g., Struts).  
* Service Locator pattern (`ServiceFactory.getService(...)`) supplies business services.  
* Validation is performed by throwing a custom `ValidationException`, presumably caught higher‑up in the framework.

---

## 2. Detailed Description  
### Execution Flow

1. **Prepare Phase (`prepareModule`)**  
   * Obtains the current merchant ID from the HTTP session.  
   * Builds a package map (`moduleid`, locale) via `ShippingUtil.buildPackageMap`.  
   * Retrieves the merchant’s existing configuration (`ConfigurationResponse`) and stores it.

2. **Display Phase (`displayModule`)**  
   * If a configuration exists, extracts the keys (`fedexground-keys`) and properties (`fedexground-properties`).  
   * Reads the selected package (`package-fedexground`) and defaults to `"04"` when absent.

3. **Save Phase (`saveModule`)**  
   * Performs field‑level validation for required credentials.  
   * Serialises the keys and properties into single configuration strings.  
   * Calls `ShippingUtil.arrangeConfigurationsToSave` to create a list of `Configuration` objects to be persisted.  
   * Persists via `MerchantService.saveOrUpdateMerchantConfigurations`.

4. **Delete Phase (`deleteModule`)**  
   * Calls `MerchantService.cleanConfigurationLikeKeyModule` to delete all keys that start with `SHP_RT_` for this module and merchant.

### Assumptions & Constraints
* **Session state** – Assumes a `ProfileConstants.context` attribute exists in the HTTP session.  
* **Locale handling** – Special‑casing for the `"EUR"` variant sets the country code to `"X1"`.  
* **Configuration format** – Relies on the specific key names (`fedexground-keys`, `fedexground-properties`, etc.).  
* **Null‑safety** – Raw types (`List`, `Map`) are used; the code does not guard against potential `NullPointerException` when iterating over returned maps.  
* **Thread‑safety** – The action object is likely instantiated per request, so field sharing between threads is not a problem.

### Architecture Choices
* The action is tightly coupled to `ShippingUtil`, which performs all heavy lifting for formatting and parsing.  
* The use of a Service Locator (`ServiceFactory`) simplifies dependency injection but makes unit testing harder (requires mocking static calls).  
* The class exposes many getters/setters, enabling the framework to bind form fields automatically.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters / Inputs | Return / Side‑Effects | Comments |
|--------|---------|---------------------|-----------------------|----------|
| `deleteModule()` | Removes all shipping configuration entries for the current merchant. | None | None (throws `Exception` on failure) | Uses `MerchantService.cleanConfigurationLikeKeyModule` to delete keys. |
| `displayModule()` | Populates internal fields from the persisted configuration for rendering. | None | None | Sets `keys`, `properties`, `packageSelection`. |
| `prepareModule()` | Loads reference data and current configuration. | None | None | Populates `packageMap`, `configurations`. |
| `saveModule()` | Validates user input and persists configuration. | None | None (throws `ValidationException` on missing fields) | Builds configuration strings via `ShippingUtil` and persists via `MerchantService`. |
| `getConfigurations()` | Getter for `configurations`. | None | `ConfigurationResponse` | – |
| `setConfigurations(...)` | Setter for `configurations`. | `ConfigurationResponse` | None | – |
| `getKeys()` / `setKeys(...)` | Accessor for FedEx credentials. | `IntegrationKeys` | – | – |
| `getPackageMap()` / `setPackageMap(...)` | Accessor for package options map. | `Map<String,String>` | – | – |
| `getPackageSelection()` / `setPackageSelection(...)` | Accessor for currently selected package. | `String` | – | – |
| `getGlobalServicesMap()` / `setGlobalServicesMap(...)` | Accessor for global services map. | `Map<String,String>` | – | – |
| `getGlobalServicesSelection()` / `setGlobalServicesSelection(...)` | Accessor for selected services. | `List` | – | – |
| `getProperties()` / `setProperties(...)` | Accessor for shipping properties. | `IntegrationProperties` | – | – |

**Reusable / Utility Methods**  
* The majority of heavy lifting is delegated to `ShippingUtil`.  
* Validation logic is repeated only in `saveModule()`.  

---

## 4. Dependencies  

| External / Internal | Nature | Notes |
|---------------------|--------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Used for null‑safe string checks. |
| `com.salesmanager.central.profile.*` | Internal | Holds session context. |
| `com.salesmanager.central.util.ValidationException` | Internal | Custom validation exception. |
| `com.salesmanager.core.service.*` | Internal | Service locator for business logic. |
| `com.salesmanager.core.service.common.model.*` | Internal | Domain objects for keys/properties. |
| `com.salesmanager.core.service.merchant.*` | Internal | Handles configuration persistence. |
| `com.salesmanager.core.service.reference.*` | Internal | Reference data provider (unused). |
| `com.salesmanager.core.util.ShippingUtil` | Internal | Static helper methods for shipping configuration. |
| `java.util.*` | Standard | Collections and locale. |

No external network or platform‑specific APIs are invoked; all interactions are via the internal service layer.

---

## 5. Additional Notes  

### Strengths
* **Clear separation of concerns** – Business logic is encapsulated in `ShippingUtil` and services; the action only orchestrates data flow.  
* **Use of constants** – Module ID (`moduleid`) is defined once, reducing duplication.  
* **Internationalization** – Error messages are fetched via `getText()`, enabling multi‑language support.  

### Weaknesses & Edge Cases  
1. **Raw types** – `List` and `Map` are used without generics, which can lead to `ClassCastException` at runtime and hinders readability.  
2. **Null‑safety** – Methods such as `prepareModule()` assume `packages` and `config` are non‑null, yet the code does not handle `null` gracefully in all cases.  
3. **Duplicate code** – The same `StringUtils.isBlank` checks appear multiple times; a helper method could reduce repetition.  
4. **Hardcoded defaults** – The default package ID `"04"` is a magic string; it should be defined as a constant with a descriptive name.  
5. **Internationalization fallback** – The logic that changes `country` to `"X1"` for the `"EUR"` variant is fragile; a better approach would be to use a locale‑specific lookup table.  
6. **Service Locator** – Using `ServiceFactory.getService()` impedes testability; dependency injection (e.g., constructor injection) would allow mocking of services.  
7. **Exception handling** – All methods declare `throws Exception`, which can obscure the specific failure points. Narrower exceptions would aid debugging.  
8. **Logging** – No logging is present; adding debug/info/error logs would help trace execution and diagnose issues.  

### Future Enhancements  
* **Type safety** – Replace raw collections with parameterised types (`List<...>`, `Map<String, String>`).  
* **Dependency injection** – Refactor to accept services via constructor or setter injection.  
* **Centralised validation** – Move field‑level validation into a reusable validator component.  
* **Constants for defaults** – Define `DEFAULT_PACKAGE_ID` and any other magic values.  
* **Unit tests** – With DI in place, write comprehensive tests for each action method, mocking `MerchantService` and `ShippingUtil`.  
* **Internationalization of configuration keys** – Use locale‑specific key prefixes instead of hardcoded module ID strings.  
* **Logging** – Integrate a logging framework (SLF4J/Logback) to record key events and errors.  

Overall, the class fulfills its intended purpose but would benefit from modern Java practices (generics, DI, logging) and a cleaner separation of validation and formatting logic.

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

import java.util.List;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.ValidationException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.ShippingUtil;

public class ShippingfedexgroundAction extends ShippingModuleAction {

	// user selections submited from the page
	private List globalServicesSelection = null;
	private String packageSelection = null;
	private IntegrationKeys keys;
	private IntegrationProperties properties;

	private final static String moduleid = "fedexground";

	private ConfigurationResponse configurations;

	private Map<String, String> globalServicesMap;
	private Map<String, String> packageMap;// available packages options options

	@Override
	public void deleteModule() throws Exception {
		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		mservice.cleanConfigurationLikeKeyModule("SHP_RT_", moduleid,
				merchantid);

	}

	@Override
	public void displayModule() throws Exception {
		if (configurations != null) {
			IntegrationKeys keys = (IntegrationKeys) configurations
					.getConfiguration("fedexground-keys");
			setKeys(keys);

			IntegrationProperties props = (IntegrationProperties) configurations
					.getConfiguration("fedexground-properties");
			setProperties(props);

			// choosen package [1 package allowed]
			String packageoption = (String) configurations
					.getConfiguration("package-fedexground");
			if (!StringUtils.isBlank(packageoption)) {
				setPackageSelection(packageoption);
			} else {// default value
				setPackageSelection("04");
			}



		}

	}

	@Override
	public void prepareModule() throws Exception {
		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);

		Locale locale = getLocale();

		String country = locale.getCountry();
		if (locale.getVariant().equals("EUR")) {
			country = "X1";
		}



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

	@Override
	public void saveModule() throws Exception {
		boolean fielderror = false;
		if (this.getKeys() == null
				|| StringUtils.isBlank(this.getKeys().getKey1())) {
			addFieldError("keys.key1", getText("errors.required.fedexkey"));
			fielderror = true;
		}
		if (StringUtils.isBlank(this.getKeys().getUserid())) {
			addFieldError("keys.userid", getText("errors.required.userid"));
			fielderror = true;
		}
		if (StringUtils.isBlank(this.getKeys().getPassword())) {
			addFieldError("keys.password",
					getText("errors.required.fedexpassword"));
			fielderror = true;
		}
		if (StringUtils.isBlank(this.getKeys().getKey2())) {
			addFieldError("keys.key2", getText("errors.required.fedexmeter"));
			fielderror = true;
		}



		if (fielderror) {
			throw new ValidationException("Missing fields");
		}

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		String submitedcredentials = ShippingUtil.buildShippingKeyLine(keys);

		String submitedproperties = ShippingUtil
				.buildShippingPropertiesLine(this.getProperties());



		String packageOption = getPackageSelection();

		List modulestosave = ShippingUtil.arrangeConfigurationsToSave(
				merchantid, configurations, moduleid, submitedcredentials,
				submitedproperties, packageOption, null, null);

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		mservice.saveOrUpdateMerchantConfigurations(modulestosave);

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

	public Map<String, String> getPackageMap() {
		return packageMap;
	}

	public void setPackageMap(Map<String, String> packageMap) {
		this.packageMap = packageMap;
	}

	public String getPackageSelection() {
		return packageSelection;
	}

	public void setPackageSelection(String packageSelection) {
		this.packageSelection = packageSelection;
	}

	public Map<String, String> getGlobalServicesMap() {
		return globalServicesMap;
	}

	public void setGlobalServicesMap(Map<String, String> globalServicesMap) {
		this.globalServicesMap = globalServicesMap;
	}

	public List getGlobalServicesSelection() {
		return globalServicesSelection;
	}

	public void setGlobalServicesSelection(List globalServicesSelection) {
		this.globalServicesSelection = globalServicesSelection;
	}

	public IntegrationProperties getProperties() {
		return properties;
	}

	public void setProperties(IntegrationProperties properties) {
		this.properties = properties;
	}

}



```
