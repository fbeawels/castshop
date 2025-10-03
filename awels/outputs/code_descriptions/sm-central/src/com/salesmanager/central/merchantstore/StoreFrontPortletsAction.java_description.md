# StoreFrontPortletsAction.java

## Review

## 1. Summary
`StoreFrontPortletsAction` is a Struts‑style action class that manages the configuration of “portlets” (small UI widgets) on a merchant’s storefront.  
It interacts with a **ReferenceService** for retrieving available portlet modules and custom‑portlet labels, and with a **MerchantService** for persisting merchant‑specific configuration. The class supports three main flows:

| Flow | Purpose |
|------|---------|
| `display()` | Load the list of available portlets, the currently selected ones, and any custom portlet definitions. |
| `save()` | Persist the merchant’s selected core portlets. |
| `savePortlet()`, `saveCustomPortlets()`, `deleteCustomPortlet()` | Manage the creation, update, and deletion of custom portlet labels. |

Design patterns: the **Factory** pattern (via `ServiceFactory`) is used to obtain service instances. The class is a thin controller delegating business logic to services, keeping the action layer mostly concerned with request/response handling.

---

## 2. Detailed Description
### Core Components
| Component | Role |
|-----------|------|
| `ReferenceService` | Provides access to core modules (portlet definitions) and dynamic labels (custom portlets). |
| `MerchantService` | Handles persistence of merchant configuration objects (`MerchantConfiguration`). |
| `LabelUtil` | Supplies localized strings for UI labels (e.g., portlet positions). |
| `BaseAction` | Provides common Struts helpers (`setPageTitle`, `setSuccessMessage`, `setTechnicalMessage`, `prepareLanguages`, etc.). |

### Execution Flow
1. **Initialization**  
   The action is instantiated by the Struts dispatcher; dependencies are lazily loaded via `ServiceFactory`.

2. **`display()`**  
   - Sets the page title.  
   - Loads all core modules belonging to the `STORE_FRONT_PORTLETS_CODE`.  
   - Reverses the list (likely to display in a particular order).  
   - Retrieves the merchant’s current `STORE_PORTLETS_` configuration.  
   - Builds a `Map<String,String>` (`selectedPortlets`) of portlet codes that are currently enabled.  
   - Loads any custom portlet labels defined for the merchant.  
   - Stores all results in action properties for rendering.

3. **`customPortletsDetails()`**  
   Prepares a single `DynamicLabel` instance for editing:  
   - Loads the label, then its descriptions for all merchant languages.  
   - Fills `descriptions` list in UI order.

4. **`preparePortletsPositions()`**  
   Builds a `portletsPositions` map of position constants to localized labels (left, right, bottom landing).

5. **`savePortlet()`**  
   Persists a single custom portlet label.  
   - Validates required fields (sortOrder, title).  
   - Creates or updates `DynamicLabelDescription` objects for every merchant language.  
   - Calls `ReferenceService.saveOrUpdateDynamicLabel`.

6. **`saveCustomPortlets()`**  
   Toggles visibility of existing custom portlets based on the user’s selection.  
   - Iterates over all labels, matching them against the selected IDs.  
   - Calls `ReferenceService.saveDynamicLabel` to persist visibility changes.

7. **`deleteCustomPortlet()`**  
   Deletes a single custom portlet label if it belongs to the current merchant.

8. **`save()`**  
   Persists the list of selected core portlets.  
   - Builds a semicolon‑separated string from the `selection` array.  
   - Updates or creates a `MerchantConfiguration` record.

### Assumptions & Constraints
- `reflanguages` and `getText()` are provided by `BaseAction`.  
- The merchant ID is obtained via `super.getContext().getMerchantid()`.  
- The action relies on the existence of a `MerchantConfiguration` with key `STORE_PORTLETS_`.  
- UI rendering expects raw collections/maps (no generics) – a legacy constraint.  

---

