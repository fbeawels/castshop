# StoreFrontContentAction.java

## Review

## 1. Summary  

**Purpose & Scope**  
`StoreFrontContentAction` is a Struts‑style action that manages *dynamic labels* – i.e. custom page‑content blocks – for the storefront of a merchant. The action provides the CRUD operations that a merchant (or an admin on behalf of the merchant) uses to list, create/update, view details, and delete such content blocks.  

**Key Components**  
| Component | Responsibility |
|-----------|----------------|
| `displayList()` | Load all available content blocks for a given merchant, including the built‑in sections (`product`, `category`, etc.) and any template‑specific custom sections. |
| `saveList()` | Persist the *visibility* flags for each content block. |
| `displayDetails()` | Show the editable form for a single content block, pre‑populating all language variants. |
| `save()` | Persist a new or edited content block (title + per‑language description). |
| `delete()` | Remove a content block that belongs to the current merchant. |
| `prepareContentList()` | Populate the list of sections (IDs → human‑readable labels) used throughout the action. |
| `getTemplateSectionIds()` | Read store‑template specific configuration to determine any custom content sections. |
| Getters/Setters | Expose `sectionId` and the `pageContentList` map to the view layer. |

**Design & Patterns**  
* **Service Locator** – the action obtains `ReferenceService` and `MerchantService` via a `ServiceFactory`.  
* **Command/Action** – each public method is a Struts action command returning a navigation string (`SUCCESS`, `INPUT`).  
* **Data‑Access Object** – entities such as `DynamicLabel` are retrieved and persisted via the service layer.  
* **Locale/Label Utility** – `LabelUtil` is used for i18n look‑ups of section names.  

The code uses standard Java collections, Log4j for logging, and Apache Commons Lang (`StringUtils`) for string utilities.

---

## 2. Detailed Description  

### Execution Flow  

1. **Initialization** – The action is created per request (Struts typically does this).  
2. **`displayList()`**  
   * Sets the page title and prepares the language list.  
   * Builds a list of section IDs (the five built‑in sections + any custom sections from the template config).  
   * Calls `ReferenceService.getDynamicLabels(...)` to fetch all labels for those sections.  
   * Results are stored in the inherited `pages` collection (assumed to be defined in `ContentAction`).  
3. **`displayDetails()`**  
   * Prepares the page title, languages, and the content list map.  
   * Loads the `DynamicLabel` instance for the requested ID.  
   * Builds a per‑language title/description list for the UI.  
4. **`save()`** – Handles both create & update.  
   * Validates the title; builds a `DynamicLabel` instance (or updates the existing one).  
   * Iterates over all supported languages, creating a `DynamicLabelDescription` for each.  
   * Persists the entity via `ReferenceService.saveOrUpdateDynamicLabel(...)`.  
5. **`saveList()`** – Updates the *visible* flag for each label.  
   * Iterates over all labels, compares each label’s ID against the list of selected IDs (`getVisible()`), and sets `visible` accordingly.  
   * Persists the collection via `ReferenceService.saveDynamicLabel(...)`.  
6. **`delete()`** – Retrieves the label, checks that it belongs to the current merchant, then deletes it.  
7. **`prepareContentList()`** – Called by several actions to build `pageContentList` – a map of section ID → label. It merges the default sections and any custom ones read from the store’s module configuration.  

### Assumptions & Constraints  

* The `MerchantService`/`ReferenceService` are available via the `ServiceFactory` – the code assumes a correctly configured service locator.  
* The current request context (`super.getContext()`) provides a merchant ID and locale.  
* `LabelConstants` contains integer IDs for each built‑in section; the code assumes those constants are stable and unique.  
* The view layer expects the `pages`, `titles`, `descriptions`, and `pageContentList` collections to be populated.  
* No explicit permission checks beyond merchant ID equality are performed – the surrounding framework is expected to enforce security.  

### Architecture & Design Choices  

