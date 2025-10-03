# ContentAction.java

## Review

## 1. Summary  

`ContentAction` is a Struts‑2 action (extends `BaseAction`) that provides the common plumbing for managing “page content” entities in a merchant‑store application.  The core responsibilities are:  

| Responsibility | What it does | Key classes involved |
|----------------|--------------|----------------------|
| **Dynamic Label handling** | Loads, populates, and persists `DynamicLabel` objects that represent multi‑language page titles, descriptions, and URLs. | `DynamicLabel`, `DynamicLabelDescription`, `ReferenceService` |
| **Image upload validation** | Validates the MIME type and size of an uploaded image before it is persisted. | `uploadImage*` fields, `imgctypes` map |
| **Visibility handling** | Marks a list of pages as visible or hidden based on user selection. | `visible` array, `DynamicLabel` `setVisible()` |
| **Localization support** | Prepares language lists and orders content fields according to those languages. | `reflanguages` map (inherited from `BaseAction`) |

The class uses a **static initializer** to read configuration values (max file size, allowed MIME types) from a properties file via `org.apache.commons.configuration`.  Logging is done with **Log4j**.

---

## 2. Detailed Description  

### 2.1 Core Data Structures  

| Field | Type | Purpose |
|-------|------|---------|
| `titles` / `descriptions` | `List<String>` | Stores title/description values for each language. |
| `sefurl` | `List<String>` | Stores SEO‑friendly URLs. |
| `label` | `DynamicLabel` | The entity being created/edited. |
| `pages` | `Collection<DynamicLabel>` | List of all page entities for display. |
| `imgctypes` | `Map` (raw) | Maps allowed image MIME types to themselves for quick lookup. |
| `maximagesize` / `maxfilesize` | `long` | Max allowed sizes read from config. |
| `visible` | `String[]` | IDs of pages selected to be visible. |
| `uploadImage*` | `String`, `File` | Staged image upload details. |

### 2.2 Initialization Flow  

```java
static {
    // read config values
    // parse sizes
    // populate imgctypes
}
```

The block is executed once per JVM when the class is first loaded.  It assumes that `conf` is a thread‑safe singleton from `PropertiesUtil`.  However, `imgctypes` is a raw `HashMap`; this leads to unchecked warnings and possible `ClassCastException` if the map is used elsewhere.

### 2.3 Runtime Behavior  

1. **Action Invocation** – `ContentAction` is used as a base class for concrete actions (e.g. `EditPageAction`).  
2. **Language Preparation** – `super.prepareLanguages()` is called inside `getPageDetails()` to load the list of supported languages into the inherited `reflanguages` map.  
3. **Page Retrieval / Update** –  
   * `getPageDetails()` fetches the current `DynamicLabel` from the reference service, builds a language‑ordered list of descriptions, and populates the `descriptions` list.  
   * `populateLabel()` is invoked (likely from `execute()` in a subclass) to convert the user‑supplied lists into `DynamicLabelDescription` objects and set them on the `label`.  
   * During this conversion it also validates an image upload (MIME type + size).  
4. **Visibility Update** – `updatePageList()` iterates over all pages, checks whether each page’s ID is present in the `visible` array, and sets `visible` accordingly.  
5. **Persistence** – Not shown in this class; a subclass would call the reference service to persist the modified `label`.  

### 2.4 Clean‑up  

No explicit cleanup is required because file uploads are handled by Struts2’s built‑in `File` upload interceptor.  The class does not keep open resources beyond that.

### 2.5 Design Choices  