## 3. Functions/Methods
| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `display()` | Load data for the main portlet configuration screen. | None (uses request context) | `SUCCESS` or `INPUT` | Populates `portlets`, `selectedPortlets`, `configuredPortlets`, `customPortlets`. |
| `customPortletsDetails()` | Prepare a single custom portlet for editing. | None | `SUCCESS` | Sets `label`, `descriptions`. |
| `preparePortletsPositions()` | Build a UI map of portlet positions. | None | None | Sets `portletsPositions`. |
| `savePortlet()` | Persist or update a custom portlet label. | `label`, `descriptions` from UI | `SUCCESS` or `INPUT` | Writes to DB via `ReferenceService`. |
| `saveCustomPortlets()` | Persist visibility of custom portlets. | `selectionCustomPortlets` | `SUCCESS` or `INPUT` | Updates DB. |
| `deleteCustomPortlet()` | Delete a custom portlet label. | `label` (must have ID) | `SUCCESS` | Removes from DB. |
| `save()` | Persist selected core portlets. | `selection` array | `SUCCESS` or `INPUT` | Updates/creates `MerchantConfiguration`. |
| Getter/Setter pairs | Standard JavaBean accessors for action properties. | — | — | — |

Utility methods are provided by `ServiceFactory`, `LabelUtil`, and `MerchantConfigurationUtil` (for building/parsing configuration strings).

---

## 4. Dependencies
| Library | Type | Usage |
|---------|------|-------|
| **Apache Commons Lang** (`org.apache.commons.lang.StringUtils`) | Third‑party | String validation. |
| **Apache Log4j** (`org.apache.log4j.Logger`) | Third‑party | Logging. |
| **SalesManager Core** | Third‑party | `ServiceFactory`, `ReferenceService`, `MerchantService`, entity classes (`MerchantConfiguration`, `DynamicLabel`, etc.), constants. |
| **SalesManager Central** | Internal | `BaseAction`, language utilities (`LabelUtil`). |
| **Java Collections Framework** | Standard | `List`, `Map`, `Set`, `Collection`. |
| **Java Date** (`java.util.Date`) | Standard | Timestamps for configuration records. |

Platform: Java EE (likely running in a servlet container, given the Struts‑style action).

---

## 5. Additional Notes & Recommendations
### 5.1. Null‑Pointer Risk
- **`configuredPortlets`** is never instantiated before it is used in `display()`.  
  ```java
  this.configuredPortlets.put(conf.getConfigurationModule(), conf.getConfigurationModule());
  ```  
  This will throw a `NullPointerException` unless `configuredPortlets` is set elsewhere.  
  **Fix**: initialize it in the constructor or before use:  
  ```java
  if (configuredPortlets == null) {
      configuredPortlets = new HashMap<>();
  }
  ```

- **`portletsPositions`** and other raw `Map`/`Collection` fields are never instantiated in the constructor.  
  **Fix**: create them in `preparePortletsPositions()` and wherever they are used.

### 5.2. Generics & Type Safety
- The code uses raw types (`Map`, `Collection`, `Set`) in many places.  
  **Impact**: compile‑time type checking is lost, potentially leading to runtime `ClassCastException`.  
  **Recommendation**: refactor to use generics, e.g. `Map<String, String>`, `Collection<CoreModuleService>`, `Set<DynamicLabelDescription>`.

### 5.3. Code Duplication & Flow
- `super.setSuccessMessage()` is called in multiple branches.  
  **Suggestion**: centralize success handling after the try/catch block.

### 5.4. Performance & Concurrency
- `Collections.reverse((List) portlets);` casts a collection to a `List` without checking.  
  **Risk**: if the underlying collection is not a list, this will throw an exception.  
  **Recommendation**: obtain a `List` directly (`new ArrayList<>(portlets)`) and then reverse it.

### 5.5. Business Logic
- The logic for toggling custom portlet visibility in `saveCustomPortlets()` is convoluted and error‑prone.  
  **Improvement**: build a `Set<Long>` of selected IDs once, then iterate over labels to set `visible` accordingly.