* **Separation of Concerns** – The action delegates persistence to services, keeping business logic out of the controller.  
* **Hard‑coded Section IDs** – The built‑in section IDs are magic numbers; they are defined as constants but the code still hard‑codes the numeric values in several places.  
* **Locale Handling** – `LabelUtil` is used extensively; the action sets the locale on the util before fetching translated labels.  
* **Dynamic Label Pattern** – The use of `DynamicLabel` + `DynamicLabelDescription` is a classic i18n content‑management approach.  
* **Custom Sections** – By reading a configuration map and looking up the keys that start with `"content-"`, the code dynamically adds custom content sections for a given storefront template.  

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `Map getTemplateSectionIds()` | Reads the store template configuration to build a map of *custom section ID* → *label* for the UI. | None (uses current context’s merchant) | `Map<Integer,String>` | Logs errors; no state change except returned map. |
| `String displayList()` | Loads all dynamic labels for the current merchant and prepares the UI list. | None | `SUCCESS` (Struts result) | Fills `pages` collection; sets page title; sets technical error on exception. |
| `String saveList()` | Persists visibility flags for each label based on the submitted form. | None | `SUCCESS` | Updates `visible` on each `DynamicLabel`; calls `ReferenceService.saveDynamicLabel`. |
| `String displayDetails()` | Shows the edit form for a single content block, pre‑populating per‑language data. | None | `SUCCESS` | Fills `titles`, `descriptions`, sets page title. |
| `void prepareContentList()` | Builds `pageContentList` – a map of section ID → label – for use in forms. | None | None | Populates `pageContentList` field. |
| `String save()` | Creates or updates a `DynamicLabel` entity with its descriptions. | None | `SUCCESS` or `INPUT` | Persists via `ReferenceService.saveOrUpdateDynamicLabel`; sets success/technical messages. |
| `String delete()` | Deletes a content block that belongs to the current merchant. | None | `SUCCESS` | Calls `ReferenceService.deleteDynamicLabel`; repopulates list. |
| `int getSectionId()` / `void setSectionId(int)` | Get/set the current section ID. | None / int | int | None |
| `Map getPageContentList()` / `void setPageContentList(Map)` | Expose the content list map to the view. | None / Map | Map | None |

**Reusable/Utility Methods**  
* `prepareLanguages()` – inherited; prepares the list of supported languages.  
* `getText(String)` – Struts localization helper.  
* `addFieldError(String,String)` – used for validation errors.  

---

## 4. Dependencies  

| Library / Class | Purpose | Standard / Third‑Party |
|-----------------|---------|------------------------|
| `org.apache.commons.lang.StringUtils` | String utilities (`isBlank`) | Third‑Party |
| `org.apache.log4j.Logger` | Logging | Third‑Party |
| `ServiceFactory` | Service locator for `ReferenceService` / `MerchantService` | Third‑Party (application‑specific) |
| `ReferenceService`, `MerchantService` | CRUD for dynamic labels and merchant data | Third‑Party (application‑specific) |
| `DynamicLabel`, `DynamicLabelDescription`, `DynamicLabelDescriptionId` | Entity model for label content | Third‑Party |
| `MerchantStore`, `CategoryDescription`, `CategoryDescriptionId` | Domain entities (not directly used but imported) | Third‑Party |
| `LabelUtil` | I18n lookup helper | Third‑Party |
| `LabelConstants` | Constant section IDs | Third‑Party |
| `BaseAction` | Parent Struts action class providing context, messages, etc. | Third‑Party |
| `java.util` collections | Core Java utilities | Standard |

No platform‑specific APIs are used; the code is portable to any Java EE container that provides the referenced services.

---

## 5. Additional Notes  

