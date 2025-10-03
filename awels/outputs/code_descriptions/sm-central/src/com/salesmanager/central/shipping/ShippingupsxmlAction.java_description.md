# ShippingupsxmlAction.java

## Review

## 1. Summary  

**Purpose** – `ShippingupsxmlAction` is a Struts‑style action that manages the configuration of the UPS XML shipping module for a merchant. It:

1. Loads module configuration (`prepareModule`).
2. Populates form fields for the UI (`displayModule`).
3. Validates and persists the user’s selections (`saveModule`).
4. Removes the module configuration (`deleteModule`).

**Key components**

| Component | Role |
|-----------|------|
| `globalServicesSelection` / `packageSelection` | Form fields populated by the UI |
| `IntegrationKeys` / `IntegrationProperties` | Hold the UPS credentials & settings |
| `ShippingUtil` | Builds and parses the string representation of keys & properties |
| `MerchantService` | Persists and retrieves configuration values |
| `ReferenceService` | Provides language‑specific maps of available services & packages |
| `ShippingConstants` | Constant identifiers used when persisting configuration |
| `Logger` | Intended for diagnostic output (unused in the current code) |

The class extends `ShippingModuleAction`, presumably a base action providing request/session handling and some utility methods (e.g., `addFieldError`, `getText`). No sophisticated design patterns are visible; the implementation is a straight‑forward procedural flow wrapped in a servlet action.

## 2. Detailed Description  

### 2.1 Core flow

| Phase | Method | Action |
|-------|--------|--------|
| **Initialisation** | `prepareModule()` | *Retrieve merchant context*, lookup module‑specific configuration, load available services and packages, then pull the merchant’s existing config. |
| **Display** | `displayModule()` | Read stored config and populate the action’s fields (credentials, properties, package, selected services). |
| **Persist** | `saveModule()` | Validate fields; build credential/property strings; update or create two `MerchantConfiguration` records (credentials and package+services). |
| **Delete** | `deleteModule()` | Remove all config entries that start with `SHP_RT_` for this module. |

The lifecycle is typical for a CRUD form: *prepare → display → submit → delete*.

### 2.2 Assumptions & Constraints  

* The action relies on a **Session‑based Context** (`ProfileConstants.context`) that holds the merchant ID.
* The module only supports a single package selection but up to three global services.
* Credentials and properties are stored as a concatenated string; the actual parsing logic is hidden in `ShippingUtil`.
* All persistence goes through `MerchantService`, which is assumed to be thread‑safe and handle transactions.
* Logging infrastructure is present but not used; any errors are surfaced via `ValidationException`.

### 2.3 Architecture & Design Choices  

* **Action‑centric** – Each CRUD operation is a separate method on the action, typical of Struts 1/2 style frameworks.
* **Direct database manipulation** – Instead of a higher‑level DTO, the code manipulates `MerchantConfiguration` entities directly.
* **String‑based config storage** – Keys and properties are serialized into a single string; this reduces schema complexity but sacrifices type safety.
* **Raw types** – The code uses raw `List` and `Map` without generics, which makes the code prone to `ClassCastException` and hinders readability.

## 3. Functions/Methods  

| Method | Purpose | Parameters / Inputs | Returns / Effects | Notes |
|--------|---------|---------------------|-------------------|-------|
| `prepareModule()` | Loads available services/packages and the merchant’s current config. | None | Sets `globalServicesMap`, `packageMap`, `configurations`. | Throws generic `Exception`. |
| `displayModule()` | Populates form fields from stored configuration. | None | Sets `keys`, `properties`, `packageSelection`, `globalServicesSelection`. | Uses hard‑coded config keys. |
| `saveModule()` | Validates input and persists or updates the two config entries. | None | Persists via `MerchantService`. | Throws `ValidationException` if any field is missing. |
| `deleteModule()` | Removes all config entries for this module. | None | Calls `cleanConfigurationLikeKeyModule`. | No confirmation or error handling. |
| Getter/Setter pairs for all fields | Standard JavaBeans accessors. | — | — | Not type‑safe (raw types). |
| **Utility methods** (none defined) – the class relies on external utilities (`ShippingUtil`, `StringUtil`). | | | | |

### Reusable / Utility Methods  

* **`ShippingUtil.buildShippingKeyLine(keys)`** – Serialises `IntegrationKeys`.
* **`ShippingUtil.buildShippingPropertiesLine(props)`** – Serialises `IntegrationProperties`.
* **`StringUtil.buildMultipleValueLine(list)`** – Joins a list into a delimited string.

These helpers are used only in `saveModule()`.

## 4. Dependencies  