### 5.6. Internationalization
- The action relies heavily on `BaseAction.prepareLanguages()` and raw language maps (`reflanguages`).  
  **Consider**: use `List<Locale>` or `Map<Locale, String>` for clearer semantics.

### 5.7. Testing
- Unit tests should cover:
  - `display()` path with existing and non‑existing configurations.  
  - `savePortlet()` validation (missing title, sort order).  
  - `saveCustomPortlets()` when no selections are made.  
  - `deleteCustomPortlet()` edge cases (non‑existent label, foreign merchant).  

### 5.8. Documentation
- Javadoc comments are missing for public methods.  
  **Add**: method descriptions, parameter explanation, and potential exceptions.

### 5.9. Security
- No explicit security checks beyond merchant ID.  
  **Ensure**: that only the owning merchant can edit/delete its labels.

---

**Overall Assessment**  
The action fulfills its functional requirements but suffers from several code‑quality issues: potential null pointer exceptions, lack of generics, duplicated code, and unclear handling of collections. Addressing these will improve robustness, maintainability, and testability.

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
package com.salesmanager.central.merchantstore;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.Date;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.entity.reference.DynamicLabelDescriptionId;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MerchantConfigurationUtil;

public class StoreFrontPortletsAction extends BaseAction {

	private static Logger log = Logger
			.getLogger(StoreFrontPortletsAction.class);

	Collection<CoreModuleService> portlets;
	Map<String, String> selectedPortlets;
	Map<String, String> configuredPortlets;

	Collection customPortlets;

	String[] selection;// selected portlets
	String[] selectionCustomPortlets;// selection custom portlets

	private List<String> descriptions = new ArrayList<String>();

	private DynamicLabel label = null;

	private Map portletsPositions = null;

	private MerchantConfiguration mc = null;

	public MerchantConfiguration getMc() {
		return mc;
	}

	public void setMc(MerchantConfiguration mc) {
		this.mc = mc;
	}

	public String[] getSelection() {
		return selection;
	}

	public void setSelection(String[] selection) {
		this.selection = selection;
	}

	public Map<String, String> getSelectedPortlets() {
		return selectedPortlets;
	}

	public void setSelectedPortlets(Map<String, String> selectedPortlets) {
		this.selectedPortlets = selectedPortlets;
	}

	public Collection<CoreModuleService> getPortlets() {
		return portlets;
	}

	public void setPortlets(Collection<CoreModuleService> portlets) {
		this.portlets = portlets;
	}

