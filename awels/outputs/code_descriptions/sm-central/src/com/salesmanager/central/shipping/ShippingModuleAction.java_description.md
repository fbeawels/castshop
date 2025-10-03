# ShippingModuleAction.java

## Review

## 1. Summary

`ShippingModuleAction` is an abstract Struts‑style action that manages the
configuration of shipping modules (e.g., UPS, FedEx) for a merchant.
The class is responsible for:

* Loading the current shipping configuration from the database.
* Determining which module is currently active/selected.
* Providing a user‑interface for displaying, editing, or deleting a module.
* Persisting any changes back to the database.

It delegates the actual UI rendering, module‑specific saving, and module
cleanup to subclasses via the abstract methods `displayModule()`,
`saveModule()`, `deleteModule()`, and `prepareModule()`.

The code heavily relies on `MerchantService` for CRUD operations, the
`MerchantConfiguration` entity for configuration storage, and a
configuration file for a single flag that allows or forbids multiple
shipping modules.

---

## 2. Detailed Description

### 2.1 Core Components

| Component | Role |
|-----------|------|
| `ShippingModuleAction` | Base action for all shipping‑module actions. |
| `MerchantService` | Service layer used to read/write `MerchantConfiguration` objects. |
| `MerchantConfiguration` | Entity that stores a key/value pair for a merchant’s shipping settings. |
| `ShippingConstants` | Constants that hold configuration keys (`MODULE_SHIPPING_RT_MODULE_INDIC_NAME`,
  `INTERNATIONAL_SHIPPING`). |
| `Configuration` (`conf`) | Holds global application settings (e.g. `central.modules.shipping.rt.allowmultiplemodules`). |
| `Context` | Stores per‑session data such as the current `merchantid`. |

### 2.2 Execution Flow

| Phase | What Happens |
|-------|--------------|
| **Preparation** (`prepare()`) | * Retrieve the current merchant’s ID from the session. <br>* Load all `SHP_` configuration keys. <br>* Extract the shipping type (national/international) and store it in `shippingType`. <br>* Build a map (`configurationModuleNames`) that maps module identifiers to their `MerchantConfiguration` records. <br>* Derive the module ID from the request URI (e.g. `/shipping/UPS_display.action` → `UPS`). <br>* Populate request attributes for the view. <br>* Call `prepareModule()` for subclass‑specific preparation. |
| **Display** (`display()`) | Calls `prepare()`, sets `moduleEnabled` from the current configuration, and delegates rendering to `displayModule()`. |
| **Save** (`save()`) | Calls `prepare()`, then computes a list of `MerchantConfiguration` objects that need to be updated or created based on the following rules:<br> * If only one module is allowed (`allowmultiplemodules=false`) the action will: <br>   - Keep the existing module if it is the same as the submitted one, otherwise reset the existing module and create a new one. <br>   - If multiple modules are already configured, disable all but the submitted one. <br>* If multiple modules are allowed, simply enable/disable the submitted module. <br>After computing the new or updated configurations, `saveModule()` is invoked for any module‑specific persistence logic, then `mservice.saveOrUpdateMerchantConfigurations(updateableModules)` persists the changes. |
| **Delete** (`delete()`) | Calls `prepare()`, deletes the configuration record that matches the submitted module ID, invokes `deleteModule()` for subclass cleanup, and returns a `"deletecomplete"` result. |

### 2.3 Assumptions & Constraints

* The request URI always contains a module identifier followed by an underscore.
* `MerchantConfiguration` values for module status are stored as the string `"true"` or an empty string (`""`).
* The configuration property `central.modules.shipping.rt.allowmultiplemodules` is a string (`"true"`/`"false"`).
* The application uses Struts‑2 (`prepare()` pattern) but the code does not actually implement `Preparable`. The developer left a comment about enabling that later.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs / Side Effects |
|--------|---------|--------|------------------------|
| `prepare()` | Load all `SHP_` configs, build module map, set request attributes, determine current module status. | None (reads from session & request) | Sets internal state (`shippingType`, `moduleName`, `currentModuleEnabled`, `configurationModuleNames`, etc.) |
| `display()` | Display the module configuration UI. | None | Calls `prepare()`, sets `moduleEnabled`, calls abstract `displayModule()`. Returns `"SUCCESS"`. |
| `save()` | Persist changes to a shipping module. | None (uses submitted parameters) | Builds a list of `MerchantConfiguration` objects, calls abstract `saveModule()`, saves via `MerchantService`. Returns `"SUCCESS"`. |
| `delete()` | Remove a module configuration. | None | Calls `prepare()`, deletes the configuration from DB, calls abstract `deleteModule()`. Returns `"deletecomplete"`. |
| `displayModule()`, `saveModule()`, `deleteModule()`, `prepareModule()` | Abstract hooks for subclasses to implement module‑specific behavior (UI rendering, custom persistence, cleanup). | None | Implementation‑specific. |
| Getters / Setters (e.g., `getShippingType()`, `setShippingType()`, etc.) | Standard JavaBean accessors for form fields. | None | Return/modify the corresponding field. |

