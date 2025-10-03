# StoreFrontPagesAction.java

## Review

## 1. Summary  

`StoreFrontPagesAction` is a Struts‑style action that lets a merchant manage **custom “storefront” pages** (FAQ, Shipping policy, etc.).  
It:

| Feature | What it does |
|---------|--------------|
| **Display list** (`displayList`) | Loads all dynamic labels belonging to the “storefront custom pages” section for the current merchant and locale. |
| **Save list** (`saveList`) | Persists the *visible* flag for every page, based on the list of selected IDs. |
| **Display details** (`displayDetails`) | Loads a single label and its descriptions for all merchant‑available languages, ordering them per the language list. |
| **Save** (`save`) | Creates/updates a label and its language‑specific descriptions (title, body, SEF‑URL). |
| **Delete** (`delete`) | Removes a label if it belongs to the current merchant. |

The class relies on a `ReferenceService` (obtained via a `ServiceFactory`) to interact with the persistence layer.  It extends `ContentAction`, which supplies helpers such as `setPageTitle`, `prepareLanguages`, and message handling (`setSuccessMessage`, `setTechnicalMessage`, etc.).  No external frameworks beyond Apache Commons Lang and Log4j are used.

---

## 2. Detailed Description  

### Execution Flow  

1. **Initialization** – Struts injects the action into the request, populating fields such as `label`, `visible`, `titles`, `descriptions`, `sefurl`, and `reflanguages` (the list of language IDs the merchant supports).  
2. **Runtime behavior** – Depending on the user action (list, edit, delete) the appropriate method is invoked.  
3. **Cleanup** – No explicit resource cleanup is required; the action is short‑lived per request.  

### Core Interaction Points  

| Method | Purpose | Key Interactions |
|--------|---------|------------------|
| `displayList()` | Retrieve all labels for the merchant & section | `ReferenceService.getDynamicLabels(...)` |
| `saveList()` | Persist visibility flags | `ReferenceService.saveDynamicLabel(labels)` |
| `displayDetails()` | Prepare data for the edit form | `ReferenceService.getDynamicLabel(id)` |
| `save()` | Persist a single label & its descriptions | `ReferenceService.saveOrUpdateDynamicLabel(label)` |
| `delete()` | Remove a label | `ReferenceService.deleteDynamicLabel(l)` |

### Design Choices & Constraints  

* **Service Layer** – The class uses a `ReferenceService` obtained from a `ServiceFactory`.  This suggests a DAO‑style architecture where persistence is abstracted behind a service.  
* **Language Handling** – All operations consider the merchant’s supported languages (`reflanguages`).  Descriptions are keyed by language ID.  
* **Visibility** – The UI probably sends an array of IDs (`visible`) that are currently selected; `saveList()` sets the `visible` flag accordingly.  
* **Error Handling** – Exceptions are caught, a technical error message is set, and the stack trace is logged.  No re‑throw or rollback is performed; the service layer presumably manages transactions.  

---

## 3. Functions/Methods  

| Method | Inputs | Outputs / Side Effects | Notes |
|--------|--------|------------------------|-------|
| `String displayList()` | None (uses `context.merchantId`) | Returns `SUCCESS`; populates `pages` with labels. | Sets page title, prepares languages. |
| `String saveList()` | None (uses `label.visible` array) | Persists visibility flags; returns `SUCCESS`. | Parses IDs; logs errors for bad IDs; re‑calls `displayList()` on failure. |
| `String displayDetails()` | None (uses `label.id`) | Loads `label` and populates `titles`, `descriptions`, `sefurl`. | Builds a language‑ordered map of descriptions. |
| `String save()` | None (uses `label`, `titles`, `descriptions`, `sefurl`, `reflanguages`) | Persists or updates a label; returns `SUCCESS` or `INPUT` on validation error. | Validates title, iterates over all supported languages, creates/updates descriptions. |
| `String delete()` | None (uses `label.id`) | Deletes a label if it belongs to the current merchant; returns `SUCCESS`. | Calls `displayList()` after deletion. |

### Reusable / Utility Methods  

* **`setPageTitle(String)`** – Sets the page title key.  
* **`prepareLanguages()`** – Loads the list of supported language IDs.  
* **`addFieldError(String, String)`** – Adds field‑specific validation errors.  
* **`setSuccessMessage()` / `setTechnicalMessage()`** – Helper for UI messages.  

These helpers are inherited from `ContentAction` (not shown in the snippet).

---

## 4. Dependencies  

| Dependency | Type | Comments |
|------------|------|----------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For `isBlank` checks. |
| `org.apache.log4j.Logger` | Third‑party | Basic logging. |
| `com.salesmanager.core.service.reference.ReferenceService` | Third‑party | Service layer for label persistence. |
| `com.salesmanager.core.service.ServiceFactory` | Third‑party | Service locator pattern. |
| `com.salesmanager.core.entity.reference.*` | Third‑party | JPA/Hibernate entities. |
| `com.salesmanager.core.constants.LabelConstants` | Internal | Holds section ID constants. |
| `com.salesmanager.central.BaseAction` / `ContentAction` | Internal | Base action helpers. |

All dependencies are standard Java EE / Spring‑ish components. No platform‑specific code is visible.

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality & Maintainability  