| Choice | Rationale | Critique |
|--------|-----------|----------|
| **Static config loading** | Simplifies access to constants across the application. | Thread‑safe, but `imgctypes` is mutable and shared; potential for accidental modification. |
| **Use of raw types** | Legacy code style. | Modern Java best practice is to use generics; raw types lead to unchecked warnings. |
| **String arrays for IDs** | Mirrors form parameter binding (`visible`). | String conversion + parsing adds runtime overhead and error risk. |
| **Returning `null` from `updatePageList()`** | Indicates “nothing to update”. | Returning the original collection (possibly empty) is more predictable for callers. |
| **`populateLabel()` returns a boolean error flag** | Simplifies validation flow. | The method name suggests side‑effects; better to rename or split into `populateLabel()` and `validateUpload()` for clarity. |

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `populateLabel()` | Builds `DynamicLabelDescription` objects from `titles` & `descriptions`; validates an uploaded image. | None (uses fields). | `boolean` – `true` if an error was detected. | Adds field errors to the action; mutates `label`’s description set. |
| `getVisible() / setVisible(String[])` | Getter/Setter for selected page IDs. | `String[]` | `String[]` | None |
| `getTitles() / setTitles(List<String>)` | Getter/Setter for titles. | `List<String>` | `List<String>` | None |
| `getDescriptions() / setDescriptions(List<String>)` | Getter/Setter for descriptions. | `List<String>` | `List<String>` | None |
| `getSefurl() / setSefurl(List<String>)` | Getter/Setter for SEO URLs. | `List<String>` | `List<String>` | None |
| `getLabel() / setLabel(DynamicLabel)` | Getter/Setter for the current label. | `DynamicLabel` | `DynamicLabel` | None |
| `getPages() / setPages(Collection<DynamicLabel>)` | Getter/Setter for all pages. | `Collection<DynamicLabel>` | `Collection<DynamicLabel>` | None |
| `updatePageList(Collection<DynamicLabel>)` | Sets each page’s visibility flag based on `visible`. | `Collection<DynamicLabel>` | `Collection<DynamicLabel>` (or `null`) | Mutates each `DynamicLabel`’s `visible` flag. |
| `getPageDetails()` | Loads current label from DB, populates language‑ordered description list. | None | None | Mutates `label` and `descriptions`; may add technical error. |
| `getUploadImageFileName()` / `setUploadImageFileName(String)` | File upload getter/setter. | `String` | `String` | None |
| `getUploadImageContentType()` / `setUploadImageContentType(String)` | MIME type getter/setter. | `String` | `String` | None |
| `getUploadImage()` / `setUploadImage(File)` | Uploaded file getter/setter. | `File` | `File` | None |

*Reusable utility methods*:  
- None explicitly, but `populateLabel()` contains generic validation logic that could be extracted.

---

## 4. Dependencies  

| Library | Version (assumed) | Role |
|---------|-------------------|------|
| **Apache Commons Configuration** | `commons-configuration` | Reads config properties in the static block. |
| **Apache Commons Lang** | `StringUtils` | Checks for blank strings. |
| **Log4j** | `org.apache.log4j.Logger` | Logging of errors and debug info. |
| **Struts 2** | `org.apache.struts2.interceptor.PrincipalProxy` | Action base, file upload handling. |
| **Custom Core** | `com.salesmanager.core.*` | Entity and service classes (`DynamicLabel`, `ReferenceService`, `ServiceFactory`, `PropertiesUtil`). |
| **JDK** | `java.io`, `java.util` | Standard collections, IO, etc. |

All are third‑party libraries except the JDK.  No database or web container dependencies are declared directly; they are mediated through the `ReferenceService` and Struts2.

---

## 5. Additional Notes  

### 5.1 Code Quality & Maintainability  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (`Map imgctypes = new HashMap();`, `Set descs = label.getDescriptions();`) | Unchecked casts, potential `ClassCastException`. | Use generics: `Map<String, String> imgctypes = new HashMap<>();`, `Set<DynamicLabelDescription> descs = label.getDescriptions();` |
| **Hard‑coded strings** (e.g. `"false"` sentinel) | Fragile; prone to bugs if changed elsewhere. | Use a boolean flag or an enum; store selected IDs as longs. |
| **Static mutable state** (`imgctypes`, `maximagesize`, `maxfilesize`) | Thread‑safety risk if modified elsewhere. | Make them `final` after init or use immutable collections. |
| **Error handling** (returning `boolean` in `populateLabel()`) | Confusing contract. | Separate validation into its own method that throws or returns a result object. |
| **No generics for collections** (`List sefurl` etc.) | Unclear element type; potential unchecked warnings. | Add generics everywhere. |
| **StringTokenizer** | Legacy; can be replaced by `String.split`. | Use `ctlist.split(";")`. |
| **Logging of sensitive data** | `uploadImageFileName` might expose PII. | Ensure logging only logs safe data. |
| **Missing null‑check for `reflanguages`** | Could cause NPE if not initialized. | Guard against null. |

