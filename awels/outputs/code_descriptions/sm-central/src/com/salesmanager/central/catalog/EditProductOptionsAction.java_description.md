# EditProductOptionsAction.java

## Review

## 1. Summary

**Purpose**  
`EditProductOptionsAction` is a Struts‑2 action that manages product option entities for a merchant store. It can list, create, edit, and delete product options, while handling multi‑language support for option names and comments.

**Key Components**

| Component | Role |
|-----------|------|
| `prepare()` | Initialises request context, loads supported languages, and fetches existing options/types for display. |
| `displayProductOptions()` | Simply forwards to the UI page that lists product options. |
| `editProductOptions()` | Handles both creation (when `action==0`) and deletion (when `action==1`) of a product option. |
| `addProductOption()` | Redundant copy of the “add” branch of `editProductOptions()`. |
| Model fields (`names`, `comments`, `productOption`, `action`) | Bindings for form input; `names`/`comments` are language‑indexed lists. |
| `reflanguages` | Map that keeps the order‑to‑languageId relation so the UI can iterate in a deterministic order. |

**Design Patterns / Frameworks**

* **Struts‑2** – the action extends `BaseAction` and implements `Preparable`.  
* **DAO / Service Layer** – uses `CatalogService` and `MerchantService` from a `ServiceFactory`.  
* **Internationalisation** – `LanguageUtil`, `LabelUtil`, and `MessageUtil` provide i18n support.  
* **Logging** – Apache Log4j is used for error reporting.  

---

## 2. Detailed Description

### Execution Flow

1. **Preparation (`prepare()`)**  
   * Sets the page title.  
   * Retrieves the current `MerchantStore` and its supported languages.  
   * Populates the request with language data (`laanguages` attribute) and builds `reflanguages` to keep track of the language ordering.  
   * Loads all existing product options and option types and attaches them to the request.  

2. **Display (`displayProductOptions()`)**  
   * Returns `SUCCESS`, letting Struts forward to the JSP that shows the options list.

3. **Edit / Delete (`editProductOptions()`)**  
   * Validates that a `ProductOption` instance and language list exist.  
   * If `action==0` (add/update), iterates over `reflanguages`, pulls the corresponding `name`/`comment` from the language‑indexed lists, validates non‑blank names, constructs `ProductOptionDescription` objects, and stores them in a `HashSet`.  
   * Persists the option (or deletes it when `action==1`).  
   * Adds a success or error message for the UI.  

4. **Add (`addProductOption()`)**  
   * Performs **exactly** the same logic as the “add” branch of `editProductOptions()`.  
   * This duplication is unnecessary; the method could simply delegate to `editProductOptions()`.

### Assumptions & Constraints

* The size of `names`/`comments` lists must equal the number of supported languages.  
* The order of entries in `names`/`comments` is guaranteed to match the ordering stored in `reflanguages`.  
* The application relies on a `MerchantService`/`CatalogService` stack that uses Hibernate (or a similar ORM) under the hood.  
* Internationalisation is handled externally; the code assumes the message keys exist.  

### Architectural Choices

* **Separation of Concerns** – The action only orchestrates service calls; business logic resides in the service layer.  
* **Preparable** – `prepare()` is used to load data that the view needs, which is idiomatic in Struts‑2.  
* **Explicit Mapping (`reflanguages`)** – The developer chose a `Map<Integer, Integer>` to preserve ordering; a `List<Integer>` would have been more straightforward.  

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `prepare()` | Initialises request with languages, options, and option types. | None | void | Sets request attributes, logs errors. |
| `displayProductOptions()` | Forwards to the options list view. | None | `String` (`SUCCESS`) | None. |
| `editProductOptions()` | Adds or deletes a product option based on `action`. | None (uses action fields) | `String` (`SUCCESS`) | Persists changes, sets request messages. |
| `addProductOption()` | Adds a product option (duplicate logic). | None | `String` (`SUCCESS`) | Persists new option, sets request messages. |
| `get/set` methods for `names`, `comments`, `productOption`, `action`, `languages`, `reflanguages` | Standard JavaBean accessors. | – | – | – |

**Reusable / Utility Methods**

* The construction of `ProductOptionDescription` objects from language data is repeated in two places; this logic could be extracted into a private helper to avoid duplication.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | For blank string checks. |
| `org.apache.log4j.Logger` | Third‑party | Logging framework. |
| `com.opensymphony.xwork2.Preparable` | Struts‑2 | Action interface. |
| `com.salesmanager.central.*` | In‑house | `BaseAction`, `Context`, `ProfileConstants`. |
| `com.salesmanager.core.*` | In‑house | Entities (`ProductOption`, `ProductOptionDescription`, etc.), services (`CatalogService`, `MerchantService`), util classes (`LabelUtil`, `LanguageUtil`, `MessageUtil`). |
| `javax.servlet.*` | Servlet API | For request handling (indirectly via `BaseAction`). |