### Code‑Quality & Maintainability  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (`Map`, `List`, `Set`) | Compile‑time safety lost; potential `ClassCastException`. | Use generics (`Map<Integer,String>`, `List<DynamicLabel>`, etc.). |
| **Exception swallowing** (`catch (Exception e) { log.error(e); ... }`) | Hard to diagnose root cause; user may see vague error. | Log with a clear message, rethrow as a runtime exception or set a specific error code/message. |
| **Duplicated code** (building `pageContentList` in several places) | Hard to maintain if logic changes. | Extract into a private method or a helper component. |
| **Magic numbers / strings** (`LabelConstants.STORE_FRONT_CUSTOM_CONTENT_PRODUCT`, `"content-"`, `"false"`) | Error‑prone; changes require code edits. | Keep all constants in a dedicated class; use enums for section IDs. |
| **Unnecessary object conversions** (`Map customContents = getTemplateSectionIds(); ids.addAll(customContents.keySet());`) | Potential `ClassCastException` if keys are not `Integer`. | Declare `customContents` with generic type and ensure the key type. |
| **Potential null dereference** (`this.getLabel().getTitle()` when `label` could be null) | Crash on null. | Guard against null or provide a default. |
| **Logging** (`log.error(e)` without context) | Hard to trace origin. | Use `log.error("message", e);` to include stack trace and context. |
| **Hard‑coded visibility logic** (`if (sId.equals("false")) { dl.setVisible(false); } else { dl.setVisible(true); }`) | Unclear intent; might set incorrect visibility. | Pass a boolean array or use a map of ID → visible flag. |
| **No validation** beyond title check; missing checks for duplicate titles, empty descriptions, etc. | Poor user experience. | Add Struts validation or programmatic checks. |
| **Security** – only merchant ID is checked in `delete()`. If other privileged actions can be performed by non‑merchant users, consider adding role checks. | Potential privilege escalation. | Add role/permission checks at the service layer. |

### Edge Cases & Failure Modes  

* **Store with no template configuration** – `getTemplateSectionIds()` will return an empty map; list actions still function but no custom sections are shown.  
* **Concurrent modifications** – two admins editing the same label concurrently may overwrite changes because `saveList()` reads the full collection and writes it back without optimistic locking.  
* **Large number of sections** – the code builds lists and maps on every request; could be cached per merchant to reduce load.  
* **Missing i18n keys** – `LabelUtil.getText()` may return null or the key itself; the UI may display a blank label.  

### Future Enhancements  

1. **Refactor to a Service Layer** – Move business logic from the action into a dedicated `StoreFrontContentService`.  
2. **Use Dependency Injection** – Replace the `ServiceFactory` locator with constructor/setter injection (Spring, CDI).  
3. **Unit Tests** – Add JUnit tests for each action method, mocking the services.  
4. **Optimistic Locking** – Add a version field to `DynamicLabel` to prevent lost updates.  
5. **Bulk Visibility Update** – Instead of iterating through all labels, accept a map of ID → visibility flag.  
6. **Internationalization** – Externalize all hard‑coded strings, including the `"content-"` prefix.  
7. **Caching** – Cache the static section list per merchant/template to avoid repeated configuration lookups.  
8. **API for Content Sections** – Expose a REST endpoint to fetch/edit content blocks, enabling a decoupled front‑end.  

---

### Overall Assessment  

The class implements the required CRUD functionality for dynamic storefront content. It is tightly coupled to the application’s service layer and uses standard Java EE patterns. The biggest improvement opportunities lie in **type safety**, **code reuse**, **error handling**, and **separation of concerns**. Addressing these will make the codebase more robust, easier to maintain, and more testable.

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
import java.util.Collection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.entity.catalog.CategoryDescription;
import com.salesmanager.core.entity.catalog.CategoryDescriptionId;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.entity.reference.DynamicLabelDescriptionId;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.LabelUtil;

public class StoreFrontContentAction extends ContentAction {

	private static final long serialVersionUID = 4033353809089229393L;

	private Logger log = Logger.getLogger(StoreFrontContentAction.class);



	private int sectionId = LabelConstants.STORE_FRONT_CUSTOM_CONTENT_PRODUCT;

