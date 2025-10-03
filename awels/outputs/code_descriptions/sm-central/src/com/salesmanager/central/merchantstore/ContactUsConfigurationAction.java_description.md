# ContactUsConfigurationAction.java

## Review

## 1. Summary
**Purpose & Functionality**  
`ContactUsConfigurationAction` is a Struts‑2 action class that manages the “Contact Us” configuration for a merchant storefront. It lets an administrator:

- **Display** current settings (map visibility, store info, custom HTML text).
- **Save** updated settings to the database.
- (Placeholder for) **Delete** configuration – currently unimplemented.

**Key Components**

| Component | Role |
|-----------|------|
| `MerchantService` | CRUD on merchant store and configurations. |
| `ReferenceService` | CRUD on `DynamicLabel` objects that hold multilingual HTML snippets. |
| `ConfigurationRequest / Response` | DTOs for retrieving configuration values. |
| `MerchantConfiguration` | Entity that stores boolean flags (`configurationValue`, `configurationValue1`). |
| `DynamicLabel` & `DynamicLabelDescription` | Hold localized custom HTML for the Contact Us page. |
| `reflanguages` (inherited from `BaseAction`) | Map of language indexes to language IDs used for multi‑language support. |

**Design Patterns & Frameworks**

- **Command / DTO** pattern via `ConfigurationRequest`/`Response`.  
- **Data Access Object** via service factories.  
- **MVC**: Action class (controller), services (model), and Struts result pages (view).  
- Uses **Apache Commons Lang** for `StringUtils`, **Log4j** for logging, and the **SalesManager** framework for business logic.

---

## 2. Detailed Description
### Initialization
- The action inherits from `BaseAction`, which provides helper methods (`prepareLanguages`, `setPageTitle`, etc.) and the context containing the current merchant ID.
- The constructor is implicit; no special initialization logic is required.

### Runtime Flow
1. **`display()`**  
   - Prepares language support and page title.  
   - Retrieves current configuration for the `CONTACTUS` key.  
   - Sets `showMap` and `showBasicStoreInformation` booleans based on config values.  
   - Loads the custom HTML label (`DynamicLabel`) for the Contact Us section and populates `contactUsDescription` with localized descriptions.

2. **`save()`**  
   - Prepares languages and page title.  
   - Reads the incoming `contactUsDescription` list (one entry per language).  
   - Builds a map of `DynamicLabelDescription` objects for each submitted language.  
   - Loads or creates a `MerchantConfiguration` for the `CONTACTUS` key and updates the flags from the action fields (`showMap`, `showBasicStoreInformation`).  
   - Persists the configuration.  
   - If any label descriptions were provided, updates the `DynamicLabel` (creates or updates via `ReferenceService`).  
   - Sets a success or technical error message.

3. **`delete()`**  
   - Stubbed; currently only sets the page title and returns `SUCCESS`. Intended to remove the configuration and label.

### Cleanup
No explicit cleanup is required; the services manage their own transactions and sessions.

### Assumptions & Constraints
- `reflanguages` is a `Map<Integer, Integer>` (index → language ID) provided by `BaseAction`.  
- Each index corresponds to a language that the storefront supports.  
- The `contactUsDescription` list is assumed to be of the same length as `reflanguages`.  
- The action is executed in a web context where the merchant ID is already set in the context.

---

## 3. Functions/Methods
| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `public String save()` | Persists current Contact Us configuration and label descriptions. | None (uses action fields). | `SUCCESS` string. | Writes to DB via `MerchantService` and `ReferenceService`; logs errors; sets success/technical messages. |
| `public String delete()` | Placeholder for removing configuration. | None. | `SUCCESS` or `INPUT`. | Currently only logs and sets page title. |
| `public String display()` | Loads existing configuration and label to display in UI. | None. | `SUCCESS` | Sets action fields, populates `contactUsDescription`. |
| `public MerchantStore getStore()` | Getter for the merchant store. | None. | `MerchantStore` | None |
| `public void setStore(MerchantStore store)` | Setter for merchant store. | `MerchantStore` | None |
| `public boolean isShowMap()` | Getter for map visibility flag. | None. | `boolean` | None |
| `public void setShowMap(boolean showMap)` | Setter for map visibility flag. | `boolean` | None |
| `public boolean isShowBasicStoreInformation()` | Getter for store info flag. | None. | `boolean` | None |
| `public void setShowBasicStoreInformation(boolean showBasicStoreInformation)` | Setter for store info flag. | `boolean` | None |
| `public List<String> getContactUsDescription()` | Getter for list of HTML snippets. | None | `List<String>` | None |
| `public void setContactUsDescription(List<String> contactUsDescription)` | Setter for list. | `List<String>` | None |
| `public DynamicLabel getLabel()` | Getter for the dynamic label. | None | `DynamicLabel` | None |
| `public void setLabel(DynamicLabel label)` | Setter for the label. | `DynamicLabel` | None |