No platform‑specific or external web frameworks beyond Struts‑2 are used.

---

## 5. Additional Notes

### Strengths
* Clear separation between presentation and business layers.  
* Internationalisation support is consistently used.  
* The `prepare()` method pre‑loads all data needed by the view, avoiding unnecessary round‑trips.

### Weaknesses & Edge Cases
1. **Code Duplication**  
   * `editProductOptions()` and `addProductOption()` contain identical “add” logic.  
   * Suggest refactoring into a single private method or delegating one to the other.

2. **Raw Types & Generics**  
   * `Map` and `Collection` are used without generics (e.g., `Map languagesMap`).  
   * Leads to unchecked casts and potential `ClassCastException`.  
   * Replace with typed collections: `Map<Integer, Language>` and `Collection<Language>`.

3. **List Indexing Assumptions**  
   * `names.get(langcount)` and `comments.get(langcount)` assume lists are properly populated and aligned with `reflanguages`.  
   * If a list is shorter, an `IndexOutOfBoundsException` will be thrown.  
   * Validate list sizes before iteration.

4. **Missing Transaction Management**  
   * Service calls are wrapped in a try/catch but no explicit transaction handling.  
   * Depending on the underlying ORM configuration, you might need to ensure atomicity (especially for delete operations).

5. **Error Handling**  
   * Errors are logged but the user only sees a generic “technical” message.  
   * Consider exposing more context or redirecting to a dedicated error page.

6. **Concurrency & Thread Safety**  
   * The action is a per‑request instance, so thread safety is not a concern.  
   * However, if the service layer uses shared mutable state, ensure proper synchronization.

7. **Internationalisation Keys**  
   * The code references keys like `messages.productoption.name.required`.  
   * Ensure these keys exist in all supported locales to avoid missing‑message warnings.

8. **Possible Refactoring**  
   * Use a `List<ProductOptionDescription>` instead of a `HashSet` if ordering matters.  
   * Consider using a builder pattern for `ProductOptionDescription` construction.  
   * Replace the `Map<Integer, Integer>` (`reflanguages`) with a simple `List<Integer>` representing language IDs in the required order.

### Future Enhancements
* **Bulk Operations** – Add support for editing/deleting multiple options at once.  
* **Validation Framework** – Integrate Struts‑2 validators to centralise input validation.  
* **Unit Tests** – Write unit tests for `editProductOptions()` and `addProductOption()` to cover edge cases.  
* **RESTful API** – Expose product option CRUD operations via a REST endpoint for better integration with front‑end frameworks.  

--- 

**Verdict**  
The action serves its purpose but would benefit from refactoring to eliminate duplication, tighten type safety, and improve robustness against malformed input. Addressing these points will make the code easier to maintain and extend.

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
package com.salesmanager.central.catalog;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.Preparable;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.catalog.ProductOption;
import com.salesmanager.core.entity.catalog.ProductOptionDescription;
import com.salesmanager.core.entity.catalog.ProductOptionDescriptionId;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.MessageUtil;

public class EditProductOptionsAction extends BaseAction implements Preparable {

	private List<String> names = new ArrayList<String>();
	private List<String> comments = new ArrayList<String>();
	private ProductOption productOption = null;
	private int action = -1; // 0 is add 1 is delete

	private Collection<Language> languages;// used in the page as an index
	private Map<Integer, Integer> reflanguages = new HashMap();// reference
																// count -
																// languageId

	private Logger log = Logger.getLogger(EditProductOptionsAction.class);

	public void prepare() {

		try {
			
			super.setPageTitle("label.product.productoptions.title");

			MerchantService service = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			MerchantStore mstore = service.getMerchantStore(merchantid);

			if (mstore == null) {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"errors.profile.storenotcreated"));
			} else {

				Map languagesMap = mstore.getGetSupportedLanguages();

				languages = languagesMap.values();// collection reverse the map

				super.getServletRequest().setAttribute("laanguages", languages);

				// int count = languagesMap.size()-1;
				int count = 0;
				Iterator langit = languagesMap.keySet().iterator();
				while (langit.hasNext()) {
					Integer langid = (Integer) langit.next();
					Language lang = (Language) languagesMap.get(langid);
					reflanguages.put(count, langid);
					count++;
				}

			}

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			Collection options = cservice.getProductOptions(merchantid);
			super.getServletRequest().setAttribute("options", options);

