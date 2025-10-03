# PaymentModuleAction.java

## Review

## 1. Summary  
**Purpose**  
`PaymentModuleAction` is an abstract Struts‑style action that handles CRUD operations for payment modules (e.g., PayPal, Authorize.Net, COD) in the SalesManager system. It is responsible for:  
* Loading current payment module configuration from the database.  
* Rendering the module’s configuration page (`display`).  
* Persisting configuration changes (`save`).  
* Removing a module’s configuration (`delete`).  

**Key components**  
* **`prepare()`** – called before every action to populate configuration maps and module information from the HTTP request and merchant context.  
* **`display()`, `save()`, `delete()`** – public action methods invoked by the framework.  
* **Abstract hooks** – `displayModule()`, `saveModule()`, `deleteModule()`, `prepareModule()` that concrete subclasses must implement for module‑specific UI and data handling.  
* **`MerchantService` / `ReferenceService`** – service layer proxies for CRUD and reference data.  
* **`ConfigurationResponse`** – DTO holding all configuration key/value pairs for the merchant.  

**Design patterns & frameworks**  
* *Template Method* – `PaymentModuleAction` defines the algorithmic skeleton (prepare → module hook → core logic) while delegating module‑specific work to subclasses.  
* *Strategy* – The use of `ReferenceService.getPaymentMethodsMap()` retrieves module metadata (sub‑type, name).  
* *Data‑Transfer Object (DTO)* – `ConfigurationResponse` and `MerchantConfiguration`.  
* *Frameworks* – Apache Struts (action pattern), Apache Commons Configuration/Lang, Log4j, and a custom `ServiceFactory` for dependency lookup.  

---

## 2. Detailed Description  

### 2.1 Initialization (`prepare`)  
1. **Merchant context** – fetches the merchant id from the HTTP session (`ProfileConstants.context`).  
2. **Configuration loading** – builds a `ConfigurationRequest` (prefix `"MD_PAY_"`) and retrieves a `ConfigurationResponse` that contains all merchant payment settings.  
3. **Map construction** – iterates over each `MerchantConfiguration` to populate:  
   * `configurationModuleNames` – map of module‑name → configuration (indicator of whether the module is enabled).  
   * `configurationModuleGatewayNames` – map of gateway‑module-name → configuration.  
4. **Request path parsing** – extracts the module id from the URI (e.g., `/payment/pp_display.action` → module id `"pp"`).  
5. **Current module state** – if the module is already configured, pulls its enabled flag and sets `currentModuleName`/`currentModuleEnabled`.  
6. **Hook** – calls `prepareModule()` to allow subclasses to perform any module‑specific pre‑processing.  

### 2.2 Runtime Behavior  
* **`display()`**  
  * Invokes `prepare()`.  
  * Sets `moduleEnabled` to the current enabled flag.  
  * Calls abstract `displayModule()` which renders the configuration UI.  

* **`save()`**  
  * Calls `prepare()` to get fresh config maps.  
  * Uses `ReferenceService` to look up the module’s metadata (to determine if it is a gateway).  
  * Builds a list `updateableModules` that contains the configuration objects to be persisted:  
    * If only one module is configured, it toggles that one.  
    * If multiple modules exist, it disables all except the submitted one, handling gateway semantics specially.  
  * Delegates to `saveModule()` for module‑specific persistence.  
  * Persists all configurations via `MerchantService.saveOrUpdateMerchantConfigurations`.  

* **`delete()`**  
  * Calls `prepare()`.  
  * Deletes the configuration entry for the module (both indicator and gateway entry, if present).  
  * Delegates to `deleteModule()` for module‑specific cleanup.  

### 2.3 Cleanup  
`delete()` and the service calls clean up any stale configuration rows. The abstract hooks allow subclasses to perform any necessary resource cleanup (e.g., removing external API credentials).  

### 2.4 Assumptions & Constraints  
* The URL structure is strictly `/payment/{moduleId}_display.action`.  
* Only one payment module can be *enabled* at a time; the logic disables all others when enabling a new one.  
* Gateways are identified by `coreModuleServiceSubtype == 1`.  
* Configuration keys are prefixed with `"MD_PAY_"` and contain either `MODULE_PAYMENT_INDICATOR_NAME` or `MODULE_PAYMENT_GATEWAY`.  
* `MerchantService` and `ReferenceService` are singletons obtained via `ServiceFactory`.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs / Side‑Effects |
|--------|---------|--------|------------------------|
| `prepare()` | Initializes configuration maps, extracts module id, sets current state. | None (reads from session & request) | Populates `configurationModuleNames`, `configurationModuleGatewayNames`, `moduleName`, `currentModuleName/Enabled`; calls `prepareModule()` |
| `display()` | Render module configuration page. | None | Returns `"SUCCESS"`; sets `moduleEnabled`; delegates to `displayModule()` |
| `save()` | Persist configuration changes. | None (uses request parameters) | Updates configuration entries, calls `saveModule()`, persists via service, sets success message; returns `"SUCCESS"` |
| `delete()` | Remove module configuration. | None | Deletes entries via service, calls `deleteModule()`, sets success message; returns `"deletecomplete"` |
| `displayModule()` | Abstract hook – concrete UI rendering. | None | Implementation‑specific |
| `saveModule()` | Abstract hook – concrete data persistence beyond generic config. | None | Implementation‑specific |
| `deleteModule()` | Abstract hook – concrete cleanup. | None | Implementation‑specific |
| `prepareModule()` | Abstract hook – concrete pre‑processing. | None | Implementation‑specific |
| Getters/Setters | Standard JavaBean accessors for fields. | – | – |

