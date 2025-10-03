# ShippingfedexAction.java

## Review

## 1. Summary  
`ShippingfedexAction` is a Struts‑style action class that manages the configuration of a FedEx shipping module for a merchant. The class handles four core stages of the configuration workflow:

| Stage | Responsibility |
|-------|----------------|
| **prepareModule** | Load available services and packages from the reference data store and populate UI‑ready maps. |
| **displayModule** | Populate the action’s fields with existing merchant configuration (keys, properties, selected services, selected package). |
| **saveModule** | Validate submitted data, build configuration strings, and persist the new or updated settings via `MerchantService`. |
| **deleteModule** | Remove all FedEx‑related configuration entries for a merchant. |

The code uses several helper utilities (`ShippingUtil`, `StringUtil`, `ValidationException`) and relies on a `ServiceFactory` singleton to obtain core services (`MerchantService`, `ReferenceService`). The action is tied to a fixed module id (`"fedex"`).

## 2. Detailed Description  
### Core Data Flow
1. **Preparation**  
   * `prepareModule()` pulls the locale, then queries the `ReferenceService` for the FedEx service configuration for the “XX” country (a hard‑coded fallback).  
   * It uses `ShippingUtil.getConfigurationValuesMap()` to convert the raw config string into a `Map<String,String>` of service codes → descriptions, and stores it in `internationalServicesMap`.  
   * It builds a generic package map and stores it in `packageMap`.  
   * Finally, it fetches the current merchant’s configuration via `MerchantService` and stores the resulting `ConfigurationResponse` in `configurations`.

2. **Display**  
   * `displayModule()` reads the `ConfigurationResponse` (if present).  
   * It extracts the keys (`fedex-keys`), properties (`fedex-properties`), selected package (`package-fedex`), and selected international services (`service-intl-fedex`) and populates the corresponding action fields.  
   * The selected services are converted from a `Map` of keys to a `List<String>` of service codes.

3. **Validation & Persistence**  
   * `saveModule()` validates the presence of keys (`key1`, `key2`, `userid`, `password`) and the presence of 1–3 selected international services.  
   * On failure, it registers field errors and throws a `ValidationException`.  
   * On success it constructs the credentials string (`ShippingUtil.buildShippingKeyLine(keys)`), the properties string, the selected services line (comma‑separated), and the package option.  
   * It calls `ShippingUtil.arrangeConfigurationsToSave()` to get a list of key/value pairs and hands that to `MerchantService.saveOrUpdateMerchantConfigurations()`.

4. **Deletion**  
   * `deleteModule()` deletes all configuration entries for the merchant that start with `"SHP_RT_"` and belong to the FedEx module.

### Design Choices & Assumptions
* The action follows a classic *action–service* pattern; business logic is off‑loaded to service factories and utilities.  
* The module id is hard‑coded; the code could instead inject a constant or read it from a properties file.  
* The code assumes the configuration values are stored as single strings that can be parsed into maps; the parsing logic is externalized to `ShippingUtil`.  
* Thread safety is not a concern because Struts actions are request‑scoped, but the use of static factories may cause issues in a highly concurrent environment.

## 3. Functions/Methods  
| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `deleteModule()` | Removes all FedEx‑related config entries for the current merchant. | None (uses session context). | Throws `Exception` on failure. |
| `displayModule()` | Populates the action’s fields from the merchant’s stored configuration. | None. | Sets keys, properties, package selection, selected services. |
| `prepareModule()` | Loads reference data (available services & packages) and fetches current merchant config. | None. | Populates `internationalServicesMap`, `packageMap`, and `configurations`. |
| `saveModule()` | Validates user input, builds configuration strings, and persists them. | None (uses action fields). | Persists data; may throw `ValidationException`. |
| Getters / Setters | Standard JavaBean accessors for the fields. | N/A | Return / set the corresponding private field. |

### Reusable Utility Methods (from other classes)
* `ShippingUtil.buildShippingKeyLine(IntegrationKeys)` – builds the FedEx credentials string.  
* `ShippingUtil.buildShippingPropertiesLine(IntegrationProperties)` – builds the properties string.  
* `ShippingUtil.buildMultipleValueLine(List)` – joins selected services into a comma‑separated string.  
* `ShippingUtil.arrangeConfigurationsToSave(...)` – converts all data into a list of key/value pairs for persistence.

## 4. Dependencies  
| Library / Framework | Type | Notes |
|---------------------|------|-------|
| `com.salesmanager.*` | Application services | Core business logic; `MerchantService`, `ReferenceService`, `ServiceFactory`. |
| `org.apache.commons.lang.StringUtils` | Third‑party | String utilities. |
| `org.apache.log4j.Logger` | Logging | Classic Log4j (deprecated in newer Java versions). |
| `java.util.*` | Standard | Collections, Locale, Map, etc. |
| `javax.servlet.*` | Servlet API | For `getServletRequest()` (inherited from parent). |