| Library / Framework | Role | Standard / 3rd‑Party | Notes |
|---------------------|------|----------------------|-------|
| `org.apache.commons.lang.StringUtils` | Null/blank checks | 3rd‑party | Commonly used, but not fully leveraged (missing null‑guard for `keys`). |
| `org.apache.log4j.Logger` | Logging | 3rd‑party | Declared but never used. |
| `com.salesmanager.core.service.*` | Persistence & reference services | 3rd‑party | Part of the application’s core services layer. |
| `com.salesmanager.central.profile.*` | Session context | 3rd‑party | Provides `Context` with merchant ID. |
| `com.salesmanager.central.util.*` | Validation exception, string utils | 3rd‑party | `ValidationException` used to surface form errors. |
| `com.salesmanager.core.util.ShippingUtil` | Builds config strings | 3rd‑party | Encapsulates UPS‑specific serialization logic. |
| `javax.servlet.*` (implied) | Servlet request handling | Standard | Provided by the web container. |

No platform‑specific APIs; all dependencies are typical enterprise Java libraries.

## 5. Additional Notes  

### 5.1 Edge Cases & Potential Failures  

1. **Null‑Pointer Risks**  
   * `this.getKeys()` could be null; `getKey1()` then throws NPE.  
   * `getGlobalServicesSelection()` may be null and is cast to a `List` without generics.  

2. **Hard‑coded String Keys**  
   * The module relies on exact string literals for config keys (`"upsxml-keys"`, `"package-upsxml"`, etc.). A typo would silently fail.  

3. **Redundant Date Creation**  
   * `new Date(new Date().getTime())` is equivalent to `new Date()`.  

4. **No Transaction / Error Handling**  
   * `MerchantService.saveOrUpdateMerchantConfigurations` is called without try/catch; any persistence exception bubbles up as a generic `Exception`.  

5. **Security**  
   * Credentials are stored as plain strings. There is no evidence of encryption or secure hashing.  

6. **Logging**  
   * The `log` instance is never used; any unexpected error will not be recorded.  

### 5.2 Code‑Style / Maintainability  

* **Generics** – All collections are raw; modern Java should use `List<String>` / `Map<String, String>`.  
* **Method Duplication** – Building of credential strings in `saveModule` could be extracted to a helper.  
* **Validation** – The commented‑out `@Validations` annotation suggests a plan to use declarative validation; the manual checks should be moved to a validator class.  
* **Readability** – Long blocks of code (e.g., the two large `if` blocks in `saveModule`) could be split into private helper methods (`buildCredentialsConfig`, `buildPackageConfig`).  

### 5.3 Suggested Enhancements  

1. **Use Generics** throughout the code to eliminate unchecked casts.  
2. **Implement proper null‑safety** for `keys` and other critical objects.  
3. **Move validation to a separate validator** (e.g., `ShippingupsxmlActionValidator`) and enable the Struts validation framework.  
4. **Add logging** in every major step to aid debugging.  
5. **Encrypt credentials** before persisting.  
6. **Add unit tests** for each public method, especially `saveModule` validation logic.  
7. **Introduce constants** for config keys instead of hard‑coded strings.  
8. **Wrap persistence calls in a transaction** or use service‑level transactions to ensure atomicity.  

---

**Verdict** – The class fulfills its intended CRUD functionality but would benefit significantly from modern Java practices, better error handling, and stronger security measures. Addressing the points above would improve maintainability, reliability, and safety.

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
import java.util.Iterator;
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
import com.salesmanager.core.entity.reference.ModuleConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.ShippingUtil;
import com.salesmanager.core.util.StringUtil;

//@Validation
public class ShippingupsxmlAction extends ShippingModuleAction {

	// user selections submited from the page
	private List globalServicesSelection = null;
	private String packageSelection = null;
	private IntegrationKeys keys;
	private IntegrationProperties properties;

	private final static String moduleid = "upsxml";

	private ConfigurationResponse configurations;

	private Map<String, String> globalServicesMap;// available services options
	private Map<String, String> packageMap;// available packages options options

	private Logger log = Logger.getLogger(ShippingupsxmlAction.class);