### Notable Utility / Re‑usable Methods

* None. The class contains only domain logic and abstract hooks.  
  (Potential improvement: extract the module‑status logic into a helper method.)

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads global application properties. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Used for string manipulation (e.g., `removeStart`, `isBlank`). |
| `org.apache.log4j.Logger` | Third‑party | Logging; however it is declared but never used. |
| `com.salesmanager.central.BaseAction` | Internal | Likely a Struts action base class. |
| `com.salesmanager.central.profile.Context` | Internal | Stores session context (merchant ID). |
| `com.salesmanager.central.profile.ProfileConstants` | Internal | Holds session attribute keys. |
| `com.salesmanager.central.util.PropertiesHelper` | Internal | Provides a singleton `Configuration`. |
| `com.salesmanager.core.constants.ShippingConstants` | Internal | Contains constant keys. |
| `com.salesmanager.core.entity.merchant.MerchantConfiguration` | Internal | JPA/Hibernate entity. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Factory for service objects. |
| `com.salesmanager.core.service.merchant.*` | Internal | Service and request/response DTOs. |

All dependencies are either Java standard libraries or third‑party Apache commons / log4j. No platform‑specific dependencies are evident.

---

## 5. Additional Notes & Recommendations

### 5.1 Code Quality Issues

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw collections** (`List`, `Iterator`, `Map`) | Unchecked casts; risk of `ClassCastException`. | Use generics (`List<MerchantConfiguration>`, `Iterator<MerchantConfiguration>`, `Map<String, MerchantConfiguration>`). |
| **String‑based boolean flags** (`"true"` / `""`) | Hard to read; error‑prone. | Store booleans as `true`/`false` strings or use a dedicated enum. |
| **Magic strings** (`"true"`, `"false"`, configuration keys) | Hard to maintain. | Centralize in constants or enums. |
| **Hard‑coded URI parsing** (`substring` and `indexOf("_")`) | Breaks if URI pattern changes. | Use Struts parameters or a more robust path extractor. |
| **Unnecessary null‑check** on `conf.getProperty(...)` | Minor inefficiency. | Use `Boolean.parseBoolean(conf.getProperty(...))` or wrap in a helper. |
| **No `@Override` annotations** | Possible signature mismatch. | Add `@Override` to methods overriding base class methods. |
| **Unused `Logger`** | Dead code. | Either use it for debugging or remove. |
| **Lack of input validation** | Security and data integrity risk. | Validate `moduleName`, `moduleEnabled`, etc. before processing. |
| **Redundant `config` variable** | Confusing. | Remove or rename to `currentConf`. |
| **Exception handling** | Swallowing exceptions may hide problems. | Log or re‑throw exceptions with meaningful messages. |

### 5.2 Design Improvements

1. **Encapsulate Module Logic** – The `save()` method contains long nested conditionals that handle all module‑status scenarios. Extract this logic into a small helper class (e.g., `ShippingModuleManager`) with clear methods like `enableModule`, `disableModule`, `allowMultipleModules()`, etc. This will make `save()` easier to read and test.

2. **Use Enums** – Define an enum `ShippingMode { NATIONAL, INTERNATIONAL }` and another for `ModuleEnabled { ENABLED, DISABLED }`. This replaces string flags and improves type safety.

3. **Parameter Binding** – Instead of parsing the request URI, let Struts inject the `moduleName` as an action parameter (e.g., via `<s:url>` or `<s:param>`). This removes fragile string manipulation.

4. **Persisting Changes** – The `MerchantService` API could accept a `Set<MerchantConfiguration>` rather than a `List`. Also consider using batch updates or transaction boundaries.

5. **Unit Tests** – Given the heavy business logic, add unit tests that exercise all branches of `save()` and `delete()`.

6. **Internationalization** – The page title is set as `module.` + moduleid. Use i18n properties rather than hard‑coded keys.

7. **Logging** – Add meaningful log statements at key decision points (e.g., “Enabling module X”, “Disabling module Y”) to aid troubleshooting.

### 5.3 Edge Cases Not Handled