| Issue | Impact | Fix |
|-------|--------|-----|
| **Raw types** (`Collection<DynamicLabel>`, `Set`, `Map`) | Compile‑time warnings, potential `ClassCastException`. | Use generics everywhere (`Set<DynamicLabelDescription>` etc.). |
| **Unnecessary casts / raw iteration** | Verbose, error‑prone. | Use enhanced for‑loops (`for (DynamicLabel dl : labels)`). |
| **`reflanguages` handling** | Unclear contract, possible `ClassCastException` if types change. | Define `reflanguages` as `List<Integer>` or `Map<Integer,Integer>` with proper generics and document the meaning of keys/values. |
| **Error handling** | Silent failures (e.g., catching generic `Exception`, logging but continuing). | Catch specific exceptions, rethrow or return appropriate HTTP status. |
| **Magic string “false”** in `saveList()` | Confusing, not self‑explanatory. | Replace with a boolean flag or enum. |
| **Duplicate code** (`setPageTitle`, `prepareLanguages`) | Violates DRY. | Extract into a `@Before` helper or a dedicated method. |
| **No unit tests** | Hard to guarantee correctness. | Add JUnit tests covering each method, mocking `ReferenceService`. |

### 5.2 Functional Improvements  

1. **Validation Framework** – Replace manual `addFieldError` checks with a bean‑validation framework (e.g., Hibernate Validator).  
2. **Transaction Management** – Ensure that `ReferenceService` methods are transactional; consider adding rollback on failure.  
3. **Optimistic Locking** – Prevent lost updates when multiple users edit the same label.  
4. **Pagination** – `displayList()` could support paging for merchants with many pages.  
5. **Internationalization** – The `setPageTitle` calls use keys; ensure all UI texts are available in all supported locales.  

### 5.3 Security  

* **Authorization** – `delete()` verifies `merchantId`; similar checks should be added to `save()` and `saveList()` to prevent cross‑merchant data tampering.  
* **Input Sanitization** – Descriptions and titles should be sanitized to avoid XSS if rendered as HTML.  

### 5.4 Performance  

* **Bulk Update** – `saveList()` updates all labels in one call (`saveDynamicLabel(labels)`), which is efficient. However, consider using a batch update with a single SQL statement if the underlying ORM doesn’t.  
* **Lazy Loading** – Ensure that `label.getDescriptions()` is eagerly fetched or handled carefully to avoid N+1 problems.  

### 5.5 Documentation  

* Add Javadoc to each method, describing parameters, return values, and exceptions.  
* Document the contract of `reflanguages`, `visible`, `titles`, `descriptions`, `sefurl` arrays/lists.  

---

### Bottom Line  

`StoreFrontPagesAction` implements the core CRUD for merchant‑specific storefront pages in a fairly straightforward way.  The biggest technical debt lies in the use of raw types, ambiguous language handling, and limited error handling.  Refactoring to use generics, a proper validation framework, and clearer naming will make the code safer, easier to test, and more maintainable.  Adding unit tests and tightening security checks will further strengthen the module.

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
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.entity.reference.DynamicLabelDescriptionId;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.reference.ReferenceService;

/**
 * Custom pages [FAQ, Shipping Policies ...]
 * @author Carl Samson
 *
 */
public class StoreFrontPagesAction extends ContentAction {

	private static final long serialVersionUID = 4033353809089229393L;

	private Logger log = Logger.getLogger(StoreFrontPagesAction.class);



	private final static int SECTION_ID = LabelConstants.STORE_FRONT_CUSTOM_PAGES;

	/**
	 * Retreives Dynamic labels for a given section id and a merchant is
	 * 
	 * @return
	 */
	public String displayList() {

		try {
			
			super.setPageTitle("label.storefront.contentpagelist");

			super.prepareLanguages();

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			// get all pages
			pages = rservice.getDynamicLabels(super.getContext()
					.getMerchantid(), SECTION_ID, super.getLocale());

		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);
		}

		return SUCCESS;
	}

	public String saveList() {

		try {

			// get all
			super.setPageTitle("label.storefront.contentpagelist");
			super.prepareLanguages();
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			// get all pages
			Collection<DynamicLabel> labels = rservice.getDynamicLabels(super
					.getContext().getMerchantid(), SECTION_ID, super
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

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			displayList();
		}

		displayList();
		return SUCCESS;

	}

	public String displayDetails() {

		try {
			
			super.setPageTitle("label.storefront.contentpagedetails");

			super.prepareLanguages();

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
							sefurl.add(description.getSeUrl());
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

	public String save() {

		try {

			boolean hasError = false;
			super.setPageTitle("label.storefront.contentpagedetails");
			super.prepareLanguages();
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);

			//should never happen
			if (label == null) {
				label = new DynamicLabel();
			}
			
			if (StringUtils.isBlank(this.getLabel().getTitle())) {
				super
						.addFieldError(
								"title",
								getText("error.message.storefront.contentpageidrequired"));
				hasError = true;
			}



			Iterator i = reflanguages.keySet().iterator();
			while (i.hasNext()) {
				int langcount = (Integer) i.next();
				String title = (String) this.getTitles().get(langcount);
				String description = (String) this.getDescriptions().get(
						langcount);
				String seurl = (String) this.getSefurl().get(langcount);

				if (StringUtils.isBlank(title)) {
					super
							.addFieldError(
									"titles[" + langcount + "]",
									getText("error.message.storefront.contentpagetitlerequired"));
					hasError = true;
				}

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
				dldescription.setDynamicLabelTitle(title);
				dldescription.setSeUrl(seurl);

				Set descs = label.getDescriptions();
				if (descs == null) {
					descs = new HashSet();
				}

				descs.add(dldescription);

				label.setMerchantId(super.getContext().getMerchantid());
				label.setSectionId(LabelConstants.STORE_FRONT_CUSTOM_PAGES);
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



}



```
