# StoreFrontAction.java

## Review

## 1. Summary

`StoreFrontAction` is a Struts‑2/ Spring‑based action that manages the front‑end configuration of a merchant store.  
It handles:

* **Template selection** – loads the list of available front‑end templates and determines the currently selected one.  
* **Dynamic labels** – loads, creates, updates and deletes `DynamicLabel` entities for store description, page title, meta‑keywords, meta‑description and slide images.  
* **Slide management** – allows ordering of slides, editing slide metadata, uploading/deleting slide images.  
* **UI preparation** – prepares collections for the view layer (templates, slide list, image paths, etc.).

The action relies heavily on the `ReferenceService` for label CRUD, the `MerchantService` for store information, and a few utility beans (`FileModule`, `FileUtil`, `LabelUtil`).  

The code uses a mix of raw collections (`Collection`, `List`, `Map`) and manual type‑casting rather than generics, which is a legacy style often seen in older Java codebases.

---

## 2. Detailed Description

### Core Flow

| Step | Method | Purpose | Notes |
|------|--------|---------|-------|
| 1 | `displayStoreFrontConfig()` | Entry point for the “store front configuration” page. | Calls `prepareLanguages()` (inherited) and `prepareContent()`. |
| 2 | `prepareContent()` | Loads templates, current template, slider configuration, and all dynamic labels that belong to the store. | Handles language‑specific values for the four label sections. |
| 3 | `editStoreFontConfig()` | Handles form submission to update store description, title, meta‑keywords and meta‑description. | Deletes existing labels of the four sections, builds new `DynamicLabel` objects per language, and persists them. |
| 4 | `updateSlideList()` | Receives an ordered list of slide labels (likely from a UI drag‑drop). | Calls `super.updatePageList()` to reorder the labels then persists them. |
| 5 | `viewSlide()` | Prepares a single slide for display, including image metadata. | Creates a `DynamicImage` instance and places it in the request scope. |
| 6 | `deleteSlide()` | Deletes a slide label and its associated image file. | Checks that the label exists and removes the image from the file system. |
| 7 | `editSlide()` | Handles editing of a slide (metadata and optional image upload). | Uses `FileModule` to upload new images and updates the label accordingly. |
| 8 | `deleteFile()` | Removes only the image file of a slide, leaving the label itself intact. | Validates ownership before deleting. |

### Interaction with External Services

* **`ReferenceService`** – CRUD for `DynamicLabel` entities, fetching by section, etc.  
* **`MerchantService`** – Retrieves the `MerchantStore` (for template module).  
* **`FileModule`** – Handles file upload/download/delete for images.  
* **`LabelUtil` / `CountryUtil` / `FileUtil`** – Utility classes for localization, file path resolution, etc.  
* **`SpringUtil.getBean()`** – Retrieves the `FileModule` bean named `localfile`.

### Assumptions & Constraints

* The action is request‑scoped; no synchronization is required.  
* The `reflanguages` collection (likely a `List<Integer>`) is populated by `prepareLanguages()` in the parent class.  
* The action assumes that the current merchant ID and locale are available in the `Context`.  
* All file operations are performed under the merchant’s “bin” directory.  

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `prepareContent()` | Loads template list, current template, slider config, and dynamic labels. | None (uses `Context`) | Populates `templates`, `currrentTempate`, `sliderConf`, `storeDescription`, `storeFrontPageTitle`, `metaKeywords`, `metaDescription`, `pages` | None |
| `displayStoreFrontConfig()` | Display config page. | None | `"SUCCESS"` | Calls `prepareContent()` |
| `editStoreFontConfig()` | Save store description, title, meta‑keywords/description. | Form values (`storeDescription`, `storeFrontPageTitle`, `metaKeywords`, `metaDescription`) | `"SUCCESS"` or `"INPUT"` | Persists new `DynamicLabel` entities; deletes old ones |
| `updateSlideList()` | Reorder slide labels. | None (label list comes from request) | `"SUCCESS"` | Calls `updatePageList()`, saves reordered labels |
| `viewSlide()` | Show a single slide. | `label` (set by Struts) | `"SUCCESS"` | Sets request attribute `"SLIDE"` |
| `deleteSlide()` | Delete a slide label & image. | `label` (set by Struts) | `"SUCCESS"` or `"INPUT"` | Deletes DB record and image file |
| `editSlide()` | Edit slide metadata & upload new image. | `label` (set by Struts), `uploadImage` | `"SUCCESS"` or `"INPUT"` | Persists label, uploads image |
| `deleteFile()` | Delete only slide image. | `label` (set by Struts) | `"SUCCESS"` or `"unauthorized"` | Removes file, updates DB |
| `getTemplates()`, `setTemplates()` | Getter/Setter for templates | None | Collection | |
| `getCurrrentTempate()`, `setCurrrentTempate()` | Getter/Setter for current template | None | CoreModuleService | |
| `getMetaDescription()`, `setMetaDescription()` | Getter/Setter | None | List<String> | |
| `getMetaKeywords()`, `setMetaKeywords()` | Getter/Setter | None | List<String> | |
| `getStoreFrontPageTitle()`, `setStoreFrontPageTitle()` | Getter/Setter | None | List<String> | |
| `getStoreDescription()`, `setStoreDescription()` | Getter/Setter | None | List<String> | |
| `getSliderConf()` | Getter | None | ModuleConfiguration | |
| `getSlideList()` | Getter for slide list (unused / missing variable) | None | String[] | |