### 5.2 Edge Cases Not Handled  

1. **Empty or missing `titles` list** – `getTitles()` may be `null`; `populateLabel()` assumes a list with size >0.  
2. **`visible` contains non‑numeric strings other than `"false"`** – parsing fails silently, defaulting to `false`.  
3. **Large number of pages** – iterating over all pages for visibility could become a performance bottleneck.  
4. **Duplicate language IDs** – `reflanguages` may contain duplicates; resulting descriptions may be overwritten.  
5. **Concurrent requests** – static config fields are immutable, but `imgctypes` is a raw mutable map; potential for accidental modifications.  

### 5.3 Potential Enhancements  

| Area | Idea |
|------|------|
| **Validation framework** | Integrate Struts2 validation XML or Java annotations to separate validation from business logic. |
| **Service layer** | Move `populateLabel()` logic into a service (e.g. `LabelService`) for easier unit testing. |
| **File handling** | Use a dedicated file upload service that validates MIME type and size, stores the file, and returns a reference. |
| **Internationalization** | Store language IDs and titles in a separate `Language` entity for clearer mapping. |
| **Unit tests** | Add tests for `populateLabel()`, `updatePageList()`, and `getPageDetails()`. |
| **Configuration** | Use Spring `@ConfigurationProperties` or a dedicated config class instead of static initializer. |
| **Use of Streams** | Replace loops with Java 8 Streams for readability. |
| **Immutable DTOs** | Return immutable DTOs to callers instead of exposing mutable collections. |

---

### 5.4 Summary of Strengths  

* The code demonstrates a clear separation of concerns between data population, validation, and visibility management.  
* It leverages established libraries (Apache Commons, Log4j, Struts2) and follows the typical Struts2 action pattern.  
* Configuration values are centralized in `PropertiesUtil`, making it easy to change limits without code changes.

### 5.5 Summary of Weaknesses  

* Heavy reliance on legacy, raw‑type Java collections.  
* Mixed responsibilities in a single method (`populateLabel()` does both building the entity and validation).  
* Lack of modern Java features (generics, streams, immutability).  
* No defensive programming around potentially null or malformed input.  
* Static mutable state that could be problematic in a highly concurrent environment.

---  

**Recommendation**: Refactor the class to adopt generics, split responsibilities into smaller, testable methods or services, and replace legacy constructs with modern Java best practices.  This will improve type safety, maintainability, and testability while preserving the current functionality.

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
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.StringTokenizer;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.apache.struts2.interceptor.PrincipalProxy;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.catalog.EditProductAction;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.entity.reference.DynamicLabelDescriptionId;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.PropertiesUtil;


/**
 * Common content attributes
 * @author Carl Samson
 *
 */
public abstract class ContentAction extends BaseAction {
	
	
	/**
	 * 
	 */
	

	private static final long serialVersionUID = -2458466574834932255L;
	protected List<String> titles = new ArrayList<String>();
	protected List<String> descriptions = new ArrayList<String>();
	
	// image upload
	private String uploadImageFileName;
	private String uploadImageContentType;
	private File uploadImage;
	private static long maximagesize;
	private static long maxfilesize;
	private static Map imgctypes = new HashMap();
	
	private static Logger log = Logger.getLogger(ContentAction.class);
	
	String[] visible;// selected page content
	
	
	protected List<String> sefurl = new ArrayList<String>();
	//protected String title;//unique identifier
	
	protected DynamicLabel label = null;

	protected Collection<DynamicLabel> pages = null;
	
	private static Configuration conf = PropertiesUtil.getConfiguration();
	
	static {

		String smaxfsize = conf.getString("core.product.image.maxfilesize");
		if (smaxfsize == null) {
			log
					.error("Properties core.product.image.maxfilesize not defined in config.properties");
			smaxfsize = "100000";
		}
		long maxsize = 0;
		try {
			maxsize = Long.parseLong(smaxfsize);

		} catch (Exception e) {
			log
					.error("Properties core.product.image.maxfilesize not an integer");
			maxsize = 100000;
		}

		maximagesize = maxsize;

		smaxfsize = conf.getString("core.product.file.maxfilesize");
		if (smaxfsize == null) {
			log
					.error("Properties core.product.file.maxfilesize not defined in config.properties");
			smaxfsize = "8000000";
		}
		try {
			maxsize = Long.parseLong(smaxfsize);

		} catch (Exception e) {
			log
					.error("Properties core.product.file.maxfilesize not an integer");
			maxsize = 100000;
		}

		String ctlist = conf.getString("core.product.image.contenttypes");

		if (ctlist == null) {
			log.error("No content types defined for images");
		} else {

			StringTokenizer st = new StringTokenizer(ctlist, ";");
			while (st.hasMoreTokens()) {
				String ct = (String) st.nextToken();
				imgctypes.put(ct, ct);
			}
		}
		maxfilesize = maxsize;
	}
	