* **Invalid URI** – If the request does not contain an underscore or a valid module ID, `pathnocontext.indexOf("_")` will return `-1` and `substring` will throw an exception.
* **Null `moduleName` / `moduleEnabled`** – The code checks `moduleEnabled` but not `moduleName`. If `moduleName` is null, the logic may fail or create an empty key.
* **Concurrent updates** – Two users editing the same module could cause race conditions. Consider optimistic locking on `MerchantConfiguration`.
* **Empty `MerchantConfiguration` list** – If the merchant has no shipping configuration, `configvo` will be null, which could lead to NPE in subsequent calls.

### 5.4 Future Enhancements

* **Dynamic Module Discovery** – Instead of hard‑coding shipping modules, scan a plugins directory or database table to load available modules at runtime.
* **RESTful API** – Replace Struts actions with a REST controller (e.g., Spring MVC) for easier integration.
* **Caching** – Cache the module configuration per merchant to reduce database round‑trips.
* **Audit Trail** – Store who changed the module and when.

---

### Bottom Line

`ShippingModuleAction` is a functional but heavily coupled, legacy‑style action class that manages shipping module configuration. The core algorithm for enabling/disabling modules works, but the implementation is tangled and difficult to maintain. Refactoring to use generics, enums, a dedicated helper class, and improved error handling would greatly improve readability, testability, and robustness. The class serves as a solid foundation for module‑specific actions, but it would benefit from modernization and stricter code quality controls.

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
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.util.PropertiesHelper;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;

/**
 * Takes care of displaying, editind and removing configuration for a given
 * shipping module
 * 
 * @author Administrator
 * 
 */
// public abstract class ShippingModuleAction extends CentralBaseAction
// implements Preparable {
public abstract class ShippingModuleAction extends BaseAction {

	private String currentModuleEnabled;// used in the jsp page as a checkbox
	private String shippingType;// national or international
	private String currentModuleName;// upsxml, fedex, canadapost, usps already
										// configured
	private String moduleName;// module submited
	private String moduleEnabled;// module submited

	private ConfigurationResponse configurationVo;// expose to sub classes

	private Map<String, MerchantConfiguration> configurationModuleNames;
	private Configuration conf = PropertiesHelper.getConfiguration();

	private Logger log = Logger.getLogger(ShippingModuleAction.class);

	public void prepare() throws Exception {

		// SHP_RT_INDNM indicator and name [name coma delimited]
		// SHP_RT_INDNMCRED indicator, name & credentials, will have to remove
		// the credentials

		// SHP_ZONES_SHIPPING international or domestic

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();


		// Get everything related to shipping
		ConfigurationRequest requestvo = new ConfigurationRequest(merchantid
				.intValue(), true, "SHP_");
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		ConfigurationResponse responsevo = mservice.getConfiguration(requestvo);
		List config = responsevo.getMerchantConfigurationList();

		if (config != null) {
			this.setConfigurationVo(responsevo);
			Iterator it = config.iterator();
			while (it.hasNext()) {

				MerchantConfiguration m = (MerchantConfiguration) it.next();

				String key = m.getConfigurationKey();
				if (key
						.equals(ShippingConstants.INTERNATIONAL_SHIPPING)) {// national
																					// or
																					// international
					this.setShippingType(m.getConfigurationValue());
					super.getServletRequest().setAttribute("zoonesshipping",
							m.getConfigurationValue());// this is for the
														// include shipping page
				} else if (key
						.equals(ShippingConstants.MODULE_SHIPPING_RT_MODULE_INDIC_NAME)) {// indicator
																							// and
																							// name
					// @TODO, parse token ?
					if (configurationModuleNames == null)
						configurationModuleNames = new HashMap();
					configurationModuleNames.put(m.getConfigurationValue1(), m);
				}

			}
		}

		// get module name
		String pathnocontext = StringUtils.removeStart(super
				.getServletRequest().getRequestURI(), super.getServletRequest()
				.getContextPath()
				+ "/shipping/");
		// pathnocontext is moduleid/dsiplay.action
		// retreive moduleid
		String moduleid = pathnocontext
				.substring(0, pathnocontext.indexOf("_"));

		this.setModuleName(moduleid);
		super.getServletRequest().setAttribute("shippingModule", moduleid);
		super.setPageTitle("module." + moduleid);

		if (this.getConfigurationModuleNames() != null
				&& this.getConfigurationModuleNames().containsKey(moduleid)) {
			MerchantConfiguration conf = (MerchantConfiguration) this
					.getConfigurationModuleNames().get(moduleid);
			if (conf.getConfigurationValue1() != null
					&& !conf.getConfigurationValue1().equals("")) {
				this.setCurrentModuleName(moduleid);
			}
			if (!StringUtils.isBlank(conf.getConfigurationValue())
					&& conf.getConfigurationValue().equals("true")) {
				this.setCurrentModuleEnabled("true");
			} else {
				this.setCurrentModuleEnabled("false");
			}
		}

		this.prepareModule();

	}