	public String display() {

		try {
			
			super.setPageTitle("label.storefront.portletsconfig");

			// get portlets
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			portlets = rservice.getCoreModules(
					CatalogConstants.STORE_FRONT_PORTLETS_CODE, "XX");
			Collections.reverse((List) portlets);

			// get selection
			ConfigurationRequest request = new ConfigurationRequest(super
					.getContext().getMerchantid(), true,
					ConfigurationConstants.STORE_PORTLETS_);
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationResponse vo = mservice.getConfiguration(request);

			if (vo != null) {
				List configurations = vo.getMerchantConfigurationList();

				if (configurations != null && configurations.size() > 0) {

					Iterator i = configurations.iterator();

					while (i.hasNext()) {

						MerchantConfiguration conf = (MerchantConfiguration) i
								.next();

						if (conf.getConfigurationKey().equals(
								ConfigurationConstants.STORE_PORTLETS_)) {
							mc = conf;
							Collection portletsList = MerchantConfigurationUtil
									.getConfigurationList(mc
											.getConfigurationValue(), ";");
							if (portletsList != null && portletsList.size() > 0) {
								Map returnMap = new HashMap();
								Iterator ii = portletsList.iterator();
								while (ii.hasNext()) {
									String p = (String) ii.next();
									returnMap.put(p, p);
								}
								selectedPortlets = returnMap;
							}
							continue;
						}

						if (conf.getConfigurationModule() != null) {
							this.configuredPortlets.put(conf
									.getConfigurationModule(), conf
									.getConfigurationModule());
						}
					}

				}
			}

			// get custom portlets
			customPortlets = rservice.getDynamicLabels(super.getContext()
					.getMerchantid(),
					LabelConstants.STORE_FRONT_CUSTOM_PORTLETS, super
							.getLocale());

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String customPortletsDetails() {
		
		super.setPageTitle("label.storefront.portletsconfig");

		try {

			super.prepareLanguages();

			this.preparePortletsPositions();
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);

			if (label != null) {

				// get label

				label = rservice.getDynamicLabel(label.getDynamicLabelId());

				Set descriptionsSet = label.getDescriptions();

				Map descriptionsMap = new HashMap();

				if (descriptionsSet != null) {

					for (Object desc : descriptionsSet) {
						DynamicLabelDescription description = (DynamicLabelDescription) desc;
						descriptionsMap.put(
								description.getId().getLanguageId(),
								description);
					}

					// iterate through languages for appropriate order
					for (int count = 0; count < reflanguages.size(); count++) {
						int langid = (Integer) reflanguages.get(count);
						DynamicLabelDescription description = (DynamicLabelDescription) descriptionsMap
								.get(langid);
						if (description != null) {
							descriptions.add(description
									.getDynamicLabelDescription());
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

	private void preparePortletsPositions() {
		

		LabelUtil l = LabelUtil.getInstance();

		portletsPositions = new HashMap();
		portletsPositions.put(LabelConstants.LABEL_POSITION_LEFT, l.getText(
				super.getLocale(), "label.generic.position.left"));
		portletsPositions.put(LabelConstants.LABEL_POSITION_RIGHT, l.getText(
				super.getLocale(), "label.generic.position.right"));
		portletsPositions.put(LabelConstants.LABEL_POSITION_BOTTOM_LANDING, l
				.getText(super.getLocale(),
						"label.merchantstore.position.bottom.landing"));


	}

	public String savePortlet() {

		try {

			super.setPageTitle("label.storefront.portletsconfig");
			
			boolean hasError = false;

			super.prepareLanguages();

			this.preparePortletsPositions();

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);

			if (label == null) {
				label = new DynamicLabel();
			}
			
			if (this.label.getSortOrder()==null) {
				super.addFieldError("label.sortOrder",
						getText("invalid.fieldvalue.sortorder"));
				hasError = true;
			}

			if (StringUtils.isBlank(this.label.getTitle())) {
				super.addFieldError("label.title",
						getText("error.message.storefront.portletidrequired"));
				hasError = true;
			}

			Iterator i = reflanguages.keySet().iterator();
			while (i.hasNext()) {
				int langcount = (Integer) i.next();

				String description = (String) this.getDescriptions().get(
						langcount);

				int submitedlangid = (Integer) reflanguages.get(langcount);
				// create
				DynamicLabelDescriptionId id = new DynamicLabelDescriptionId();
				id.setLanguageId(submitedlangid);
				if (label != null) {
					id.setDynamicLabelId(label.getDynamicLabelId());
				}

				DynamicLabelDescription dldescription = new DynamicLabelDescription();
				dldescription.setId(id);
				dldescription.setDynamicLabelDescription(description);
				dldescription.setDynamicLabelTitle("-");

				Set descs = label.getDescriptions();
				if (descs == null) {
					descs = new HashSet();
				}

				descs.add(dldescription);

				label.setMerchantId(super.getContext().getMerchantid());
				label.setSectionId(LabelConstants.STORE_FRONT_CUSTOM_PORTLETS);
				label.setDescriptions(descs);

			}

			if (hasError) {
				return INPUT;
			}

			rservice.saveOrUpdateDynamicLabel(label);

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String saveCustomPortlets() {

		try {
			
			super.setPageTitle("label.storefront.portletsconfig");

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			Collection<DynamicLabel> labels = rservice.getDynamicLabels(super
					.getContext().getMerchantid(),
					LabelConstants.STORE_FRONT_CUSTOM_PORTLETS);

			if (labels != null && labels.size()>0) {

				for (Object o : labels) {

					DynamicLabel dl = (DynamicLabel) o;
					String[] labelIds = this.getSelectionCustomPortlets();

					if (labelIds != null && labelIds.length > 0) {

						boolean found = false;
						for (int i = 0; i < labelIds.length; i++) {
							String sId = labelIds[i];
							try {
								long id = Long.parseLong(sId);
								if (dl.getDynamicLabelId() == id) {
									found = true;
								}

							} catch (Exception e) {
								log.error("Wrong id " + sId);
								if (sId.equals("false")) {
									dl.setVisible(false);
								} else {
									dl.setVisible(true);
								}
							}

						}
						if (found == true) {
							dl.setVisible(true);
						} else {
							dl.setVisible(false);
						}

					} else {
						dl.setVisible(false);
					}
				}

				rservice.saveDynamicLabel(labels);
				super.setSuccessMessage();
			}

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}
		this.display();// prepare display elements
		return SUCCESS;

	}

	public String deleteCustomPortlet() {

		try {
			
			super.setPageTitle("label.storefront.portletsconfig");

			super.prepareLanguages();
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			DynamicLabel l = rservice.getDynamicLabel(this.getLabel()
					.getDynamicLabelId());
			if (l != null) {
				if (l.getMerchantId() == super.getContext().getMerchantid()) {
					rservice.deleteDynamicLabel(l);
				}
			}

			this.display();
			super.setSuccessMessage();

		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
			this.display();
		}

		return SUCCESS;

	}

	public String save() {

		try {
			
			super.setPageTitle("label.storefront.portletsconfig");

			this.display();
			// save selected protlets

			// get selection first
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			if (selection == null) {

				if (mc != null) {
					mservice.deleteMerchantConfiguration(mc);
					this.selectedPortlets = null;
					super.setSuccessMessage();
					return SUCCESS;
				} else {
					return SUCCESS;
				}
			}

			List l = Arrays.asList(selection);

			String line = MerchantConfigurationUtil.buildConfigurationLine(l,
					";");

			Collection portletsList = MerchantConfigurationUtil
					.getConfigurationList(line, ";");
			if (portletsList != null && portletsList.size() > 0) {
				Map returnMap = new HashMap();
				Iterator i = portletsList.iterator();
				while (i.hasNext()) {
					String p = (String) i.next();
					returnMap.put(p, p);
				}
				selectedPortlets = returnMap;
			}

			if (mc == null) {
				mc = new MerchantConfiguration();
				mc.setConfigurationKey(ConfigurationConstants.STORE_PORTLETS_);
				mc.setDateAdded(new Date());
				mc.setLastModified(new Date());
				mc.setMerchantId(super.getContext().getMerchantid());
			}

			mc.setConfigurationValue(line);
			mservice.saveOrUpdateMerchantConfiguration(mc);

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}

		return SUCCESS;

	}

	public Map<String, String> getConfiguredPortlets() {
		return configuredPortlets;
	}

	public void setConfiguredPortlets(Map<String, String> configuredPortlets) {
		this.configuredPortlets = configuredPortlets;
	}

	public Collection getCustomPortlets() {
		return customPortlets;
	}

	public void setCustomPortlets(Collection customPortlets) {
		this.customPortlets = customPortlets;
	}

	public String[] getSelectionCustomPortlets() {
		return selectionCustomPortlets;
	}

	public void setSelectionCustomPortlets(String[] selectionCustomPortlets) {
		this.selectionCustomPortlets = selectionCustomPortlets;
	}

	public DynamicLabel getLabel() {
		return label;
	}

	public void setLabel(DynamicLabel label) {
		this.label = label;
	}

	public List<String> getDescriptions() {
		return descriptions;
	}

	public void setDescriptions(List<String> descriptions) {
		this.descriptions = descriptions;
	}

	public Map getPortletsPositions() {
		return portletsPositions;
	}

	public void setPortletsPositions(Map portletsPositions) {
		this.portletsPositions = portletsPositions;
	}

}



```
