# ShippingfedexexpressAction.java

## Review

## 1. Summary  

**Purpose**  
`ShippingfedexexpressAction` is a Struts‑style action that manages the configuration of the FedEx Express shipping module in a merchant‑centric e‑commerce platform. It allows a merchant to:

* View the current configuration (`displayModule`)  
* Prepare data for the UI (`prepareModule`)  
* Persist new or updated settings (`saveModule`)  
* Remove all FedEx Express settings (`deleteModule`)  

**Key Components**  

| Component | Role |
|-----------|------|
| `IntegrationKeys` | Holds FedEx API credentials (key, key2, userid, password). |
| `IntegrationProperties` | Holds additional FedEx properties (e.g., country code, packaging, etc.). |
| `InternationalServicesSelection` / `DomesticServicesSelection` | Lists of service codes selected by the merchant. |
| `PackageSelection` | The single package type chosen. |
| `ConfigurationResponse` | Contains all merchant‑specific module configurations fetched from the DB. |
| `ReferenceService` | Provides module‑specific configuration maps (services, packages) that are locale‑aware. |
| `MerchantService` | Persists or deletes module configurations for a merchant. |

**Design Patterns & Frameworks**  

* **Command/Action** – The class extends a generic `ShippingModuleAction`, fitting into a web‑framework that maps HTTP requests to action methods (`displayModule`, `prepareModule`, `saveModule`, `deleteModule`).  
* **DAO/Service Layer** – Interaction with `MerchantService` and `ReferenceService` keeps persistence logic separate from the action.  
* **Configuration Builder** – `ShippingUtil` contains static helpers to build key/value strings for storage and to parse configuration lines into maps.  
* **Validation** – Uses `ValidationException` and `addFieldError` (likely from Struts) to report missing fields.

---

## 2. Detailed Description  

### Execution Flow  

| Stage | What Happens | Dependencies |
|-------|--------------|--------------|
| **Preparation (`prepareModule`)** | 1. Retrieve merchant ID from session. <br>2. Load service configuration (international services) for the merchant’s country, falling back to generic `XX`. <br>3. Build a map of available international services and packages via `ShippingUtil`. <br>4. Load existing merchant configuration. | `ReferenceService`, `MerchantService`, `ShippingUtil` |
| **Display (`displayModule`)** | 1. If a configuration exists, extract keys, properties, package, and international services. <br>2. Populate the action’s fields so the JSP can render them. | `ConfigurationResponse`, `ShippingUtil` |
| **Save (`saveModule`)** | 1. Validate mandatory fields (keys, user‑id, password, meter, services). <br>2. If validation fails, throw `ValidationException`. <br>3. Build configuration lines for keys, properties, package, and services. <br>4. Assemble a list of configurations via `ShippingUtil.arrangeConfigurationsToSave`. <br>5. Persist them through `MerchantService`. | `MerchantService`, `ShippingUtil`, `StringUtil` |
| **Delete (`deleteModule`)** | Remove all configurations whose keys match the pattern `SHP_RT_*` for this module and merchant. | `MerchantService` |

### Assumptions & Constraints  

* The merchant is always authenticated and the `Context` is stored in the HTTP session.  
* All configuration keys follow a specific naming convention (`fedexexpress-keys`, `fedexexpress-properties`, `package-fedexexpress`, `service-intl-fedexexpress`).  
* At most three international services may be selected.  
* Domestic services are not currently persisted; the field exists only for completeness.  
* The code assumes that `ShippingUtil` correctly serializes and deserializes configuration lines.  

### Architecture & Design Choices  

