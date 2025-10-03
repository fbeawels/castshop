# EditProductOptionsValuesAction.java

## Review

## 1. Summary

**Purpose**  
`EditProductOptionsValuesAction` is a Struts 2 action that manages CRUD operations for *product option values* in a multi‑merchant, multi‑language e‑commerce platform.  
The action handles:

| Operation | What it does |
|-----------|--------------|
| **displayProductOptionsValues** | Show all option values or the values belonging to a specific `ProductOption`. |
| **associateProductOptionValue** | Link an existing `ProductOptionValue` to a `ProductOption`. |
| **editProductOptionsValues** | Add, delete, or unlink option values, including image handling. |
| **addProductOptionValue** | Create a new `ProductOptionValue` (with optional image upload). |

**Key Components**

* **`CatalogService`** – Business service used to persist/lookup catalog entities.  
* **`MerchantService`** – Retrieves merchant context.  
* **`FileModule`** – Handles file upload / delete via Spring beans.  
* **`Context` / `ProfileConstants`** – Store session‑level information (merchant ID, language).  
* **`ProductOption…` / `ProductOptionValue…`** – JPA entities representing catalog metadata.  

**Design Patterns / Libraries**

* **Struts 2** – Action, `Preparable`, request/session handling.  
* **Spring** – Dependency injection for `FileModule` via `SpringUtil`.  
* **Apache Commons** – `Configuration`, `StringUtils`.  
* **Log4j** – Logging.  
* **Java Collections** – Raw types are still used in several places.  

---

## 2. Detailed Description

### Initialization (`prepare`)
* Sets the page title.
* Pulls the merchant context from the HTTP session and retrieves the associated `MerchantStore`.
* Builds a `languages` collection and a `reflanguages` map that maps an *index* to the language ID – used to correlate form input names with language codes.
* Stores these objects in request attributes for the JSP to render.

### Execution Paths

1. **`displayProductOptionsValues`**  
   * Loads the `ProductOption` (if provided) and all its values.  
   * Creates a *display list* of all product‑option values **not** currently linked to the selected option.  
   * Sets the `optionsvalues` request attribute for the view.

2. **`associateProductOptionValue`**  
   * Validates the presence of both a `ProductOption` and a `ProductOptionValueId`.  
   * Calls `associateProductOptionValueToProductOption` on the `CatalogService`.  
   * Returns a result string `"associate-success"` so the result page can show a success message.

3. **`editProductOptionsValues`**  
   * Handles three actions (add, delete, remove association) indicated by the `action` field.  
   * **Add**: builds `ProductOptionValueDescription` objects for each language; saves via `CatalogService`.  
   * **Delete**: removes the image file from disk (if present) then deletes the entity.  
   * **Remove association**: detaches a value from its option.

4. **`addProductOptionValue`**  
   * Similar to *Add* in `editProductOptionsValues`, but also supports file upload for an image.  
   * After persisting the entity, if an image was provided it uploads the file and updates the entity’s image field.

### Cleanup
No explicit cleanup is performed. File deletion occurs only when a value is removed. Logging and error messages are the primary cleanup mechanisms.

### Dependencies & Assumptions
* The code assumes the existence of a `Context` object in the HTTP session, containing the current `merchantid` and `lang`.
* `ServiceFactory` is a legacy singleton holder for DAO services; it is not thread‑safe in its current form.
* `SpringUtil.getBean("localfile")` fetches a `FileModule`; this bean must be registered in the Spring context.
* Image uploads are written directly to a file system location derived from `FileUtil.getProductFilePath()`, which is not shown in the snippet.
* All string comparison and emptiness checks are performed using `StringUtils.isBlank`.