	public String display() throws Exception {

		// @todo, remove if re-enable preparable
		this.prepare();

		this.setModuleEnabled(this.getCurrentModuleEnabled());
		displayModule();

		return SUCCESS;
	}

	public String save() throws Exception {

		this.prepare();
		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		if (configurationModuleNames == null) {
			configurationModuleNames = new HashMap();
		}

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		Date dt = new Date();

		MerchantConfiguration config = null;

		List updateableModules = new ArrayList();

		if (this.getConfigurationModuleNames() != null) {// RT is configured
			// check if only one allowed
			if (conf
					.getProperty("central.modules.shipping.rt.allowmultiplemodules") == null
					|| conf.getProperty(
							"central.modules.shipping.rt.allowmultiplemodules")
							.equals("false")) {
				// ONE ONLY ALLOWED
				// if one module configured
				if (this.getConfigurationModuleNames().size() == 1) {

					// if module== this module, set to true and update, else set
					// to false update and create a new one

					if (this.getConfigurationModuleNames().containsKey(
							this.getCurrentModuleName())) {// this is the one
															// configured
						// Same module configured
						MerchantConfiguration mconf = (MerchantConfiguration) this
								.getConfigurationModuleNames().get(
										this.getCurrentModuleName());
						if (this.getModuleEnabled() != null
								&& this.getModuleEnabled().equals("true")) {
							mconf.setConfigurationValue("true");
						} else {
							mconf.setConfigurationValue("false");
						}
						updateableModules.add(mconf);
					} else {
						// Get module, set flag to false
						Collection coll = this.getConfigurationModuleNames()
								.values();
						Object[] obj = coll.toArray();

						MerchantConfiguration conf = (MerchantConfiguration) obj[0];
						if (this.getModuleEnabled() != null
								&& this.getModuleEnabled().equals("true")) {
							// conf.setConfigurationValue("true");
							conf.setConfigurationValue("");
						}
						updateableModules.add(conf);

						// create a new one
						MerchantConfiguration newconfiguration = new MerchantConfiguration();
						newconfiguration
								.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_MODULE_INDIC_NAME);
						newconfiguration.setConfigurationModule("");
						newconfiguration.setMerchantId(merchantid);
						if (this.getModuleEnabled() != null
								&& this.getModuleEnabled().equals("true")) {
							newconfiguration.setConfigurationValue("true");
						} else {
							newconfiguration.setConfigurationValue("");
						}
						newconfiguration.setConfigurationValue1(this
								.getModuleName());
						newconfiguration.setDateAdded(new Date(dt.getTime()));
						
						config = newconfiguration;

						updateableModules.add(newconfiguration);

					}
				} else {// keep the good one and set to false the others

					// delete all modules
					Collection coll = this.getConfigurationModuleNames()
							.values();

					Iterator i = coll.iterator();
					while (i.hasNext()) {
						MerchantConfiguration conf = (MerchantConfiguration) i
								.next();
						if (!conf.getConfigurationValue1().equals(
								this.getModuleName())) {
							
							conf.setConfigurationValue("");
							updateableModules.add(conf);
						}
					}

					if (this.getConfigurationModuleNames().containsKey(
							this.getModuleName())) {
						// contains submited module
						MerchantConfiguration conf = this
								.getConfigurationModuleNames().get(
										this.getModuleName());
						if (this.getModuleEnabled() != null
								&& this.getModuleEnabled().equals("true")) {
							conf.setConfigurationValue("true");
						} else {
							conf.setConfigurationValue("");
						}
						
						updateableModules.add(conf);
						config = conf;
					} else {// create a new one

						MerchantConfiguration newconfiguration = new MerchantConfiguration();
						newconfiguration
								.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_MODULE_INDIC_NAME);
						newconfiguration.setConfigurationModule("");
						newconfiguration.setMerchantId(merchantid);
						if (this.getModuleEnabled() != null
								&& this.getModuleEnabled().equals("true")) {
							newconfiguration.setConfigurationValue("true");
						} else {
							newconfiguration.setConfigurationValue("");
						}
						newconfiguration.setConfigurationValue1(this
								.getModuleName());
						newconfiguration.setDateAdded(new Date(dt.getTime()));
						
						config = newconfiguration;
						this.getConfigurationModuleNames().put(
								this.getModuleName(), newconfiguration);
						updateableModules.add(newconfiguration);
					}
				}

			} else {
				// MULTIPLE ARE ALLOWED
				if (this.getConfigurationModuleNames().containsKey(
						this.getModuleName())) {
					// contains submited module
					MerchantConfiguration conf = this
							.getConfigurationModuleNames().get(
									this.getModuleName());
					if (this.getModuleEnabled() != null
							&& this.getModuleEnabled().equals("true")) {
						conf.setConfigurationValue("true");
					} else {
						conf.setConfigurationValue("");
					}
					
					config = conf;
					updateableModules.add(conf);
				} else {// create a new one
					MerchantConfiguration newconfiguration = new MerchantConfiguration();
					newconfiguration
							.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_MODULE_INDIC_NAME);
					newconfiguration.setConfigurationModule("");
					newconfiguration.setMerchantId(merchantid);
					if (this.getModuleEnabled() != null
							&& this.getModuleEnabled().equals("true")) {
						newconfiguration.setConfigurationValue("true");
					} else {
						newconfiguration.setConfigurationValue("");
					}
					newconfiguration.setConfigurationValue1(this
							.getModuleName());
					newconfiguration.setDateAdded(new Date(dt.getTime()));
					// mservice.saveOrUpdateMerchantConfiguration(newconfiguration);
					config = newconfiguration;
					this.getConfigurationModuleNames().put(
							this.getModuleName(), newconfiguration);
					updateableModules.add(newconfiguration);
				}

			}

		} else {// Nothing configured

			MerchantConfiguration newconfiguration = new MerchantConfiguration();
			newconfiguration
					.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_MODULE_INDIC_NAME);
			newconfiguration.setConfigurationModule("");
			if (this.getModuleEnabled() != null
					&& this.getModuleEnabled().equals("true")) {
				newconfiguration.setConfigurationValue("true");
			} else {
				newconfiguration.setConfigurationValue("");
			}
			newconfiguration.setConfigurationValue1(this.getModuleName());
			newconfiguration.setDateAdded(new Date(dt.getTime()));
			newconfiguration.setMerchantId(merchantid);
			
			config = newconfiguration;
			this.getConfigurationModuleNames().put(this.getModuleName(),
					newconfiguration);
			updateableModules.add(newconfiguration);
		}

		this.saveModule();
		
		mservice.saveOrUpdateMerchantConfigurations(updateableModules);
		super.setSuccessMessage();

		return SUCCESS;
	}

	public String delete() throws Exception {

		this.prepare();
		// delete module name and indicator
		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		// delete only the good one
		if (this.getConfigurationModuleNames()
				.containsKey(this.getModuleName())) {
			MerchantConfiguration conf = (MerchantConfiguration) this
					.getConfigurationModuleNames().get(this.getModuleName());
			mservice.deleteMerchantConfiguration(conf);
		}
		// }
		this.deleteModule();
		super.setSuccessMessage();

		return "deletecomplete";
	}

	public abstract void displayModule() throws Exception;

	public abstract void saveModule() throws Exception;

	public abstract void deleteModule() throws Exception;

	public abstract void prepareModule() throws Exception;

	public String getShippingType() {
		return shippingType;
	}

	public void setShippingType(String shippingType) {
		this.shippingType = shippingType;
	}

	public String getCurrentModuleName() {
		return currentModuleName;
	}

	public void setCurrentModuleName(String currentModuleName) {
		this.currentModuleName = currentModuleName;
	}

	public String getModuleName() {
		return moduleName;
	}

	public void setModuleName(String moduleName) {
		this.moduleName = moduleName;
	}

	public String getCurrentModuleEnabled() {
		return currentModuleEnabled;
	}

	public void setCurrentModuleEnabled(String currentModuleEnabled) {
		this.currentModuleEnabled = currentModuleEnabled;
	}

	public String getModuleEnabled() {
		return moduleEnabled;
	}

	public void setModuleEnabled(String moduleEnabled) {
		this.moduleEnabled = moduleEnabled;
	}

	public Map<String, MerchantConfiguration> getConfigurationModuleNames() {
		return configurationModuleNames;
	}

	protected ConfigurationResponse getConfigurationVo() {
		return configurationVo;
	}

	protected void setConfigurationVo(ConfigurationResponse configurationVo) {
		this.configurationVo = configurationVo;
	}

}



```
