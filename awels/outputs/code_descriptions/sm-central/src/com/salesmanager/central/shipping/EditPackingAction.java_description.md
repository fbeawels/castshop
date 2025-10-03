# EditPackingAction.java

## Review

## 1. Summary  
**Purpose** – `EditPackingAction` is a Struts/XWork action that displays and updates the configuration of a shipping‑packing module for a merchant. It reads the merchant’s selected packing module, pulls the available modules from a reference service, and displays the configuration options. When the user submits a new configuration, it validates the data via the selected module’s `storeConfiguration` method and persists it.

**Key Components**  
| Component | Role |
|-----------|------|
| `services` | Collection of available packing modules (`CoreModuleService`). |
| `service` | The module selected by the user (form backing object). |
| `boxInformation` | Map of module name → `PackageDetail` (current configuration values). |
| `pageInformation` | Map of module name → configuration file name (used for UI rendering). |
| `sizeUnit` / `weightUnit` | Current units for dimensions/weight from the merchant’s context. |
| `display()` | Loads modules, determines the selected module, and prepares data for the view. |
| `editPackingOption()` | Validates and persists a new configuration submitted by the user. |

**Frameworks / Libraries**  
* Apache Struts / XWork (`BaseAction`, `ValidationException`)  
* Spring (`SpringUtil.getBean`) for dependency injection  
* Apache Commons Lang (`StringUtils`)  
* Apache Log4j (`Logger`)  
* Custom services (`ServiceFactory`, `MerchantService`, `ReferenceService`) from SalesManager core  

The action relies on the `SalesManager` domain model (`MerchantConfiguration`, `PackageDetail`, `CoreModuleService`) and utility classes (`CountryUtil`, `LocaleUtil`, `MessageUtil`).

---

## 2. Detailed Description  

### Execution Flow  

1. **User Request** – The user opens the “Edit Packing” page.  
2. **`display()`**  
   * Sets page title.  
   * Retrieves the current merchant context (country, weight/size units).  
   * Uses `ReferenceService` to fetch all shipping modules of type *P packing*.  
   * Converts the module list to the current locale.  
   * Loads the merchant’s current configuration (`SHP_PACK`).  
   * Determines the selected module (`moduleSelected`), falling back to a default if none exists.  
   * Iterates over each module:
     * Retrieves the Spring bean (`CalculatePackingModule`).  
     * Stores the configuration options file name in `pageInformation`.  
     * If the merchant has a configuration for the module, obtains `PackageDetail` via `getConfigurationOptions` and stores it in `boxInformation`.  
3. **Render** – The Struts view uses getters (`getServices()`, `getBoxInformation()`, `getPageInformation()`, `getModuleSelected()`) to populate the form.  
4. **Form Submit** – The user changes values and submits the form.  
5. **`editPackingOption()`**  
   * Calls `display()` again to rebuild the UI context (redundant but keeps state).  
   * Checks that the `service` object was bound from the form; if not, logs an error and returns `ERROR`.  
   * Retrieves the module bean.  
   * Calls `storeConfiguration` on the bean to validate and persist the new configuration.  
   * Handles `ValidationException` by adding an error message and returning `ERROR`.  
   * On success, sets a success message and returns `SUCCESS`.  

### Assumptions & Constraints  

* The action presumes that the form will bind a `CoreModuleService` (`service`) containing the selected module name.  
* It assumes that every module’s Spring bean implements `CalculatePackingModule`.  
* The merchant’s configuration is stored under the key `"SHP_PACK"`.  
* No transaction boundaries are defined in the action; persistence is delegated to the module’s bean.  
* The code relies on raw types (`Collection`, `Map`) which may cause unchecked cast warnings.  

### Architecture & Design Choices  

* **Separation of Concerns** – The action focuses on web‑level orchestration; business logic resides in the module beans.  
* **Extensibility** – New packing modules can be added as Spring beans without changing the action.  
* **Use of ServiceFactory** – Centralized creation of core services decouples the action from concrete implementations.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `display()` | Prepares module list and current configuration for the UI. | none | `String` – always `SUCCESS` | Populates `services`, `boxInformation`, `pageInformation`, `moduleSelected`, `sizeUnit`, `weightUnit`. Sets technical error on exception. |
| `editPackingOption()` | Validates and persists user‑submitted configuration. | none | `String` – `SUCCESS` or `ERROR` | Calls `display()`, logs errors, sets success or error messages, may throw `ValidationException`. |
| Getters/Setters | Standard bean properties (`boxInformation`, `services`, `moduleSelected`, `service`, `pageInformation`, `shippingInformation`, `sizeUnit`, `weightUnit`). | – | – | – |
| `setContext()` (inherited from `BaseAction`) | Provides the merchant context used throughout. | – | – | – |