	private Map pageContentList = new HashMap();
	
	
	private Map getTemplateSectionIds() {
		
		Map customIds = new HashMap();
		
		
		try {
			
			ReferenceService rservice = (ReferenceService) ServiceFactory
			.getService(ServiceFactory.ReferenceService);
			
			MerchantService mservice = (MerchantService) ServiceFactory
			.getService(ServiceFactory.MerchantService);
			
			MerchantStore store = mservice.getMerchantStore(super.getContext().getMerchantid());
			
			//Map storeConfigurations = (Map)super.getServletRequest().getSession().getAttribute("STORECONFIGURATION");
			Map storeConfigurations = rservice.getModuleConfigurationsKeyValue(
					store.getTemplateModule(), store.getCountry());
			
			
			
			if(storeConfigurations!=null) {
				
				LabelUtil labelUtil = LabelUtil.getInstance();
				labelUtil.setLocale(super.getLocale());
				
				for(Object o : storeConfigurations.keySet()) {
					String key = (String)o;
					if(key.startsWith("content-")) {
						String value = (String)storeConfigurations.get(key);
						//label is in store front template per module
						String l = labelUtil.getText(store.getTemplateModule() + ".text.position." + value);
						if(!StringUtils.isBlank(l)) {
							
							try {
								int id = Integer.parseInt(value);
								customIds.put(id, l);
							} catch (Exception e) {
								log.error("Cannot parse position for template content key " + key);
							}
						}
					}
				}
			}
			
			
		} catch (Exception e) {
			log.error(e);
		}
		
		return customIds;
		
	}

	/**
	 * Retreives Dynamic labels for a given section id and a merchant is
	 * 
	 * @return
	 */
	public String displayList() {

		try {
			
			super.setPageTitle("label.storefront.contentpagelist");

			super.prepareLanguages();
			
			if(label==null) {
				label = new DynamicLabel();
				label.setSectionId(sectionId);//assign a default section
			}
			
			List ids = new ArrayList();
			ids.add(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_PRODUCT);
			ids.add(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_CATEGORY);
			ids.add(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_CHECKOUT);
			ids.add(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_THANKYOU);
			ids.add(LabelConstants.FB_PAGE);
			
			Map customContents = getTemplateSectionIds();
			ids.addAll(customContents.keySet());

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			// get all pages
			pages = rservice.getDynamicLabels(super.getContext()
					.getMerchantid(), ids, super.getLocale());


		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
		}

		return SUCCESS;
	}