**Reusable Methods**

* `prepareContent()` and `editStoreFontConfig()` contain repeated logic for iterating over languages and building label maps; these could be extracted into helper methods to reduce duplication.

---

## 4. Dependencies

| Dependency | Type | Usage |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String null/blank checks |
| `org.apache.log4j.Logger` | Third‑party | Logging |
| `com.salesmanager.central.*` | Project classes | Base action, context, utilities |
| `com.salesmanager.core.*` | Core module | Entities, services, constants |
| `com.salesmanager.core.util.*` | Core utilities | File, label, country utilities |
| `com.salesmanager.core.module.model.application.FileModule` | Spring bean | File upload/delete |
| `com.salesmanager.core.service.*` | Service layer | ReferenceService, MerchantService, ServiceFactory |
| `com.salesmanager.core.util.SpringUtil` | Spring helper | Bean lookup |

*All external dependencies are standard in the SalesManager stack (core, central, commons‑lang, log4j). No platform‑specific APIs are used.*

---

## 5. Additional Notes

### Strengths
* Clear separation of concerns: UI preparation vs. business logic.
* Good use of service layer to abstract persistence.
* Validation of ownership before deleting a file.
* Utilization of internationalization helpers (`LabelUtil`, `CountryUtil`).

### Potential Issues / Edge Cases
1. **Raw Types & Type Safety**  
   * Many collections (`Collection`, `Map`) are used without generics, leading to unchecked casts. This can cause `ClassCastException` at runtime and makes code harder to maintain.

2. **Missing Field (`visible`)**  
   * `getSlideList()` returns `visible`, but this field is never defined in the class. Likely a copy‑paste error that will cause a compilation error.

3. **Typos**  
   * `currrentTempate` (double “rrr”) and `reflanguages` (assumed to be a list of language IDs but used like a map). These naming inconsistencies make the code brittle.

4. **Null Safety**  
   * In `deleteSlide()` the variable `label` is used before checking if it’s null. If the request does not provide a label, a `NullPointerException` will be thrown.  
   * Similar risk exists in `deleteFile()` where `label.getImage()` is accessed without null‑check.

5. **Exception Propagation in `editSlide()`**  
   * The method declares `throws Exception` but also catches and rethrows inside. This is redundant; the method could simply let the exception bubble up.

6. **Hard‑coded Strings**  
   * The action uses magic strings like `"core.bin.images"`, `"errors.filetoolarge"`, and label keys. Extracting them into constants would improve maintainability.

7. **Security**  
   * Deleting or editing slides only checks that the label’s `merchantId` matches the context. No CSRF protection is visible here; the parent framework may provide it, but it should be confirmed.

8. **File Path Construction**  
   * In `deleteFile()` a `StringBuffer` is used to concatenate paths; `Paths` or `File` constructors would be safer and clearer.

9. **Resource Leaks**  
   * File uploads/downloads use the `FileModule` bean, but there’s no explicit cleanup of temporary files. Ensure that the underlying implementation handles this.