The action does not contain reusable utilities; most logic is specific to the packing configuration workflow.

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| `com.opensymphony.xwork2.validator.ValidationException` | Third‑party | XWork validation framework. |
| `org.apache.commons.lang.StringUtils` | Third‑party | Utility for string operations. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.core.*` | Core | Domain entities, services, constants, and utilities. |
| `com.salesmanager.central.*` | Central layer | Base action, context, and reference action. |
| `org.apache.commons.logging` (indirectly via Log4j) | Standard | Logging façade. |

All dependencies are either standard open‑source libraries or internal SalesManager modules. No platform‑specific APIs are used beyond the servlet request/response objects provided by the framework.

---

## 5. Additional Notes  

### Strengths  
* **Modular** – New packing modules can be added as Spring beans without code changes.  
* **Clear separation** – UI logic stays in the action, business logic in the module beans.  
* **Locale support** – Uses `LocaleUtil` to adapt module lists to the user’s language.  

### Weaknesses & Edge Cases  

1. **Raw Types** – `services`, `boxInformation`, `pageInformation` use raw collections and maps.  
   * This leads to unchecked cast warnings and potential `ClassCastException`.  
   * Recommendation: use generics (`Collection<CoreModuleService>`, `Map<String, PackageDetail>`, `Map<String, String>`).

2. **Null‑Pointer Risks**  
   * If `service` is null after form binding, the method logs an error but still proceeds to `display()` (which may overwrite state).  
   * In `editPackingOption()`, `mod.storeConfiguration` can throw a generic `Exception` that is swallowed; only `ValidationException` is caught.

3. **Redundant `display()` Call**  
   * Calling `display()` inside `editPackingOption()` rebuilds all context but does not guarantee that the form’s posted values are preserved.  
   * Consider reusing the already loaded data or at least not re‑initializing `services` unless needed.

4. **Exception Handling**  
   * The outer `try/catch` blocks swallow generic `Exception` and only log the stack trace.  
   * This hides the specific cause from the user and makes debugging harder.  
   * Use more granular exception handling or propagate exceptions to a global error handler.

5. **Hard‑Coded Keys**  
   * The key `"SHP_PACK"` is hard‑coded; if the configuration key changes, the code must be updated.  
   * Introduce a constant or enum for configuration keys.

6. **Performance**  
   * The action repeatedly calls `ReferenceService.getShippingModules()` and `SpringUtil.getBean()` for every module, which could be expensive if many modules exist.  
   * Caching the module bean references could improve performance.

7. **Security**  
   * The action does not perform explicit permission checks; it assumes that the user is already authorized to edit merchant configuration.  
   * Ensure that higher‑level security (e.g., Spring Security, Struts interceptor) validates the merchant’s identity and role.

### Suggested Enhancements  

* **Add Generic Types** – Upgrade all collections and maps to use generics.  
* **Refactor Exception Flow** – Distinguish between validation errors and system errors; provide user‑friendly messages.  
* **Unit Tests** – Write tests for `display()` and `editPackingOption()` to cover successful and error scenarios.  
* **Configuration Constants** – Extract `"SHP_PACK"` and `ShippingConstants.DEFAULT_PACKING_MODULE` into a dedicated constants class.  
* **Logging Improvement** – Include more context in logs (merchant ID, module name) to aid troubleshooting.  
* **Performance Optimisation** – Cache bean lookups and module lists in a request/session scope if they are reused across actions.  
* **UI Validation** – Complement server‑side validation with client‑side checks to reduce unnecessary server round‑trips.

Overall, the action serves its purpose but could benefit from modern Java practices, clearer exception handling, and stronger type safety.

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

import java.util.Collection;
import java.util.Iterator;
import java.util.Map;
import java.util.TreeMap;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.validator.ValidationException;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.ref.RefAction;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.reference.CentralMeasureUnits;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.shipping.PackageDetail;
import com.salesmanager.core.module.model.application.CalculatePackingModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.SpringUtil;

public class EditPackingAction extends BaseAction {

	private Collection services;
	private String moduleSelected = null;// user selection
	private CoreModuleService service;

	private Map boxInformation = new TreeMap();
	private Map pageInformation = new TreeMap();

	private String sizeUnit;
	private String weightUnit;

	private PackageDetail shippingInformation;

	private Logger log = Logger.getLogger(EditPackingAction.class);

	public String display() {

		try {
			
			super.setPageTitle("label.shipping.packing.title");

			Context ctx = super.getContext();
			
			weightUnit = ctx.getWeightunit();
			sizeUnit = ctx.getSizeunit();

	
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			services = rservice
					.getShippingModules(
							ShippingConstants.INTEGRATION_SERVICE_SHIPPING_PACKING_SUBTYPE,
							CountryUtil.getCountryIsoCodeById(ctx
									.getCountryid()));

			LocaleUtil.setLocaleToEntityCollection(services, super.getLocale());

			// get module selected
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationRequest request = new ConfigurationRequest(ctx
					.getMerchantid(), "SHP_PACK");
			ConfigurationResponse response = mservice.getConfiguration(request);

			// get MerchantConfiguration object
			if (response != null) {
				MerchantConfiguration conf = response
						.getMerchantConfiguration("SHP_PACK");

				if (conf != null) {

					// parse values (applies to box module)
					this.setModuleSelected(conf.getConfigurationValue());
				} else {
					this
							.setModuleSelected(ShippingConstants.DEFAULT_PACKING_MODULE);
				}

			} else {
				this
						.setModuleSelected(ShippingConstants.DEFAULT_PACKING_MODULE);
			}

			if (services != null) {
				Iterator it = services.iterator();
				while (it.hasNext()) {
					CoreModuleService conf = (CoreModuleService) it.next();
					String module = conf.getCoreModuleName();

					CalculatePackingModule mod = null;
					try {
						mod = (CalculatePackingModule) SpringUtil
								.getBean(module);
					} catch (Exception e) {
						log.warn("Bean " + module
								+ " not defined in sm-core-beans.xml");
					}

					if (mod != null) {

						String infos = mod
								.getConfigurationOptionsFileName(super
										.getLocale());
						if (!StringUtils.isBlank(infos)) {
							pageInformation.put(module, infos);
						}
						if (response != null
								&& response
										.getMerchantConfiguration("SHP_PACK") != null) {
							PackageDetail shinfos = mod
									.getConfigurationOptions(
											response
													.getMerchantConfiguration("SHP_PACK"),
											ctx.getCurrency());
							if (shinfos != null) {
								boxInformation.put(module, shinfos);
							}
						}
					}
				}
			}

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String editPackingOption() {

		try {
			
			super.setPageTitle("label.shipping.packing.title");

			this.display();

			Context ctx = super.getContext();

			// get the module submited
			if (service == null) {
				log.error("Service information not submited");
				super.setTechnicalMessage();
				return ERROR;
			}

			// validate submited information
			String module = service.getCoreModuleName();
			CalculatePackingModule mod = null;

			try {
				mod = (CalculatePackingModule) SpringUtil.getBean(module);
			} catch (Exception e) {
				log.error("Module " + module + " not defined");
				super.setTechnicalMessage();
				return ERROR;
			}

			// validate - save
			try {
				mod.storeConfiguration(ctx.getMerchantid(), null, super
						.getServletRequest());
			} catch (ValidationException e) {
				MessageUtil.addErrorMessage(super.getServletRequest(), e
						.getMessage());
				return ERROR;
			}

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	public Map getBoxInformation() {
		return boxInformation;
	}

	public void setBoxInformation(Map boxInformation) {
		this.boxInformation = boxInformation;
	}

	public Collection<CoreModuleService> getServices() {
		return services;
	}

	public void setServices(Collection<CoreModuleService> services) {
		this.services = services;
	}

	public String getModuleSelected() {
		return moduleSelected;
	}

	public void setModuleSelected(String moduleSelected) {
		this.moduleSelected = moduleSelected;
	}

	public CoreModuleService getService() {
		return service;
	}

	public void setService(CoreModuleService service) {
		this.service = service;
	}

	public Map getPageInformation() {
		return pageInformation;
	}

	public void setPageInformation(Map pageInformation) {
		this.pageInformation = pageInformation;
	}

	public PackageDetail getShippingInformation() {
		return shippingInformation;
	}

	public void setShippingInformation(PackageDetail shippingInformation) {
		this.shippingInformation = shippingInformation;
	}

	public String getSizeUnit() {
		return sizeUnit;
	}

	public void setSizeUnit(String sizeUnit) {
		this.sizeUnit = sizeUnit;
	}

	public String getWeightUnit() {
		return weightUnit;
	}

	public void setWeightUnit(String weightUnit) {
		this.weightUnit = weightUnit;
	}

}



```