### Architectural Observations
* **Single Action, Multiple Concerns** – One class handles display, association, edit, delete, and file upload. This violates the Single Responsibility Principle and makes the action bulky.
* **Raw Types** – Many collections are declared without generics (`Collection optionList`, `Map<Integer, Integer> reflanguages = new HashMap();`). This can lead to `ClassCastException`s and hinders readability.
* **Exception Handling** – Most methods swallow `Exception` and log the stack trace, but they often continue execution or return a generic `SUCCESS`. More granular exception handling would improve robustness.
* **Hard‑coded Strings** – Paths like `"core.product.image"` and the image folder string are hard‑coded. They should be externalised to a configuration file or constants class.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `prepare()` | Initialises language data, sets page title. | None | None | Sets request attributes, logs errors. |
| `displayProductOptionsValues()` | Renders product option values (all or per option). | None | `String` (Struts result) | Sets request attributes, logs errors. |
| `associateProductOptionValue()` | Links a value to an option. | None | `String` | Calls service, logs errors. |
| `editProductOptionsValues()` | Handles add / delete / unlink of option values. | None | `String` | Persists changes, deletes files, logs errors. |
| `addProductOptionValue()` | Creates a new value (optional image). | None | `String` | Persists changes, uploads file, logs errors. |
| `getLanguages()` / `setLanguages()` | Get/set supported languages collection. | None / `Collection<Language>` | `Collection<Language>` | None |
| `getNames()` / `setNames()` | Get/set list of localized names. | None / `List<String>` | `List<String>` | None |
| `getReflanguages()` / `setReflanguages()` | Get/set map index→languageId. | None / `Map<Integer,Integer>` | `Map<Integer,Integer>` | None |
| `getAction()` / `setAction()` | Get/set action code (add=0, delete=1, remove=2). | None / `int` | `int` | None |
| `getProductOption()` / `setProductOption()` | Get/set current `ProductOption`. | None / `ProductOption` | `ProductOption` | None |
| `getProductOptionValue()` / `setProductOptionValue()` | Get/set current `ProductOptionValue`. | None / `ProductOptionValue` | `ProductOptionValue` | None |
| `getOptionList()` / `setOptionList()` | Get/set list of option values not linked. | None / `Collection` | `Collection` | None |
| `getProductOptionValueId()` / `setProductOptionValueId()` | Get/set ID of value to associate. | None / `Long` | `Long` | None |
| `getProductOptionDisplay()` / `setProductOptionDisplay()` | Get/set display object for UI. | None / `ProductOptionDisplay` | `ProductOptionDisplay` | None |
| `getUploadimage()` / `setUploadimage()` | File upload object. | None / `File` | `File` | None |
| `getUploadimageFileName()` / `setUploadimageFileName()` | Name of uploaded image. | None / `String` | `String` | None |
| `getUploadimageContentType()` / `setUploadimageContentType()` | MIME type of uploaded image. | None / `String` | `String` | None |

**Reusable Utility** – The code for building `ProductOptionValueDescription` objects is duplicated in `editProductOptionsValues` and `addProductOptionValue`. This could be extracted into a private helper method.

---

## 4. Dependencies

| Library / Framework | Role | Notes |
|---------------------|------|-------|
| **Struts 2** | Action framework | `Preparable` interface, request/session helpers. |
| **Spring** | Bean lookup (`SpringUtil.getBean`) | Provides `FileModule`. |
| **Apache Commons Configuration** | Reads properties (`conf`). | Only used for image folder path (hard‑coded). |
| **Apache Commons Lang** | `StringUtils.isBlank`. | Utility for string checks. |
| **Log4j** | Logging. | Classic log4j API. |
| **Java EE / Servlet API** | Request/Session handling (`super.getServletRequest()`). | Implicit dependency. |
| **JPA / Hibernate** | Entities (`ProductOption*`). | Not explicitly imported; assumed in `com.salesmanager.core.entity`. |
| **Custom Services** | `CatalogService`, `MerchantService`. | Provided by the application’s service layer. |
| **File Handling** | `FileModule`, `FileException`, `FileUtil`. | Handles upload and deletion. |

All dependencies are *third‑party* except for the application‑specific ones (`com.salesmanager.*`). There are no platform‑specific libraries beyond the Servlet container.

---

## 5. Additional Notes & Recommendations

### 5.1. Potential Edge Cases & Faults