	public void prepareModule() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);

		Locale locale = getLocale();

		ModuleConfiguration serviceconfig = rservice.getModuleConfiguration(
				moduleid, "service", "XX");


		if (serviceconfig == null) {
			throw new Exception("ModuleConfiguration does not exist for "
					+ moduleid + "-service-XX");
		}

		// parse services ONLY GLOBAL SERVICES

		Map servicemap = ShippingUtil.buildServiceMap(moduleid, locale);
		if (servicemap != null) {
			setGlobalServicesMap(servicemap);
		}
		// }

		// parse packages

		Map packages = ShippingUtil.buildPackageMap(moduleid, locale);
		if (packages != null) {
			setPackageMap(packages);
		}
		// }

		// get merchant configs

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		ConfigurationResponse config = mservice.getConfigurationByModule(
				moduleid, merchantid);
		
		this.setConfigurations(config);

	}

	public void displayModule() throws Exception {

		if (configurations != null) {
			IntegrationKeys keys = (IntegrationKeys) configurations
					.getConfiguration("upsxml-keys");
			setKeys(keys);

			IntegrationProperties props = (IntegrationProperties) configurations
					.getConfiguration("upsxml-properties");
			setProperties(props);

			// choosen package [1 package allowed]
			String packageoption = (String) configurations
					.getConfiguration("package-upsxml");
			if (!StringUtils.isBlank(packageoption)) {
				setPackageSelection(packageoption);
			} else {// default value
				setPackageSelection("02");
			}

			// selected services
			Map selectedservices = (Map) configurations
					.getConfiguration("service-global-upsxml");
			
			// no distinction between domestic and intl

			if (selectedservices != null) {
				Iterator i = selectedservices.keySet().iterator();
				List slist = new ArrayList();
				while (i.hasNext()) {
					String key = (String) i.next();
					slist.add(key);
				}
				setGlobalServicesSelection(slist);
			}

		}

	}

	/**
	 * Those validations are disabled
	 */
	/**
	 * @Validations( requiredFields = {
	 * @RequiredFieldValidator(message = "", key = "errors.required.upskey",
	 *                                 fieldName = "keys.key1"),
	 * @RequiredFieldValidator(message = "", key = "errors.required.userid",
	 *                                 fieldName = "keys.userId"),
	 * @RequiredFieldValidator(message = "", key =
	 *                                 "errors.required.upspassword", fieldName
	 *                                 = "keys.errors.required.upspassword") } )
	 **/

	public void saveModule() throws Exception {

		boolean fielderror = false;
		if (this.getKeys() == null
				|| StringUtils.isBlank(this.getKeys().getKey1())) {
			addFieldError("keys.key1", getText("errors.required.upskey"));
			fielderror = true;
		}
		if (StringUtils.isBlank(this.getKeys().getUserid())) {
			addFieldError("keys.userid", getText("errors.required.userid"));
			fielderror = true;
		}
		if (StringUtils.isBlank(this.getKeys().getPassword())) {
			addFieldError("keys.password", getText("errors.required.password"));
			fielderror = true;
		}

		if (this.getGlobalServicesSelection() == null
				|| this.getGlobalServicesSelection().size() == 0
				|| this.getGlobalServicesSelection().size() > 3) {
			addFieldError("globalServicesSelection",
					getText("message.error.maxglobalshipping"));
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

		String submitedcredentials = ShippingUtil.buildShippingKeyLine(keys);
		String submitedproperties = ShippingUtil
				.buildShippingPropertiesLine(this.getProperties());

		String serviceline = StringUtil
				.buildMultipleValueLine(getGlobalServicesSelection());
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
				pack.setConfigurationValue2(serviceline);
			} else {
				pack = new MerchantConfiguration();
				pack
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
				pack.setConfigurationModule(moduleid);
				pack.setDateAdded(date);
				pack.setConfigurationValue(packageOption);
				pack.setMerchantId(merchantid);
				pack.setConfigurationValue2(serviceline);
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
			pack.setConfigurationValue2(serviceline);
			modulestosave.add(pack);
		}

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		mservice.saveOrUpdateMerchantConfigurations(modulestosave);



	}

	public void deleteModule() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		mservice.cleanConfigurationLikeKeyModule("SHP_RT_", moduleid,
				merchantid);

	}

	public IntegrationKeys getKeys() {
		return keys;
	}

	public void setKeys(IntegrationKeys keys) {
		this.keys = keys;
	}

	public ConfigurationResponse getConfigurations() {
		return configurations;
	}

	public void setConfigurations(ConfigurationResponse configurations) {
		this.configurations = configurations;
	}

	public Map<String, String> getPackageMap() {
		return packageMap;
	}

	public void setPackageMap(Map<String, String> packageMap) {
		this.packageMap = packageMap;
	}

	public Map<String, String> getGlobalServicesMap() {
		return globalServicesMap;
	}

	public void setGlobalServicesMap(Map<String, String> globalServicesMap) {
		this.globalServicesMap = globalServicesMap;
	}

	public String getPackageSelection() {
		return packageSelection;
	}

	public void setPackageSelection(String packageSelection) {
		this.packageSelection = packageSelection;
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