* **Loose coupling** – The action uses service interfaces instead of directly accessing the DAO layer.  
* **Locale‑aware configuration** – `ReferenceService` fetches localized strings for services and packages.  
* **Generic key/value storage** – Configurations are stored as strings in the DB and parsed at runtime.  
* **Raw types** – The code still uses raw collections (`List`, `Map`) in several places, which reduces type safety and generates unchecked warnings.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `deleteModule()` | Removes all FedEx Express configurations for the current merchant. | None (uses session). | `void` | Deletes DB rows. |
| `displayModule()` | Loads current configuration into action fields for rendering. | None. | `void` | Populates `keys`, `properties`, `packageSelection`, `internationalServicesSelection`. |
| `prepareModule()` | Prepares locale‑specific service and package maps, and loads merchant configuration. | None. | `void` | Sets `internationalServicesMap`, `packageMap`, `configurations`. |
| `saveModule()` | Validates input, builds configuration lines, and persists them. | None (fields set via setters). | `void` | Throws `ValidationException` on error, otherwise updates DB. |
| `getConfigurations()` / `setConfigurations()` | Accessor for `configurations`. | None / `ConfigurationResponse`. | `ConfigurationResponse` / `void`. | None. |
| `getKeys()` / `setKeys()` | Accessor for FedEx API keys. | None / `IntegrationKeys`. | `IntegrationKeys` / `void`. | None. |
| `getPackageMap()` / `setPackageMap()` | Accessor for package options. | None / `Map<String,String>`. | `Map<String,String>` / `void`. | None. |
| `getPackageSelection()` / `setPackageSelection()` | Accessor for the chosen package code. | None / `String`. | `String` / `void`. | None. |
| `getDomesticServicesMap()` / `setDomesticServicesMap()` | Accessor for domestic service options (unused in current logic). | None / `Map<String,String>`. | `Map<String,String>` / `void`. | None. |
| `getInternationalServicesMap()` / `setInternationalServicesMap()` | Accessor for international service options. | None / `Map<String,String>`. | `Map<String,String>` / `void`. | None. |
| `getDomesticServicesSelection()` / `setDomesticServicesSelection()` | Accessor for selected domestic services (unused). | None / `List`. | `List` / `void`. | None. |
| `getInternationalServicesSelection()` / `setInternationalServicesSelection()` | Accessor for selected international services. | None / `List`. | `List` / `void`. | None. |
| `getProperties()` / `setProperties()` | Accessor for additional FedEx properties. | None / `IntegrationProperties`. | `IntegrationProperties` / `void`. | None. |

---

## 4. Dependencies  

| Library / Class | Type | Notes |
|-----------------|------|-------|
| `com.salesmanager.central.profile.Context` | Project | Holds merchant session data. |
| `com.salesmanager.central.profile.ProfileConstants` | Project | Provides session key constants. |
| `com.salesmanager.core.entity.reference.ModuleConfiguration` | Project | Holds static config lines. |
| `com.salesmanager.core.service.ServiceFactory` | Project | Service locator for DAO layers. |
| `com.salesmanager.core.service.common.model.IntegrationKeys` | Project | Credential container. |
| `com.salesmanager.core.service.common.model.IntegrationProperties` | Project | Property container. |
| `com.salesmanager.core.service.merchant.ConfigurationResponse` | Project | Encapsulates all merchant configs. |
| `com.salesmanager.core.service.merchant.MerchantService` | Project | Persistence for merchant configs. |
| `com.salesmanager.core.service.reference.ReferenceService` | Project | Provides static config maps. |
| `com.salesmanager.core.util.ShippingUtil` | Project | Helper for building/parsing config strings. |
| `com.salesmanager.core.util.StringUtil` | Project | Builds comma‑separated lines. |
| `org.apache.commons.lang.StringUtils` | Commons Lang | String utilities. |
| `org.apache.log4j.Logger` | Log4j | Logging (but logger class name mismatch). |
| `java.util.*` | Standard | Collections, Locale, Map, etc. |

All dependencies are project‑specific or well‑known third‑party libraries; none are platform‑specific.

---

## 5. Additional Notes  

### Strengths  

* **Separation of concerns** – The action delegates data access to services, keeping UI logic isolated.  
* **Locale support** – Service and package maps are built using the current locale.  
* **Centralized validation** – Uses Struts‑style `addFieldError` and throws `ValidationException` to halt processing.

### Weaknesses & Improvement Opportunities  

1. **Raw Types & Unchecked Warnings**  
   * Use generics everywhere (`List<String>`, `Map<String, String>`) to eliminate unchecked warnings and improve readability.  
2. **Logging Class Mismatch**  
   * Logger is instantiated with `ShippingupsxmlAction.class` – likely a copy‑paste error. It should reference `ShippingfedexexpressAction.class`.  
3. **Domestic Services Unused**  
   * The action contains fields for domestic services, but they are never loaded or persisted. Either remove them or implement similar logic as for international services.  
4. **Hardcoded Limits**  
   * The maximum of 3 international services is hardcoded; a constant or configuration value would make it easier to change.  
5. **NPE Risks**  
   * Methods like `getConfiguration("package-fedexexpress")` can return `null`. The code checks for blank but should also guard against `null` to avoid `NullPointerException` in `StringUtils.isBlank`.  
6. **Magic Strings**  
   * Configuration keys (`"fedexexpress-keys"`, `"fedexexpress-properties"`, etc.) are repeated. Centralizing them in constants reduces the chance of typos.  
7. **Missing Internationalization**  
   * Error messages are fetched via `getText`, but the action doesn’t expose a resource bundle for FedEx‑specific keys. Ensure the keys exist.  
8. **Testing & Coverage**  
   * Unit tests should exercise validation logic, configuration assembly, and interaction with `MerchantService`. Mocking `ReferenceService` will help isolate the action.  
9. **Thread Safety**  
   * The action is instantiated per request (typical in Struts), so thread safety isn’t an issue. However, if the framework re‑uses action objects, ensure that fields are cleared appropriately.  

### Future Enhancements  