| Area | Issue | Suggested Fix |
|------|-------|---------------|
| **Null Checks** | `this.getProductOption()` or `this.getProductOptionValue()` may be null in several branches, but the code often proceeds and will throw NPEs. | Add defensive checks early, return appropriate error result. |
| **Concurrent Access** | `ServiceFactory.getService` is a singleton; if the underlying services are not thread‑safe, concurrent requests could clash. | Refactor to use Spring injection for services. |
| **File Path Construction** | Image names are constructed from the value ID and original filename. If the filename contains path separators, a path traversal attack is possible. | Sanitize file names, strip directories, use a random UUID. |
| **MIME Validation** | No check on `uploadimagecontenttype`. Arbitrary files could be uploaded. | Validate against a whitelist of image MIME types. |
| **Exception Handling** | Broad `catch (Exception e)` blocks swallow all errors and often return `"SUCCESS"`, hiding failures from the user. | Narrow the catch blocks to specific exceptions; propagate or display user‑friendly error messages. |
| **Hard‑coded Strings** | `"core.product.image"` and the file folder string are magic constants. | Externalise to `properties` or constants. |
| **Raw Types & Generics** | Collections without generics reduce type safety. | Convert to generics (`Map<Integer,Integer>`, `Collection<Language>`, etc.). |
| **Duplicated Code** | Building description objects appears twice. | Extract helper method `buildDescriptions(names, reflanguages)`. |
| **User Input Validation** | Only checks for blank names, but not length, special characters, or duplicates. | Use Struts 2 validation framework or custom validator. |

### 5.2. Design Improvements

| Idea | Rationale |
|------|-----------|
| **Separate Actions** | Split into `DisplayOptionValuesAction`, `AssociateOptionValueAction`, `EditOptionValueAction`, `AddOptionValueAction`. | Improves maintainability and adheres to Single Responsibility Principle. |
| **Service‑Driven Logic** | Move all business logic (description building, file handling) into `CatalogService` or a dedicated `OptionValueService`. | Keeps action thin and focused on web concerns. |
| **Spring Integration** | Autowire services and `FileModule` directly instead of using `ServiceFactory` and `SpringUtil`. | Modern Spring practices, easier unit testing. |
| **Use of Validation Annotations** | Apply `@Required`, `@Length` annotations or Struts 2 XML validation. | Centralises validation logic, improves UI feedback. |
| **Internationalisation Enhancements** | Pass language codes directly rather than mapping indices. | Simplifies JSP binding and reduces mapping complexity. |
| **Error Reporting** | Return detailed JSON or JSP fragments on error, rather than generic messages. | Better UX for admin interface. |

### 5.3. Security & Performance

* **Security** – Ensure the uploaded file is stored in a non‑executable directory; set appropriate file permissions. Consider using a storage service (e.g., Amazon S3) for scalability.  
* **Performance** – Batch delete of unused image files when bulk deleting options. Use lazy loading for large lists of option values.  

### 5.4. Testing

* **Unit Tests** – Mock `CatalogService`, `MerchantService`, and `FileModule` to test each action branch.  
* **Integration Tests** – Verify that images are uploaded to the correct location and that associations are correctly persisted.  
* **Security Tests** – Attempt to upload a malicious file name containing `../` to ensure it is sanitized.  

---

### Final Verdict

The action achieves its functional goals but suffers from code duplication, raw types, hard‑coded strings, and a lack of robust validation. Refactoring into smaller, testable components and modernizing dependency injection will greatly improve maintainability, security, and clarity.

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

import java.io.File;
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.Preparable;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.catalog.ProductOption;
import com.salesmanager.core.entity.catalog.ProductOptionDescription;
import com.salesmanager.core.entity.catalog.ProductOptionValue;
import com.salesmanager.core.entity.catalog.ProductOptionValueDescription;
import com.salesmanager.core.entity.catalog.ProductOptionValueDescriptionId;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.module.impl.application.files.FileException;
import com.salesmanager.core.module.model.application.FileModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.SpringUtil;