	public String saveList() {

		try {
			


			// get all
			super.prepareLanguages();
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			
			List ids = new ArrayList();
			ids.add(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_PRODUCT);
			ids.add(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_CATEGORY);
			ids.add(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_CHECKOUT);
			ids.add(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_THANKYOU);
			ids.add(LabelConstants.FB_PAGE);
			
			Map customContents = getTemplateSectionIds();
			ids.addAll(customContents.keySet());
			
			// get all pages
			Collection<DynamicLabel> labels = rservice.getDynamicLabels(super
					.getContext().getMerchantid(), ids, super
					.getLocale());

			if (labels != null && labels.size()>0) {

				for (Object o : labels) {

					DynamicLabel dl = (DynamicLabel) o;

					String[] labelIds = this.getVisible();

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

			displayList();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			displayList();
		}

		return SUCCESS;

	}

	public String displayDetails() {

		try {
			
			super.setPageTitle("label.storefront.contentpagedetails");

			super.prepareLanguages();
			
			
			prepareContentList();

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
							titles.add(description.getDynamicLabelTitle());
							descriptions.add(description
									.getDynamicLabelDescription());
						}
					}
				}

			}

		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
		}

		return SUCCESS;
	}
	
	private void prepareContentList() {
		
		
		//get section list
		/**
		   Section list are
		   STORE_FRONT_CUSTOM_CONTENT_PRODUCT = 71;//bottom of the product page
		   STORE_FRONT_CUSTOM_CONTENT_CATEGORY = 72;//bottom of the category page
           STORE_FRONT_CUSTOM_CONTENT_CHECKOUT = 73;//not implemented
           STORE_FRONT_CUSTOM_CONTENT_THANKYOU = 74;//not implemented
           
           and sections specific to the html store front template
		 */
		
		LabelUtil labelUtil = LabelUtil.getInstance();
		labelUtil.setLocale(super.getLocale());
		
		
		pageContentList.put(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_PRODUCT, labelUtil.getText("label.merchantstore.position.bottom.product"));
		pageContentList.put(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_CATEGORY, labelUtil.getText("label.merchantstore.position.bottom.category"));
		//pageContentList.put(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_CHECKOUT, labelUtil.getText("label.merchantstore.position.checkout"));
		pageContentList.put(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_THANKYOU, labelUtil.getText("label.merchantstore.position.thankyou"));
		pageContentList.put(LabelConstants.STORE_FRONT_FB_PORTLET, labelUtil
				.getText(super.getLocale(),
						"label.portlet.fb"));
		
		
		Map customContents = getTemplateSectionIds();
		
		pageContentList.putAll(customContents);
		
		
	}

	public String save() {

		try {
			
			super.setPageTitle("label.storefront.contentpagedetails");

			boolean hasError = false;

			super.prepareLanguages();
			
			prepareContentList();
			
			//get section list
			/**
			   Section list are
			   STORE_FRONT_CUSTOM_CONTENT_PRODUCT = 71;//bottom of the product page
			   STORE_FRONT_CUSTOM_CONTENT_CATEGORY = 72;//bottom of the category page
	           STORE_FRONT_CUSTOM_CONTENT_CHECKOUT = 73;//not implemented
	           STORE_FRONT_CUSTOM_CONTENT_THANKYOU = 74;//not implemented
	           
	           and sections specific to the html store front template
			 */
			
			LabelUtil labelUtil = LabelUtil.getInstance();
			labelUtil.setLocale(super.getLocale());
			
			
			pageContentList.put(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_PRODUCT, labelUtil.getText("label.merchantstore.position.bottom.product"));
			pageContentList.put(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_CATEGORY, labelUtil.getText("label.merchantstore.position.bottom.category"));
			//pageContentList.put(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_CHECKOUT, labelUtil.getText("label.merchantstore.position.checkout"));
			pageContentList.put(LabelConstants.STORE_FRONT_CUSTOM_CONTENT_THANKYOU, labelUtil.getText("label.merchantstore.position.thankyou"));
			
			Map customContents = getTemplateSectionIds();
			
			pageContentList.putAll(customContents);
			
			
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);

			//should never happen
			if (label == null) {
				label = new DynamicLabel();
			}
			
			if (StringUtils.isBlank(this.getLabel().getTitle())) {
				super.addFieldError("title",
						getText("label.storefront.contentpageid"));
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
				dldescription.setDynamicLabelTitle("--");


				Set descs = label.getDescriptions();
				if (descs == null) {
					descs = new HashSet();
				}

				descs.add(dldescription);

				label.setMerchantId(super.getContext().getMerchantid());
				//label.setSectionId(LabelConstants.STORE_FRONT_CUSTOM_PAGES);
				label.setDescriptions(descs);

			}

			if (hasError) {
				return INPUT;
			}

			rservice.saveOrUpdateDynamicLabel(label);

			super.setSuccessMessage();

		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
			return INPUT;
		}

		return SUCCESS;

	}

	public String delete() {

		try {

			super.prepareLanguages();
			
			prepareContentList();
			
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			DynamicLabel l = rservice.getDynamicLabel(this.getLabel()
					.getDynamicLabelId());
			if (l != null) {
				if (l.getMerchantId() == super.getContext().getMerchantid()) {
					rservice.deleteDynamicLabel(l);
				}
			}

			this.displayList();
			super.setSuccessMessage();

		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
			this.displayList();
		}

		return SUCCESS;

	}
	
	public int getSectionId() {
		return sectionId;
	}

	public void setSectionId(int sectionId) {
		this.sectionId = sectionId;
	}

	public Map getPageContentList() {
		return pageContentList;
	}

	public void setPageContentList(Map pageContentList) {
		this.pageContentList = pageContentList;
	}




}



```