### Reusable/Utility Methods
The action relies heavily on methods from `BaseAction` (`prepareLanguages`, `setPageTitle`, `setSuccessMessage`, `setTechnicalMessage`) and on service factories; no additional utilities are defined within this class.

---

## 4. Dependencies
| Library | Purpose | Standard/Third‑Party |
|---------|---------|----------------------|
| `org.apache.commons.lang.StringUtils` | String null/blank checks. | Third‑Party (Apache Commons Lang) |
| `org.apache.log4j.Logger` | Logging. | Third‑Party (Log4j) |
| `com.salesmanager.central.BaseAction` | Base Struts action with helpers and context. | Internal |
| `com.salesmanager.core.*` | Core domain entities (`MerchantConfiguration`, `MerchantStore`, etc.) and service interfaces. | Internal |
| `com.salesmanager.core.service.ServiceFactory` | Service locator / factory. | Internal |
| `com.salesmanager.core.service.merchant.*` | Merchant configuration CRUD. | Internal |
| `com.salesmanager.core.service.reference.ReferenceService` | Dynamic label CRUD. | Internal |
| `java.util.*` | Collections handling. | Standard |

No platform‑specific dependencies; the code should run on any Java EE container that provides the SalesManager services.

---

## 5. Additional Notes
### Strengths
- **Clear separation** of concerns: UI logic in the action, business logic in services.  
- **Multi‑language support** via `reflanguages` mapping and `DynamicLabelDescription`.  
- **Robust error handling** with try/catch and logging.

### Potential Issues / Edge Cases
1. **`reflanguages` Not Initialized**  
   - The code assumes `reflanguages` is populated before any method call. If it is null or empty, the loops will silently skip, potentially leading to incomplete data being processed.

2. **Index‑Language Mismatch**  
   - `contactUsDescription` is indexed by language ID (not the same as `reflanguages` index). If the list size mismatches the number of languages, `IndexOutOfBoundsException` could occur. Defensive checks are missing.

3. **`label` Initialization in `save()`**  
   - `label` is used without being instantiated if it was never set by `display()` or elsewhere. This can cause a `NullPointerException` when calling `label.setDescriptions(...)`. The code should create a new `DynamicLabel` when `label == null`.

4. **`delete()` Unimplemented**  
   - Leaving a method that silently returns `SUCCESS` can be misleading. It should either throw `UnsupportedOperationException` or be fully implemented.

5. **Hardcoded Boolean Strings**  
   - The configuration values are stored as `"true"`/`"false"` strings. A boolean field would be clearer and less error‑prone.

6. **Logging**  
   - The stack trace is not logged (`e` is logged only as the exception, but not `e.printStackTrace()` or a full stack trace). Using `log.error("message", e)` would give full context.

7. **Concurrency**  
   - The action is stateful; concurrent requests for the same merchant could interleave and corrupt the configuration if not properly isolated. Transaction boundaries in the services need to ensure consistency.

### Suggested Enhancements
- **Validation**: Verify that every language has a non‑blank description before persisting; otherwise, retain the old description.
- **Null‑Safe Handling**: Guard against `label == null` and create a new instance if necessary.
- **Complete Delete Implementation**: Remove the configuration and dynamic label, or at least document that it's not supported.
- **Use Booleans**: Store configuration flags as boolean values in the database and convert to/from strings only when necessary.
- **Unit Tests**: Add tests for each action method covering normal and edge scenarios (empty lists, missing labels, etc.).
- **Refactor**: Extract repeated logic (e.g., mapping descriptions) into private helper methods for clarity and reuse.

Overall, the class provides a solid foundation for managing Contact Us settings but would benefit from defensive programming and a few cleanup steps to improve reliability and maintainability.

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
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.entity.reference.DynamicLabelDescriptionId;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;

public class ContactUsConfigurationAction extends BaseAction {

	private Logger log = Logger.getLogger(ContactUsConfigurationAction.class);

	private MerchantStore store;

	private boolean showMap = false;

	private boolean showBasicStoreInformation = true;

	private List<String> contactUsDescription = new ArrayList<String>();// text
																		// submited

	private DynamicLabel label;