---

## 4. Dependencies  

| External | Type | Notes |
|----------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Used for static configuration (via `PropertiesHelper`). |
| `org.apache.commons.lang.StringUtils` | Third‑party | Utility for string manipulation. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.*` | Internal | All domain entities, services, constants. |
| `ServiceFactory` | Internal | Simple service locator pattern. |
| `ReferenceService` | Internal | Provides payment method metadata. |
| `MerchantService` | Internal | CRUD for merchant configuration. |

No platform‑specific code is present; the class relies on a servlet container (Struts) and a service layer.

---

## 5. Additional Notes & Recommendations  

### 5.1 Edge Cases & Robustness  
* **URI Parsing** – If the request URI does not contain an underscore (`_`) or is malformed, `pathnocontext.indexOf("_")` will throw `StringIndexOutOfBoundsException`. A guard should be added.  
* **Null Checks** – Several collections (`configurationModuleNames`, `configurationModuleGatewayNames`) are lazily instantiated; however, code later assumes they are non‑null (e.g., `this.getConfigurationModuleNames().containsKey`). Defensive copying or early initialization would reduce NPE risk.  
* **Concurrent Modifications** – The action is stateless per request, but if two concurrent requests update the same module, race conditions may occur. Optimistic locking in the DAO layer or a transaction boundary in `MerchantService` should be ensured.  
* **Gateway Handling** – The logic for disabling gateway modules relies on `module.getCoreModuleServiceSubtype() == 1`. If new sub‑types are added, the code may silently fail to handle them. A more explicit enum or strategy would improve maintainability.  

### 5.2 Code Quality  
* **Magic Strings** – `"MD_PAY_"`, `"true"`, `"false"`, and the configuration key constants should be consistently referenced through constants.  
* **Repeated Logic** – The creation of a new `MerchantConfiguration` is duplicated across branches; a helper method would reduce duplication.  
* **Logging** – No log statements are present aside from the logger definition; adding debug logs for state changes would aid troubleshooting.  
* **Exception Handling** – All public methods declare `throws Exception`. Narrower exception handling or a custom unchecked exception would improve clarity.  

### 5.3 Future Enhancements  
1. **Unit Tests** – Extract business logic into a pure service class (`PaymentModuleManager`) that can be unit‑tested without Struts or servlet mocks.  
2. **Builder Pattern** – Use a builder for `MerchantConfiguration` to simplify object creation.  
3. **Configuration Validation** – Before persisting, validate that mandatory fields are present (e.g., API keys for gateways).  
4. **Internationalization** – `setPageTitle("module." + moduleid)` relies on external i18n resources; ensure fallback handling.  
5. **UI Integration** – Modernize the UI with AJAX or a single‑page approach, delegating to this backend via JSON.  

---

### 5.4 Summary of Strengths  
* Clear separation of generic payment module handling from module‑specific logic via abstract hooks.  
* Uses a configuration‑prefix approach to group payment settings, simplifying retrieval.  
* Leverages existing services and reference data to avoid duplication.  

### 5.5 Summary of Weaknesses  
* Tight coupling to request URL structure and magic strings.  
* Repetitive code blocks and lack of helper utilities.  
* Potential null pointer exceptions and concurrency concerns.  

With the above adjustments, the class would become more robust, maintainable, and easier to test.

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
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;

public abstract class PaymentModuleAction extends BaseAction {

	private String currentModuleEnabled;// used in the jsp page as a checkbox
	private String currentModuleName;// paypal, moneris, linkpoint,
										// authorizenet, psigate, cod...
	private String moduleName;// module submited
	private String moduleEnabled;// module submited

	private ConfigurationResponse configurationVo;// expose to sub classes

	private Map<String, MerchantConfiguration> configurationModuleNames;
	private Map<String, MerchantConfiguration> configurationModuleGatewayNames;
	private Configuration conf = PropertiesHelper.getConfiguration();

	private String message;// special message

	private Logger log = Logger.getLogger(PaymentModuleAction.class);

	public void prepare() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		// Get everything related to payment
		ConfigurationRequest requestvo = new ConfigurationRequest(merchantid
				.intValue(), true, "MD_PAY_");
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
				if (key.equals(PaymentConstants.MODULE_PAYMENT_INDICATOR_NAME)) {// module
																					// configured
					if (configurationModuleNames == null)
						configurationModuleNames = new HashMap();
					configurationModuleNames.put(m.getConfigurationValue1(), m);
				}

				if (key.contains(PaymentConstants.MODULE_PAYMENT_GATEWAY)) {// gateway
																			// module configured
					if (configurationModuleGatewayNames == null)
						configurationModuleGatewayNames = new HashMap();
					configurationModuleGatewayNames.put(m
							.getConfigurationModule(), m);
				}

			}
		}

		// get module name
		String pathnocontext = StringUtils.removeStart(super
				.getServletRequest().getRequestURI(), super.getServletRequest()
				.getContextPath()
				+ "/payment/");
		// pathnocontext is moduleid/dsiplay.action
		// retreive moduleid
		String moduleid = pathnocontext
				.substring(0, pathnocontext.indexOf("_"));
		this.setModuleName(moduleid);
		super.getServletRequest().setAttribute("paymentModule", moduleid);
		
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

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		Map modules = rservice.getPaymentMethodsMap(ctx.getCountryid());
		CoreModuleService module = (CoreModuleService) modules.get(this
				.getModuleName());

		if (this.getConfigurationModuleNames() != null) {// RT is configured

			// ONE ONLY ALLOWED
			// if one module configured
			if (this.getConfigurationModuleNames().size() == 1) {

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

					// check if it is a gateway

					if (module != null
							&& module.getCoreModuleServiceSubtype() == 1
							&& this.getModuleEnabled().equals("true")) {

						if (configurationModuleGatewayNames != null
								&& configurationModuleGatewayNames
										.containsKey(conf
												.getConfigurationValue1())) {

							conf.setConfigurationValue("");

						}

					}
					updateableModules.add(conf);

					// create a new one
					MerchantConfiguration newconfiguration = new MerchantConfiguration();
					newconfiguration
							.setConfigurationKey(PaymentConstants.MODULE_PAYMENT_INDICATOR_NAME);
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

				// if gateway submited
				if (module != null && module.getCoreModuleServiceSubtype() == 1
						&& this.getModuleEnabled().equals("true")

						&& this.getConfigurationModuleGatewayNames() != null) {

					Map coll = this.getConfigurationModuleGatewayNames();

					Iterator i = coll.keySet().iterator();
					while (i.hasNext()) {

						String key = (String) i.next();
						if (configurationModuleNames != null) {

							MerchantConfiguration conf = (MerchantConfiguration) configurationModuleNames
									.get(key);
							if(conf!=null) {
								conf.setConfigurationValue("");
								updateableModules.add(conf);
							}

						}

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
							.setConfigurationKey(PaymentConstants.MODULE_PAYMENT_INDICATOR_NAME);
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

		} else {// Nothing configured

			MerchantConfiguration newconfiguration = new MerchantConfiguration();
			newconfiguration
					.setConfigurationKey(PaymentConstants.MODULE_PAYMENT_INDICATOR_NAME);
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

		if (this.getConfigurationModuleNames()
				.containsKey(this.getModuleName())) {
			MerchantConfiguration conf = (MerchantConfiguration) this
					.getConfigurationModuleNames().get(this.getModuleName());
			mservice.deleteMerchantConfiguration(conf);
		}
		
		if (this.getConfigurationModuleGatewayNames()
				.containsKey(this.getModuleName())) {
			MerchantConfiguration conf = (MerchantConfiguration) this
					.getConfigurationModuleGatewayNames().get(this.getModuleName());
			mservice.deleteMerchantConfiguration(conf);
		}


		this.deleteModule();
		super.setSuccessMessage();

		return "deletecomplete";
	}

	public abstract void displayModule() throws Exception;

	public abstract void saveModule() throws Exception;

	public abstract void deleteModule() throws Exception;

	public abstract void prepareModule() throws Exception;

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

	public String getMessage() {
		return message;
	}

	public void setMessage(String message) {
		this.message = message;
	}

	public Map<String, MerchantConfiguration> getConfigurationModuleGatewayNames() {
		return configurationModuleGatewayNames;
	}

	public void setConfigurationModuleGatewayNames(
			Map<String, MerchantConfiguration> configurationModuleGatewayNames) {
		this.configurationModuleGatewayNames = configurationModuleGatewayNames;
	}

}



```