10. **Internationalization**  
    * The code manually maps language IDs to descriptions. If the number of languages grows, this can become inefficient. Consider caching the label maps or delegating to the `LabelUtil`.

### Suggested Enhancements
* Refactor to use **generics** everywhere.  
* Extract repeated label‑processing logic into private helper methods.  
* Replace raw `Map`/`Collection` with typed `Map<Integer, DynamicLabelDescription>` and `List<DynamicLabel>` where appropriate.  
* Add unit tests for the core logic (label creation, deletion, slide ordering).  
* Use Java 8+ streams for concise collection handling.  
* Add defensive checks for `null` and invalid inputs.  
* Consider using the Spring `FileSystemResource` or `Path` APIs for file operations.  
* Document the expected input formats for the `label` object (e.g., form fields mapping).  
* Implement CSRF token validation if not handled elsewhere.  

Overall, the action fulfills its responsibilities but would benefit from modernization (generics, helper utilities, clearer naming) and additional defensive programming to avoid runtime failures in edge scenarios.

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

import java.io.File;
import java.util.ArrayList;
import java.util.Collection;
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
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.util.FileException;
import com.salesmanager.central.web.DynamicImage;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.entity.reference.DynamicLabelDescriptionId;
import com.salesmanager.core.entity.reference.ModuleConfiguration;
import com.salesmanager.core.module.model.application.FileModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.SpringUtil;

public class StoreFrontAction extends ContentAction {

	private Logger log = Logger.getLogger(StoreFrontAction.class);
	private Collection templates = new ArrayList();// store front templates

	private CoreModuleService currrentTempate = null;

	private List<String> storeDescription = new ArrayList<String>();// store
																	// text
																	// submited
	private List<String> storeFrontPageTitle = new ArrayList<String>();// text submited

	private List<String> metaKeywords = new ArrayList<String>();// text submited
	private List<String> metaDescription = new ArrayList<String>();// text
																	// submited

	
	
	ModuleConfiguration sliderConf = null;
	




	private void prepareContent() throws Exception {

		super.setPageTitle("label.storesetup");
		
		Context ctx = super.getContext();

		String countryCode = CountryUtil.getCountryIsoCodeById(ctx
				.getCountryid());

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		templates = rservice.getCoreModules(
				CatalogConstants.STORE_FRONT_TEMPLATES_CODE, countryCode);

		// overwrite name
/*		if (templates != null && templates.size() > 0) {
			Iterator i = templates.iterator();
			while (i.hasNext()) {
				CoreModuleService cms = (CoreModuleService) i.next();
				try {
					String title = LabelUtil.getInstance().getText(
							ctx.getLang(),
							"module." + cms.getCoreModuleName() + ".title");
					cms.setCoreModuleServiceDescription(title);
				} catch (Exception e) {
					log.error(e);
				}
			}
		}*/

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		// get current template
		MerchantStore store = mservice.getMerchantStore(ctx.getMerchantid());

		String templateModule = store.getTemplateModule();
		// selected
		if (!StringUtils.isBlank(templateModule)) {

			currrentTempate = rservice.getCoreModuleService(ctx.getLang(),
					templateModule);
			if (currrentTempate != null) {
				currrentTempate.setCoreModuleServiceDescription(templateModule);
			}

		}
		
		//get module configuration slider for current template module
		sliderConf = rservice.getModuleConfiguration(store.getTemplateModule(), ConfigurationConstants.SLIDER_CONFIGURATION_KEY, Constants.ALLCOUNTRY_ISOCODE);

		Collection<DynamicLabel> dynamicLabels = rservice
				.getDynamicLabels(super.getContext().getMerchantid().intValue());

		if (dynamicLabels != null && dynamicLabels.size() > 0) {

			Iterator i = dynamicLabels.iterator();

			while (i.hasNext()) {

				DynamicLabel dl = (DynamicLabel) i.next();

				Set dynamicLabelSet = dl.getDescriptions();
				Iterator labelSetIterator = dynamicLabelSet.iterator();

				if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_DESCRIPTION) {

					Map labelMap = new HashMap();

					while (labelSetIterator.hasNext()) {
						DynamicLabelDescription description = (DynamicLabelDescription) labelSetIterator
								.next();
						labelMap.put(description.getId().getLanguageId(),
								description);
					}

					for (int icount = 0; icount < reflanguages.size(); icount++) {
						int langid = (Integer) reflanguages.get(icount);
						DynamicLabelDescription desc = (DynamicLabelDescription) labelMap
								.get(langid);
						if (desc != null) {
							storeDescription.add(desc
									.getDynamicLabelDescription());
						}
					}
				}

				else if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_PAGE_TITLE) {

					Map labelMap = new HashMap();

					while (labelSetIterator.hasNext()) {
						DynamicLabelDescription description = (DynamicLabelDescription) labelSetIterator
								.next();
						labelMap.put(description.getId().getLanguageId(),
								description);
					}

					for (int icount = 0; icount < reflanguages.size(); icount++) {
						int langid = (Integer) reflanguages.get(icount);
						DynamicLabelDescription desc = (DynamicLabelDescription) labelMap
								.get(langid);
						if (desc != null) {
							storeFrontPageTitle.add(desc.getDynamicLabelDescription());
						}
					}
				}
				if (dl.getSectionId() == LabelConstants.SLIDER_SECTION) {
					if(sliderConf!=null) {
						if(pages==null) {
							pages = new ArrayList();
						}
						pages.add(dl);
					}
				}

				if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_META_KEYWORDS) {

					Map labelMap = new HashMap();

					while (labelSetIterator.hasNext()) {
						DynamicLabelDescription description = (DynamicLabelDescription) labelSetIterator
								.next();
						labelMap.put(description.getId().getLanguageId(),
								description);
					}

					for (int icount = 0; icount < reflanguages.size(); icount++) {
						int langid = (Integer) reflanguages.get(icount);
						DynamicLabelDescription desc = (DynamicLabelDescription) labelMap
								.get(langid);
						if (desc != null) {
							metaKeywords.add(desc.getDynamicLabelDescription());
						}
					}

				}