* **Domestic Service Support** – Mirror the international logic for domestic services.  
* **UI Enhancements** – Provide a drop‑down for package selection and checkboxes for service selection with real‑time validation.  
* **Extensibility** – Abstract the FedEx module into a generic “ShippingModuleAction” that can be extended by other carriers (UPS, DHL, etc.).  
* **Configuration Versioning** – Store a version number for each configuration line to handle schema changes.  

Overall, the class fulfills its role but would benefit from modern Java best practices (generics, constants, improved logging) and a small amount of refactoring to eliminate unused fields and magic strings.

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
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.ValidationException;
import com.salesmanager.core.entity.reference.ModuleConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.ShippingUtil;
import com.salesmanager.core.util.StringUtil;

public class ShippingfedexexpressAction extends ShippingModuleAction {

	// user selections submited from the page
	private List domesticServicesSelection = null;
	private List internationalServicesSelection = null;
	private String packageSelection = null;
	private IntegrationKeys keys;
	private IntegrationProperties properties;

	private final static String moduleid = "fedexexpress";

	private ConfigurationResponse configurations;

	private Map<String, String> internationalServicesMap;// available services
															// options
	private Map<String, String> domesticServicesMap;// available services
													// options
	private Map<String, String> packageMap;// available packages options options

	private Logger log = Logger.getLogger(ShippingupsxmlAction.class);

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
					.getConfiguration("fedexexpress-keys");
			setKeys(keys);

			IntegrationProperties props = (IntegrationProperties) configurations
					.getConfiguration("fedexexpress-properties");
			setProperties(props);

			// choosen package [1 package allowed]
			String packageoption = (String) configurations
					.getConfiguration("package-fedexexpress");
			if (!StringUtils.isBlank(packageoption)) {
				setPackageSelection(packageoption);
			} else {// default value
				setPackageSelection("04");
			}

			Map selectedintlservices = (Map) configurations
					.getConfiguration("service-intl-fedexexpress");

			if (selectedintlservices != null) {
				Iterator i = selectedintlservices.keySet().iterator();
				List slist = new ArrayList();
				while (i.hasNext()) {
					String key = (String) i.next();
					slist.add(key);
				}
				setInternationalServicesSelection(slist);
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



		String country = "XX";

		ModuleConfiguration serviceconfig = null;


		// get intl services
		serviceconfig = rservice.getModuleConfiguration(moduleid, "service",
				country);

		if (serviceconfig == null) {
			serviceconfig = rservice.getModuleConfiguration(moduleid,
					"service", "XX");// generic
		}

		if (serviceconfig == null) {
			throw new Exception("ModuleConfiguration does not exist for "
					+ moduleid + "service XX");
		}

		String intlserviceline = serviceconfig.getConfigurationValue();

		Map intlservicemap = ShippingUtil.getConfigurationValuesMap(
				intlserviceline, moduleid, locale);
		// if(localservicemap!=null) {
		this.setInternationalServicesMap(intlservicemap);
		// }

		// }

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


		if (this.getInternationalServicesSelection() == null
				|| this.getInternationalServicesSelection().size() == 0
				|| this.getInternationalServicesSelection().size() > 3) {
			addFieldError("internationalServicesSelection",
					getText("label.shipping.chooseinternational"));
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


		String intlserviceline = null;

		if (this.getInternationalServicesSelection() != null
				&& this.getInternationalServicesSelection().size() > 0) {
			intlserviceline = StringUtil.buildMultipleValueLine(this
					.getInternationalServicesSelection());
		}

		String packageOption = getPackageSelection();


		List modulestosave = ShippingUtil.arrangeConfigurationsToSave(
				merchantid, configurations, moduleid, submitedcredentials,
				submitedproperties, packageOption, null, intlserviceline);

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

	public Map<String, String> getDomesticServicesMap() {
		return domesticServicesMap;
	}

	public void setDomesticServicesMap(Map<String, String> domesticServicesMap) {
		this.domesticServicesMap = domesticServicesMap;
	}

	public Map<String, String> getInternationalServicesMap() {
		return internationalServicesMap;
	}

	public void setInternationalServicesMap(
			Map<String, String> internationalServicesMap) {
		this.internationalServicesMap = internationalServicesMap;
	}

	public List getDomesticServicesSelection() {
		return domesticServicesSelection;
	}

	public void setDomesticServicesSelection(List domesticServicesSelection) {
		this.domesticServicesSelection = domesticServicesSelection;
	}

	public List getInternationalServicesSelection() {
		return internationalServicesSelection;
	}

	public void setInternationalServicesSelection(
			List internationalServicesSelection) {
		this.internationalServicesSelection = internationalServicesSelection;
	}

	public IntegrationProperties getProperties() {
		return properties;
	}

	public void setProperties(IntegrationProperties properties) {
		this.properties = properties;
	}

}



```