	public boolean populateLabel() {
		
		
		
		boolean hasError = false;
		
		
		if(label!=null) {
		
			Iterator i = reflanguages.keySet().iterator();
			while (i.hasNext()) {
				int langcount = (Integer) i.next();
	
				String description = (String) this.getDescriptions().get(
						langcount);
				String title = "";
				if(this.getTitles()!=null && this.getTitles().size()>0) {
				      title = (String)this.getTitles().get(
						langcount);
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
				
				dldescription.setDynamicLabelTitle("--");
				if(!StringUtils.isBlank(title)) {
					dldescription.setDynamicLabelTitle(title);
				}
	
	
				Set descs = label.getDescriptions();
				if (descs == null) {
					descs = new HashSet();
				}
	
				descs.add(dldescription);
	
				label.setMerchantId(super.getContext().getMerchantid());
				label.setDescriptions(descs);
	
			}
			
			
			// image upload validation
			if (!StringUtils.isBlank(this.getUploadImageContentType())
					&& !StringUtils.isBlank(this.getUploadImageFileName())) {
				String ct = this.getUploadImageContentType();
	
				if (!imgctypes.containsKey(ct)) {
					super
							.addFieldError(
									"uploadimage",
									getText("error.message.product.image.invalidfiletype")
											+ " "
											+ getText("label.product.uploadimage"));
					hasError = true;
				}
			}
	
			if (this.getUploadImage() != null
					&& !StringUtils.isBlank(this.getUploadImageFileName())) {
				java.io.File f = this.getUploadImage();
	
				if (f.length() > this.maximagesize) {
	
					super.addFieldError("uploadimage",
							getText("error.message.product.image.file") + " "
									+ getText("label.product.uploadimage"));
					hasError = true;
	
				}
			}
		}
		return hasError;
		
	}
	
	
	

	public String[] getVisible() {
		return visible;
	}

	public void setVisible(String[] visible) {
		this.visible = visible;
	}

	
	
	public List<String> getTitles() {
		return titles;
	}

	public void setTitles(List<String> titles) {
		this.titles = titles;
	}

	public List<String> getDescriptions() {
		return descriptions;
	}

	public void setDescriptions(List<String> descriptions) {
		this.descriptions = descriptions;
	}

	public List<String> getSefurl() {
		return sefurl;
	}

	public void setSefurl(List<String> sefurl) {
		this.sefurl = sefurl;
	}



	public DynamicLabel getLabel() {
		return label;
	}

	public void setLabel(DynamicLabel label) {
		this.label = label;
	}

	public Collection<DynamicLabel> getPages() {
		return pages;
	}

	public void setPages(Collection<DynamicLabel> pages) {
		this.pages = pages;
	}

	
	protected Collection<DynamicLabel> updatePageList(Collection<DynamicLabel> pages) {
		
		
		if (pages != null && pages.size()>0) {
			
			for (Object o : pages) {
		
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
		
			return pages;
		
		}
		
		return null;

		
	}
	

	protected void getPageDetails() {

		try {

				
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

		
	}




	public String getUploadImageFileName() {
		return uploadImageFileName;
	}




	public void setUploadImageFileName(String uploadImageFileName) {
		this.uploadImageFileName = uploadImageFileName;
	}




	public String getUploadImageContentType() {
		return uploadImageContentType;
	}




	public void setUploadImageContentType(String uploadImageContentType) {
		this.uploadImageContentType = uploadImageContentType;
	}




	public File getUploadImage() {
		return uploadImage;
	}




	public void setUploadImage(File uploadImage) {
		this.uploadImage = uploadImage;
	}







}



```