				if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_META_DESCRIPTION) {

					Map labelMap = new HashMap();

					while (labelSetIterator.hasNext()) {
						DynamicLabelDescription description = (DynamicLabelDescription) labelSetIterator
								.next();
						labelMap.put(description.getId().getLanguageId(),
								description);
					}

					for (int icount = 0; icount < reflanguages.size(); icount++) {
						int langid = (Integer) reflanguages.get(icount);
						DynamicLabelDescription desc = (DynamicLabelDescription) labelMap
								.get(langid);
						if (desc != null) {
							metaDescription.add(desc
									.getDynamicLabelDescription());
						}
					}
				}
			}
		}
	}

	/**
	 * Displays the page allowing basic store front configuration
	 * 
	 * @return
	 */
	public String displayStoreFrontConfig() {

		try {

			Context ctx = super.getContext();

			prepareLanguages();

			prepareContent();


		} catch (Exception e) {
			log.error(e);
		}
		return SUCCESS;

	}

	public String editStoreFontConfig() {

		try {

			prepareLanguages();

			prepareContent();

			Context ctx = super.getContext();

			if (this.reflanguages.size() == 0) {
				log.error("Laanguages were not loaded");
				super.setTechnicalMessage();
				return INPUT;
			}

			// retreive current values
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			Collection<DynamicLabel> dynamicLabels = rservice
					.getDynamicLabels(super.getContext().getMerchantid()
							.intValue());

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			Map submited = new HashMap();

			if (this.getStoreDescription().size() > 0) {
				submited.put(LabelConstants.STORE_FRONT_LANDING_DESCRIPTION,
						this.getStoreDescription());
			}

			if (this.getStoreFrontPageTitle().size() > 0) {
				submited.put(LabelConstants.STORE_FRONT_LANDING_PAGE_TITLE,
						this.getStoreFrontPageTitle());
			}

			if (this.getMetaKeywords().size() > 0) {
				submited.put(LabelConstants.STORE_FRONT_LANDING_META_KEYWORDS,
						this.getMetaKeywords());
			}

			if (this.getMetaDescription().size() > 0) {
				submited.put(
						LabelConstants.STORE_FRONT_LANDING_META_DESCRIPTION,
						this.getMetaDescription());
			}

			if (dynamicLabels != null && dynamicLabels.size() > 0) {

				Collection removable = new ArrayList();

				Iterator i = dynamicLabels.iterator();

				while (i.hasNext()) {

					DynamicLabel dl = (DynamicLabel) i.next();
					if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_DESCRIPTION) {
						removable.add(dl);
					}

					else if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_PAGE_TITLE) {
						removable.add(dl);
					}

					if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_META_KEYWORDS) {
						removable.add(dl);
					}

					if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_META_DESCRIPTION) {
						removable.add(dl);
					}

				}

				rservice.deleteAllDynamicLabel(removable);
			}

			Map newLabels = new HashMap();

			Map elements = new HashMap();

			Iterator submitedIterator = submited.keySet().iterator();
			while (submitedIterator.hasNext()) {
				int section = (Integer) submitedIterator.next();

				List valuesSubmited = (List) submited.get(section);

				Iterator valuesSubmitedIterator = valuesSubmited.iterator();

				Iterator i = reflanguages.keySet().iterator();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();

					String desc = (String) valuesSubmited.get(langcount);

					// if not blank
					if (!StringUtils.isBlank(desc)) {

						DynamicLabel label = null;

						int submitedlangid = (Integer) reflanguages
								.get(langcount);

						if (!newLabels.containsKey(section)) {
							label = new DynamicLabel();
							newLabels.put(section, label);
						} else {
							label = (DynamicLabel) newLabels.get(section);
						}
						// create
						DynamicLabelDescriptionId id = new DynamicLabelDescriptionId();
						id.setLanguageId(submitedlangid);

						DynamicLabelDescription description = new DynamicLabelDescription();
						description.setId(id);
						description.setDynamicLabelDescription(desc);
						description.setDynamicLabelTitle(" ");

						Set descs = label.getDescriptions();
						if (descs == null) {
							descs = new HashSet();
						}

						descs.add(description);

						label.setMerchantId(ctx.getMerchantid());
						label.setSectionId(section);
						label.setDescriptions(descs);

					}

				}

			}


			rservice.saveDynamicLabel(newLabels.values());
			super.setSuccessMessage();

			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}

	}
	
	public String updateSlideList() {
		
		
		
		try {
		
			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			// get all slides
			Collection<DynamicLabel> labels = rservice.getDynamicLabels(super
					.getContext().getMerchantid(),LabelConstants.SLIDER_SECTION, super
					.getLocale());
			
			labels = super.updatePageList(labels);
			
			
			if(labels!=null) {
				rservice.saveDynamicLabel(labels);
				super.setSuccessMessage();
			}
			

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();

		}

		displayStoreFrontConfig();
		return SUCCESS;
	}
	
	public String viewSlide() {
		
		super.setPageTitle("label.storefront.slides.title");
		
		super.getPageDetails();
		
		DynamicLabel l = super.getLabel();
		
		if(l!=null) {
			//get image
			if (!StringUtils.isBlank(l.getImage())) {
				// set image info in the request
				DynamicImage img = new DynamicImage();
				img.setEntityId(String.valueOf(l.getDynamicLabelId()));
				img.setImageName(l.getImage());

				String imgPath = FileUtil.getFileTreeBinPathForImages(super.getContext().getMerchantid());
				img.setImagePath(imgPath);
				super.getServletRequest().setAttribute("SLIDE", img);
			}
		}	
		return SUCCESS;
	}
	
	public String deleteSlide() {
		
		try {
			ReferenceService rservice = (ReferenceService) ServiceFactory
			.getService(ServiceFactory.ReferenceService);
			DynamicLabel l = rservice.getDynamicLabel(label.getDynamicLabelId());
			rservice.deleteDynamicLabel(l);
			
			//delete image
			
			if(!StringUtils.isBlank(l.getImage())) {
			
				String imgfolder = FileUtil.getFileTreeBinPathForImages(super.getContext().getMerchantid());
				FileModule futil = (FileModule) SpringUtil.getBean("localfile");
				futil.deleteFile(super.getContext().getMerchantid(), new File(imgfolder + "/" + l.getImage()));
				
			}
			
			super.setSuccessMessage();
			
		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}
		
		
		return SUCCESS;
	}
	

	
	public String editSlide() throws Exception {
		
		super.setPageTitle("label.storefront.slides.title");
		try {
			
			
			super.prepareLanguages();
			DynamicImage img = null;
			
			if(label==null) {
				return SUCCESS;
			}
			
			boolean hasError = super.populateLabel();
			
			if(hasError) {
				return INPUT;
			}
			
			if (!StringUtils.isBlank(super.getUploadImageFileName())) {
			
				FileModule futil = (FileModule) SpringUtil
				.getBean("localfile");
				
				String finalfilename = futil.uploadFile(
				super.getContext().getMerchantid(), "core.bin.images", super.getUploadImage(), super.getUploadImageFileName(), super.getUploadImageContentType());
				
				
				super.getLabel().setImage(super.getUploadImageFileName());
				

	

			
			}
			
			if(!StringUtils.isBlank(super.getLabel().getImage())) {
				
				
				// set image info in the request
				img = new DynamicImage();
				img.setImageName(super.getLabel().getImage());

				String imgPath = FileUtil.getFileTreeBinPathForImages(super.getContext().getMerchantid());
				img.setImagePath(imgPath);
				super.getServletRequest().setAttribute("SLIDE", img);
				
			}
			
			ReferenceService rservice = (ReferenceService) ServiceFactory
			.getService(ServiceFactory.ReferenceService);
			rservice.saveOrUpdateDynamicLabel(super.getLabel());
			
			if(img!=null) {
				img.setEntityId(String.valueOf(super.getLabel().getDynamicLabelId()));
			}

			super.setSuccessMessage();
			
		} catch (Exception e) {
			
			if(e instanceof FileException) {
				
				super.setMessage("errors.filetoolarge");
				
			}
			
			throw(e);
		}
		

		
		return SUCCESS;
	}
	
	public String deleteFile() throws Exception {
		
		super.setPageTitle("label.storefront.slides.title");
		
		super.prepareLanguages();
		
		ReferenceService rservice = (ReferenceService) ServiceFactory
		.getService(ServiceFactory.ReferenceService);
		
		label = rservice.getDynamicLabel(super.getLabel().getDynamicLabelId());
		
		if(label==null) {
			throw new Exception ("label.dynamicLabelId is null");
		}
		
		if(label!=null && label.getMerchantId()!=super.getContext().getMerchantid()) {
			return "unauthorized";
		}
		
		FileModule futil = (FileModule) SpringUtil
		.getBean("localfile");
		
		String imgPath = FileUtil.getFileTreeBinPathForImages(super.getContext().getMerchantid());
		
		futil.deleteFile(super.getContext().getMerchantid(), new File(new StringBuffer()
		.append(imgPath).append(label.getImage()).toString()));
		
		label.setImage(null);
		
		rservice.saveOrUpdateDynamicLabel(label);
		
		super.setSuccessMessage();
		
		return SUCCESS;
	}

	public Collection getTemplates() {
		return templates;
	}

	public void setTemplates(Collection templates) {
		this.templates = templates;
	}

	public CoreModuleService getCurrrentTempate() {
		return currrentTempate;
	}

	public void setCurrrentTempate(CoreModuleService currrentTempate) {
		this.currrentTempate = currrentTempate;
	}

	public List<String> getMetaDescription() {
		return metaDescription;
	}

	public void setMetaDescription(List<String> metaDescription) {
		this.metaDescription = metaDescription;
	}

	public List<String> getMetaKeywords() {
		return metaKeywords;
	}

	public void setMetaKeywords(List<String> metaKeywords) {
		this.metaKeywords = metaKeywords;
	}

	public List<String> getStoreFrontPageTitle() {
		return storeFrontPageTitle;
	}

	public void setStoreFrontPageTitle(List<String> pageTitle) {
		this.storeFrontPageTitle = pageTitle;
	}

	public List<String> getStoreDescription() {
		return storeDescription;
	}

	public void setStoreDescription(List<String> storeDescription) {
		this.storeDescription = storeDescription;
	}
	


	public ModuleConfiguration getSliderConf() {
		return sliderConf;
	}
	
	public String[] getSlideList() {
		return visible;
	}

}



```