	public String save() {

		try {
			
			super.setPageTitle("label.storefront.contactus.setup");

			super.prepareLanguages();

			// retrieve current merchant configuration

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			ConfigurationRequest request = new ConfigurationRequest(super
					.getContext().getMerchantid(),
					ConfigurationConstants.CONTACTUS);
			ConfigurationResponse response = mservice.getConfiguration(request);

			Map descriptionsmap = new HashMap();
			if (this.getContactUsDescription() != null
					&& this.getContactUsDescription().size() > 0) {

				Iterator i = reflanguages.keySet().iterator();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					String description = (String) this
							.getContactUsDescription().get(langcount);
					if (StringUtils.isBlank(description)) {
						continue;
					}

					int submitedlangid = (Integer) reflanguages.get(langcount);

					DynamicLabelDescription desc = new DynamicLabelDescription();
					DynamicLabelDescriptionId id = new DynamicLabelDescriptionId();
					id.setLanguageId(submitedlangid);
					desc.setId(id);
					desc.setDynamicLabelDescription(description);
					desc.setDynamicLabelTitle(" ");
					descriptionsmap.put(submitedlangid, desc);
				}
			}

			MerchantConfiguration conf = response
					.getMerchantConfiguration(ConfigurationConstants.CONTACTUS);

			if (conf != null) {
				conf = response
						.getMerchantConfiguration(ConfigurationConstants.CONTACTUS);
				if (this.isShowMap()) {
					conf.setConfigurationValue1("true");
				} else {
					conf.setConfigurationValue1("false");
				}
				if (this.isShowBasicStoreInformation()) {
					conf.setConfigurationValue("true");
				} else {
					conf.setConfigurationValue("false");
				}
			} else {
				if (this.isShowMap() || this.isShowBasicStoreInformation()) {
					conf = new MerchantConfiguration();
					conf.setConfigurationKey(ConfigurationConstants.CONTACTUS);
					conf.setMerchantId(super.getContext().getMerchantid());
					if (this.isShowMap()) {
						conf.setConfigurationValue1("true");
					} else {
						conf.setConfigurationValue1("false");
					}
					if (this.isShowBasicStoreInformation()) {
						conf.setConfigurationValue("true");
					} else {
						conf.setConfigurationValue("false");
					}
				}
			}

			if (conf != null) {
				mservice.saveOrUpdateMerchantConfiguration(conf);
			}

			if (descriptionsmap.size() > 0) {
				Set set = new HashSet();
				set.addAll(descriptionsmap.values());
				label.setDescriptions(set);
				label.setMerchantId(super.getContext().getMerchantid());
				label.setVisible(true);
				label.setSectionId(LabelConstants.STORE_FRONT_CONTACT_US);
				ReferenceService rservice = (ReferenceService) ServiceFactory
						.getService(ServiceFactory.ReferenceService);
				rservice.saveOrUpdateDynamicLabel(label);
			}

			super.setSuccessMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	/**
	 * not implemented
	 * 
	 * @return
	 */
	public String delete() {

		try {
			
			super.setPageTitle("label.storefront.contactus.setup");

			DynamicLabel label = this.getLabel();
			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}

	}

	public String display() {

		try {

			super.prepareLanguages();
			
			super.setPageTitle("label.storefront.contactus.setup");

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			store = mservice.getMerchantStore(super.getContext()
					.getMerchantid());

			ConfigurationRequest request = new ConfigurationRequest(super
					.getContext().getMerchantid(),
					ConfigurationConstants.CONTACTUS);
			ConfigurationResponse response = mservice.getConfiguration(request);

			MerchantConfiguration conf = response
					.getMerchantConfiguration(ConfigurationConstants.CONTACTUS);

			if (conf != null) {

				// display google map
				String mapConf = conf.getConfigurationValue1();
				if (mapConf != null && mapConf.equalsIgnoreCase("true")) {
					this.setShowMap(true);
				}

				// display custom address
				String basicConf = conf.getConfigurationValue();
				if (basicConf != null && basicConf.equalsIgnoreCase("false")) {
					this.setShowBasicStoreInformation(false);
				}

			}

			// custom html text
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			List labels = (List) rservice.getDynamicLabels(super.getContext()
					.getMerchantid(), LabelConstants.STORE_FRONT_CONTACT_US);

			if (labels != null && labels.size() > 0) {
				label = (DynamicLabel) labels.get(0);
			}

			if (label != null) {
				Map labelMap = new HashMap();
				Set labelSet = label.getDescriptions();
				Iterator it = labelSet.iterator();
				while (it.hasNext()) {
					DynamicLabelDescription description = (DynamicLabelDescription) it
							.next();
					labelMap.put(description.getId().getLanguageId(),
							description);
				}

				for (int icount = 0; icount < reflanguages.size(); icount++) {
					int langid = (Integer) reflanguages.get(icount);
					DynamicLabelDescription desc = (DynamicLabelDescription) labelMap
							.get(langid);
					if (desc != null) {
						contactUsDescription.add(desc
								.getDynamicLabelDescription());
					}
				}
			}

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public MerchantStore getStore() {
		return store;
	}

	public void setStore(MerchantStore store) {
		this.store = store;
	}

	public boolean isShowMap() {
		return showMap;
	}

	public void setShowMap(boolean showMap) {
		this.showMap = showMap;
	}

	public boolean isShowBasicStoreInformation() {
		return showBasicStoreInformation;
	}

	public void setShowBasicStoreInformation(boolean showBasicStoreInformation) {
		this.showBasicStoreInformation = showBasicStoreInformation;
	}

	public List<String> getContactUsDescription() {
		return contactUsDescription;
	}

	public void setContactUsDescription(List<String> contactUsDescription) {
		this.contactUsDescription = contactUsDescription;
	}

	public DynamicLabel getLabel() {
		return label;
	}

	public void setLabel(DynamicLabel label) {
		this.label = label;
	}

}



```