			Collection optionTypes = cservice.getProductOptionTypes();
			super.getServletRequest().setAttribute("optionTypes", optionTypes);

		} catch (Exception e) {
			log.error(e);
		}

	}

	public String displayProductOptions() throws Exception {

		return SUCCESS;
	}

	public String editProductOptions() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		HashSet descriptionsset = new HashSet();

		if (this.getProductOption() == null) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error("Should have received a ProductOption");
			return SUCCESS;
		}

		if (getLanguages() == null || getLanguages().size() == 0) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.profile.storenotcreated"));
			return SUCCESS;
		}

		try {

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			if (this.getAction() == 0) {

				// names

				Iterator i = reflanguages.keySet().iterator();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					String name = (String) this.getNames().get(langcount);
					String comment = (String) this.getComments().get(langcount);

					int submitedlangid = (Integer) reflanguages.get(langcount);
					String langCode = LanguageUtil
							.getLanguageStringCode(submitedlangid);

					if (StringUtils.isBlank(name)) {
						MessageUtil.addErrorMessage(super.getServletRequest(),
								LabelUtil.getInstance().getText(super.getLocale(),
										"messages.productoption.name.required")
										+ " (" + langCode + ")");
						return SUCCESS;
					}

					ProductOptionDescription desc = new ProductOptionDescription();
					ProductOptionDescriptionId id = new ProductOptionDescriptionId();
					id.setLanguageId(submitedlangid);
					desc.setProductOptionName(name);
					desc.setProductOptionComment(comment);
					desc.setId(id);

					descriptionsset.add(desc);

				}

			}

			ProductOption option = this.getProductOption();

			if (this.getAction() == 0) { // add
				option.setMerchantId(merchantid);

				option.setDescriptions(descriptionsset);

				cservice.saveOrUpdateProductOption(option);

			} else {
				cservice.deleteProductOption(option.getProductOptionId());
			}

			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
		}

		return SUCCESS;

	}

	public String addProductOption() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		HashSet descriptionsset = new HashSet();

		if (this.getProductOption() == null) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error("Should have received a ProductOption");
			return SUCCESS;
		}

		if (getLanguages() == null || getLanguages().size() == 0) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.profile.storenotcreated"));
			return SUCCESS;
		}

		try {

			// names

			Iterator i = reflanguages.keySet().iterator();
			while (i.hasNext()) {
				int langcount = (Integer) i.next();
				String name = (String) this.getNames().get(langcount);
				String comment = (String) this.getComments().get(langcount);

				int submitedlangid = (Integer) reflanguages.get(langcount);
				String langCode = LanguageUtil
						.getLanguageStringCode(submitedlangid);

				if (StringUtils.isBlank(name)) {
					MessageUtil.addErrorMessage(super.getServletRequest(),
							LabelUtil.getInstance().getText(super.getLocale(),
									"messages.productoption.name.required")
									+ " (" + langCode + ")");
					return SUCCESS;
				}

				ProductOptionDescription desc = new ProductOptionDescription();
				ProductOptionDescriptionId id = new ProductOptionDescriptionId();
				id.setLanguageId(submitedlangid);
				desc.setProductOptionName(name);
				desc.setProductOptionComment(comment);
				desc.setId(id);

				descriptionsset.add(desc);

			}

			ProductOption option = this.getProductOption();
			option.setMerchantId(merchantid);
			option.setDescriptions(descriptionsset);

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			cservice.saveOrUpdateProductOption(option);

			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
		}

		return SUCCESS;

	}

	public List<String> getComments() {
		return comments;
	}

	public void setComments(List<String> comments) {
		this.comments = comments;
	}

	public Collection<Language> getLanguages() {
		return languages;
	}

	public void setLanguages(Collection<Language> languages) {
		this.languages = languages;
	}

	public List<String> getNames() {
		return names;
	}

	public void setNames(List<String> names) {
		this.names = names;
	}

	public Map<Integer, Integer> getReflanguages() {
		return reflanguages;
	}

	public void setReflanguages(Map<Integer, Integer> reflanguages) {
		this.reflanguages = reflanguages;
	}

	public ProductOption getProductOption() {
		return productOption;
	}

	public void setProductOption(ProductOption productOption) {
		this.productOption = productOption;
	}

	public int getAction() {
		return action;
	}

	public void setAction(int action) {
		this.action = action;
	}

}



```