public class EditProductOptionsValuesAction extends BaseAction implements
		Preparable {

	private List<String> names = new ArrayList<String>();

	private ProductOption productOption;
	private ProductOptionDisplay productOptionDisplay;

	private ProductOptionValue productOptionValue = null;
	private int action = -1; // 0 is add 1 is delete

	private Collection<Language> languages;// used in the page as an index
	private Map<Integer, Integer> reflanguages = new HashMap();// reference
																// count -
																// languageId

	private Collection optionList = null;
	private Long productOptionValueId = null;

	// image upload
	private String uploadimagefilename;
	private String uploadimagecontenttype;
	private File uploadimage;

	private static Configuration conf = PropertiesUtil.getConfiguration();

	private Logger log = Logger.getLogger(EditProductOptionsValuesAction.class);

	public void prepare() {
		
		super.setPageTitle("label.product.productoptionsvalues.title");

		try {

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

		} catch (Exception e) {
			log.error(e);
		}

	}

	public String displayProductOptionsValues() throws Exception {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			Collection values = null;

			// Get optionValues
			if (this.getProductOption() != null) {// get values for a given
													// ProductOption
				long id = productOption.getProductOptionId();
				productOption = cservice.getProductOptionWithValues(this
						.getProductOption().getProductOptionId());

				if (productOption == null) {
					log
							.error("ProductOption was not supposed to be null for id "
									+ id);
					MessageUtil
							.addErrorMessage(super.getServletRequest(),
									LabelUtil.getInstance().getText(super.getLocale(),
											"errors.technical"));
					return SUCCESS;
				}

				ProductOptionDisplay pod = new ProductOptionDisplay();
				pod.setProductOptionId(productOption.getProductOptionId());
				pod.setProductOptionName(String.valueOf(productOption
						.getProductOptionId()));

				Set optdescs = productOption.getDescriptions();
				if (optdescs != null) {
					Iterator desci = optdescs.iterator();
					while (desci.hasNext()) {
						ProductOptionDescription description = (ProductOptionDescription) desci
								.next();
						if (description.getId().getLanguageId() == LanguageUtil
								.getLanguageNumberCode(ctx.getLang())) {
							pod.setProductOptionName(description
									.getProductOptionName());
						}
					}
				}

				this.setProductOptionDisplay(pod);

				values = productOption.getValues();

				// prepare association list
				Collection alllist = cservice.getProductOptionValues(ctx
						.getMerchantid());
				List displaylist = new ArrayList();
				if (alllist != null) {
					Iterator i = alllist.iterator();
					while (i.hasNext()) {
						ProductOptionValue value = (ProductOptionValue) i
								.next();
						if (!values.contains(value)) {
							ProductOptionValueDisplay pov = new ProductOptionValueDisplay();
							pov.setProductOptionValueId(value
									.getProductOptionValueId());
							pov.setProductOptionValueName(String.valueOf(value
									.getProductOptionValueId()));
							Set descs = value.getDescriptions();
							if (descs != null) {
								Iterator desci = descs.iterator();
								while (desci.hasNext()) {
									ProductOptionValueDescription description = (ProductOptionValueDescription) desci
											.next();
									if (description.getId().getLanguageId() == LanguageUtil
											.getLanguageNumberCode(ctx
													.getLang())) {
										pov
												.setProductOptionValueName(description
														.getProductOptionValueName());
									}
								}
							}
							displaylist.add(pov);
						}

					}
				}
				optionList = displaylist;

			} else {// get all values
				values = cservice.getProductOptionValues(ctx.getMerchantid());
			}

			super.getServletRequest().setAttribute("optionsvalues", values);

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
		}

		return SUCCESS;
	}

	public String associateProductOptionValue() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		if (this.getProductOption() == null) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error("Should have received a ProductOptionValue");
			return "associate-success";
		}

		if (this.getProductOptionValueId() == null) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error("Should have received a ProductOptionValue");
			return "associate-success";
		}

		if (getLanguages() == null || getLanguages().size() == 0) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.profile.storenotcreated"));
			return SUCCESS;
		}

		try {

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			cservice.associateProductOptionValueToProductOption(this
					.getProductOption().getProductOptionId(), this
					.getProductOptionValueId());

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
		}

		return "associate-success";

	}

	public String editProductOptionsValues() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		HashSet descriptionsset = new HashSet();

		if (this.getProductOptionValue() == null) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error("Should have received a ProductOptionValue");
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

			if (this.getAction() == 0) {// add

				// names

				Iterator i = reflanguages.keySet().iterator();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					String name = (String) this.getNames().get(langcount);

					int submitedlangid = (Integer) reflanguages.get(langcount);
					String langCode = LanguageUtil
							.getLanguageStringCode(submitedlangid);

					if (StringUtils.isBlank(name)) {
						MessageUtil
								.addErrorMessage(
										super.getServletRequest(),
										LabelUtil
												.getInstance()
												.getText(
														"messages.productoptionvalue.name.required")
												+ " (" + langCode + ")");
						return SUCCESS;
					}

					ProductOptionValueDescription desc = new ProductOptionValueDescription();
					ProductOptionValueDescriptionId id = new ProductOptionValueDescriptionId();
					id.setLanguageId(submitedlangid);
					desc.setProductOptionValueName(name);
					desc.setId(id);

					descriptionsset.add(desc);

				}

			}

			ProductOption option = this.getProductOption();
			ProductOptionValue optionValue = this.getProductOptionValue();

			if (this.getAction() == 0) { // add
				optionValue.setMerchantId(merchantid);
				optionValue.setDescriptions(descriptionsset);
				if (option != null && option.getProductOptionId() > 0) {
					cservice.saveOrUpdateProductOptionValueToProductOption(
							optionValue, option.getProductOptionId());
					MessageUtil.addMessage(super.getServletRequest(), LabelUtil
							.getInstance().getText(
									"message.confirmation.success"));
					return "associate-success";

				} else {
					cservice.saveOrUpdateProductOptionValue(optionValue);
					MessageUtil.addMessage(super.getServletRequest(), LabelUtil
							.getInstance().getText(
									"message.confirmation.success"));
					return SUCCESS;
				}

			} else if (this.getAction() == 1) {// delete

				optionValue = cservice.getProductOptionValue(optionValue
						.getProductOptionValueId());

				FileModule fh = (FileModule) SpringUtil.getBean("localfile");
				if (!StringUtils.isBlank(optionValue
						.getProductOptionValueImage())) {
					//String folder = conf
					//		.getString("core.product.image.filefolder")
					String folder = FileUtil.getProductFilePath()
							+ "/" + merchantid + "/";
					fh.deleteFile(merchantid, new File(new StringBuffer()
							.append(folder).append(
									optionValue.getProductOptionValueImage())
							.toString()));
				}

				cservice.deleteProductOptionValue(optionValue);

				MessageUtil.addMessage(super.getServletRequest(), LabelUtil
						.getInstance().getText("message.confirmation.success"));
				if (option != null && option.getProductOptionId() > 0) {
					return "associate-success";
				} else {
					return SUCCESS;
				}

			} else if (this.getAction() == 2) {// remove association

				if (option == null || option.getProductOptionId() == 0) {
					MessageUtil
							.addErrorMessage(super.getServletRequest(),
									LabelUtil.getInstance().getText(super.getLocale(),
											"errors.technical"));
					log.error("Should have received a ProductOption");
					return SUCCESS;
				}

				cservice.removeProductOptionValueToProductOption(option
						.getProductOptionId(), optionValue
						.getProductOptionValueId());
				MessageUtil.addMessage(super.getServletRequest(), LabelUtil
						.getInstance().getText("message.confirmation.success"));

				return "associate-success";

			}

			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));
			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			return SUCCESS;
		}

	}

	public String addProductOptionValue() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		HashSet descriptionsset = new HashSet();

		if (this.getProductOptionValue() == null) {
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

				int submitedlangid = (Integer) reflanguages.get(langcount);
				String langCode = LanguageUtil
						.getLanguageStringCode(submitedlangid);

				if (StringUtils.isBlank(name)) {
					MessageUtil
							.addErrorMessage(
									super.getServletRequest(),
									LabelUtil
											.getInstance()
											.getText(
													"messages.productoptionvalue.name.required")
											+ " (" + langCode + ")");
					return SUCCESS;
				}

				ProductOptionValueDescription desc = new ProductOptionValueDescription();
				ProductOptionValueDescriptionId id = new ProductOptionValueDescriptionId();
				id.setLanguageId(submitedlangid);
				desc.setProductOptionValueName(name);
				desc.setId(id);

				descriptionsset.add(desc);

			}

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			ProductOptionValue optionValue = this.getProductOptionValue();
			optionValue.setMerchantId(merchantid);
			optionValue.setDescriptions(descriptionsset);

			if (this.getProductOption() != null
					&& this.getProductOption().getProductOptionId() > 0) {

				cservice.saveOrUpdateProductOptionValueToProductOption(
						optionValue, this.getProductOption()
								.getProductOptionId());

				if (this.getUploadimage() != null
						&& !StringUtils.isBlank(this.getUploadimageFileName())) {
					try {
						FileModule fh = (FileModule) SpringUtil
								.getBean("localfile");
						// String folder =
						// conf.getString("core.product.image.filefolder") + "/"
						// + merchantid + "/";
						String optionName = new StringBuffer().append(
								optionValue.getProductOptionValueId()).append(
								"_").append(this.getUploadimageFileName())
								.toString();
						fh.uploadFile(merchantid, "core.product.image", this
								.getUploadimage(), optionName,
								this.uploadimagecontenttype);
						optionValue.setProductOptionValueImage(optionName);
						cservice.saveOrUpdateProductOptionValue(optionValue);
					} catch (FileException e) {
						displayProductOptionsValues();
						if (e instanceof FileException) {
							this.addActionError(getText(e.getMessage()));
							return INPUT;
						} else {
							log.error(e);
							this
									.addActionError(getText("error.message.imagesnotuploaded"));
							return INPUT;
						}
					}
				}

				MessageUtil.addMessage(super.getServletRequest(), LabelUtil
						.getInstance().getText("message.confirmation.success"));
				return "associate-success";

			} else {

				cservice.saveOrUpdateProductOptionValue(optionValue);
				MessageUtil.addMessage(super.getServletRequest(), LabelUtil
						.getInstance().getText("message.confirmation.success"));

				if (this.getUploadimage() != null
						&& !StringUtils.isBlank(this.getUploadimageFileName())) {
					try {
						FileModule fh = (FileModule) SpringUtil
								.getBean("localfile");

						String optionName = new StringBuffer().append(
								optionValue.getProductOptionValueId()).append(
								"_").append(this.getUploadimageFileName())
								.toString();
						fh.uploadFile(merchantid, "core.product.image", this
								.getUploadimage(), optionName,
								this.uploadimagecontenttype);
						optionValue.setProductOptionValueImage(optionName);
						cservice.saveOrUpdateProductOptionValue(optionValue);
					} catch (FileException e) {
						displayProductOptionsValues();
						if (e instanceof FileException) {
							this.addActionError(getText(e.getMessage()));
							return INPUT;
						} else {
							log.error(e);
							this
									.addActionError(getText("error.message.imagesnotuploaded"));
							return INPUT;
						}
					}
				}

				return SUCCESS;
			}

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			return SUCCESS;
		}

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

	public int getAction() {
		return action;
	}

	public void setAction(int action) {
		this.action = action;
	}

	public ProductOption getProductOption() {
		return productOption;
	}

	public void setProductOption(ProductOption productOption) {
		this.productOption = productOption;
	}

	public ProductOptionValue getProductOptionValue() {
		return productOptionValue;
	}

	public void setProductOptionValue(ProductOptionValue productOptionValue) {
		this.productOptionValue = productOptionValue;
	}

	public Collection getOptionList() {
		return optionList;
	}

	public void setOptionList(Collection optionList) {
		this.optionList = optionList;
	}

	public Long getProductOptionValueId() {
		return productOptionValueId;
	}

	public void setProductOptionValueId(Long productOptionValueId) {
		this.productOptionValueId = productOptionValueId;
	}

	public ProductOptionDisplay getProductOptionDisplay() {
		return productOptionDisplay;
	}

	public void setProductOptionDisplay(
			ProductOptionDisplay productOptionDisplay) {
		this.productOptionDisplay = productOptionDisplay;
	}

	public File getUploadimage() {
		return uploadimage;
	}

	public void setUploadimage(File uploadimage) {
		this.uploadimage = uploadimage;
	}

	public String getUploadimageFileName() {
		return uploadimagefilename;
	}

	public void setUploadimageFileName(String uploadimagefilename) {
		this.uploadimagefilename = uploadimagefilename;
	}

	public String getUploadimageContentType() {
		return uploadimagecontenttype;
	}

	public void setUploadimageContentType(String uploadimagecontenttype) {
		this.uploadimagecontenttype = uploadimagecontenttype;
	}

}



```