The code is tightly coupled to the `ServiceFactory` singleton and to the hard‑coded module id. No dependency injection framework is used.

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality Issues  
| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (`List`, `Map`) | No compile‑time type safety; risk of `ClassCastException`. | Replace with generics, e.g. `List<String>` and `Map<String,String>`. |
| **Hard‑coded constants** (`moduleid = "fedex"`, `"XX"`, logger class name) | Makes refactoring harder; potential bugs if module id changes. | Define a `public static final String MODULE_ID = "fedex";` and use it consistently; correct logger to `ShippingfedexAction.class`. |
| **Unnecessary commented code** | Obscures intent; may hide bugs. | Remove or move to version control notes. |
| **Potential `NullPointerException`** when casting `config.getConfiguration("service-intl-fedex")` to `Map` or `String` | If config is missing or of wrong type. | Check `instanceof` before casting; use defensive coding. |
| **Hard‑coded “XX” fallback** | Ignores locale; may load wrong service list. | Use the actual locale or provide a configurable fallback. |
| **Missing domestic services logic** | The action currently ignores domestic services completely. | Implement domestic service loading similar to international or remove unused fields. |
| **Logging not used** | Missing diagnostic information. | Add `log.info` / `log.error` around key operations and error paths. |
| **Exception handling** | Methods declare `throws Exception`; callers must handle generic exceptions. | Narrow to specific exceptions (`ValidationException`, `DataAccessException`) or wrap them. |
| **Thread‑safety of static factories** | Could cause problems in multi‑threaded environments. | Use dependency injection or thread‑safe singleton patterns. |

### 5.2 Security & Validation  
* Credentials are stored as plain strings; consider encrypting them or using a secure credential store.  
* The `saveModule()` method only checks for presence of fields; it does not validate format or length.  
* No CSRF protection is evident (though that may be handled by the framework).

### 5.3 Performance & Scalability  
* All reference data is loaded on every request (`prepareModule`). If the data is static, consider caching it in memory or a shared cache.  
* The `arrangeConfigurationsToSave()` method could be expensive if the number of config entries grows; verify its complexity.

### 5.4 Future Enhancements  
1. **Introduce Dependency Injection** (Spring, Guice) to inject `MerchantService`, `ReferenceService`, and `Logger`.  
2. **Centralize Configuration Retrieval** into a helper or repository class to avoid duplication.  
3. **Add Internationalization** for all hard‑coded strings.  
4. **Unit Test** all public methods, especially `saveModule()`’s validation logic.  
5. **Abstract Common Code** for all shipping modules (FedEx, UPS, etc.) into a base class or strategy pattern.  
6. **Add Audit Trail** – log who changed what and when.  

Overall, the class fulfills its functional requirements but would benefit significantly from modern Java best practices (generics, DI, proper logging, and clean code). Addressing the issues above will improve maintainability, safety, and extensibility.

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

public class ShippingfedexAction extends ShippingModuleAction {

	// user selections submited from the page
	private List domesticServicesSelection = null;
	private List internationalServicesSelection = null;
	private String packageSelection = null;
	private IntegrationKeys keys;
	private IntegrationProperties properties;

	private final static String moduleid = "fedex";

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
					.getConfiguration("fedex-keys");
			setKeys(keys);

			IntegrationProperties props = (IntegrationProperties) configurations
					.getConfiguration("fedex-properties");
			setProperties(props);

			// choosen package [1 package allowed]
			String packageoption = (String) configurations
					.getConfiguration("package-fedex");
			if (!StringUtils.isBlank(packageoption)) {
				setPackageSelection(packageoption);
			} else {// default value
				setPackageSelection("04");
			}

			Map selectedintlservices = (Map) configurations
					.getConfiguration("service-intl-fedex");

			if (selectedintlservices != null) {
				Iterator i = selectedintlservices.keySet().iterator();
				List slist = new ArrayList();
				while (i.hasNext()) {
					String key = (String) i.next();
					slist.add(key);
				}
				setInternationalServicesSelection(slist);
			}

			// selected services
			/*
			 * Map selectedintlservices =
			 * (Map)configurations.getConfiguration("service-intl-fedex");
			 * 
			 * if(selectedintlservices!=null) { Iterator i =
			 * selectedintlservices.keySet().iterator(); List slist = new
			 * ArrayList(); while(i.hasNext()) { String key = (String)i.next();
			 * slist.add(key); } setInternationalServicesSelection(slist); }
			 * 
			 * 
			 * Map selecteddomesticservices =
			 * (Map)configurations.getConfiguration("service-dom-fedex");
			 * 
			 * if(selecteddomesticservices!=null) { Iterator i =
			 * selecteddomesticservices.keySet().iterator(); List slist = new
			 * ArrayList(); while(i.hasNext()) { String key = (String)i.next();
			 * slist.add(key); } setDomesticServicesSelection(slist); }
			 */

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

		// String country = locale.getCountry();
		// if(locale.getVariant().equals("EUR")) {
		// country = "X1";
		// }

		String country = "XX";

		ModuleConfiguration serviceconfig = null;

		/*
		 * //get domestic services ModuleConfiguration serviceconfig =
		 * rservice.getModuleConfiguration(moduleid,"service-dom",country);
		 * 
		 * if(serviceconfig==null) { serviceconfig =
		 * rservice.getModuleConfiguration(moduleid,"service","XX");//generic }
		 * 
		 * if(serviceconfig==null) { throw new
		 * Exception("ModuleConfiguration does not exist for " + moduleid +
		 * "-service-dom-XX-" + locale.getCountry()); }
		 * 
		 * 
		 * String domesticserviceline = serviceconfig.getConfigurationValue();
		 * 
		 * 
		 * 
		 * 
		 * Map localservicemap =
		 * ShippingUtil.getConfigurationValuesMap(domesticserviceline
		 * ,moduleid,locale); if(localservicemap!=null) {
		 * this.setDomesticServicesMap(localservicemap); }
		 */

		// if user supports shipping to international
		// if(super.getShippingType().equals(ShippingConstants.INTERNATIONAL_SHIPPING))
		// {
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
