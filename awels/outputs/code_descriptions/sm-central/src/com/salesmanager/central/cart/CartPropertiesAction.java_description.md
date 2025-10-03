# CartPropertiesAction.java

## Review

## 1. Summary  
**Purpose** – `CartPropertiesAction` is a Struts 2 action that lets a merchant view, edit, and delete the branding assets (logo, banner, and other cart‑related properties) of their online store. It also loads and persists the `MerchantStore` entity and its associated configuration objects.

**Key Components**  
| Component | Role |
|-----------|------|
| `BaseAction` | Provides common functionality (request/response handling, messaging, etc.) |
| `Preparable` | Allows the action to perform initialization before the actual request method is invoked |
| `MerchantService` | Service layer used to read/write `MerchantStore` and `MerchantConfiguration` entities |
| `FileModule` | Abstraction for file upload/delete operations (Spring bean `localfile`) |
| `DynamicImage` | Lightweight DTO used to pass image metadata to the view layer |
| `Configuration` (Apache Commons) | Reads static application properties (mostly legacy code) |

**Design Patterns & Frameworks**  
* **MVC** – Struts 2 action acts as the controller, delegating business logic to services.  
* **Service Layer** – `MerchantService` hides persistence details.  
* **Dependency Injection** – Spring is used to inject the `FileModule` bean.  
* **Factory** – `ServiceFactory` provides the service instance (a simple static factory).  
* **Utility helpers** – `FileUtil`, `CurrencyUtil`, `DateUtil`, etc., are static helpers that encapsulate common logic.

---

## 2. Detailed Description  

### Execution Flow

| Phase | Method | What Happens |
|-------|--------|--------------|
| **Preparation** | `prepare()` | Sets page title, clears old image attributes, and loads language data. |
| **Display** | `display()` | Retrieves the merchant store, ensures default values for colors, domain, and continue‑shopping URL, sets `DynamicImage` objects for the logo and banner in the request, and fetches the sitemap configuration. |
| **Delete Image** | `deleteImage()` | Deletes the entire branding folder for the merchant (header logo/banner) and clears the `storelogo` field. |
| **Delete File** | `deleteFile()` | Deletes a single file (logo or banner) based on `imageId`. |
| **Save** | `saveCart()` | Persists changes to the merchant store, uploads new logo/banner files, and updates the UI with the new images. |

### Initialization & Cleanup  
* `prepare()` removes any residual `BANNER`/`LOGO` attributes from the session to avoid stale data.  
* No explicit cleanup is required on shutdown; however, file deletion is performed in `deleteImage()` and `deleteFile()`.

### Dependencies & Constraints  
* **Thread‑safety** – The action is instantiated per request by Struts 2, so instance fields are safe.  
* **Static `conf`** – Loaded once at class load time; not refreshed if properties change.  
* **File Paths** – Built by concatenating strings; no OS‑independent `File` handling.  
* **Error Handling** – Errors are caught and a generic technical message is shown; detailed logging is performed.  
* **Assumptions** –  
  * The merchant’s ID is stored in a session `ProfileConstants.context`.  
  * The `localfile` bean correctly implements the file system operations.  
  * Uploaded files are small enough to fit in memory (no stream handling).  

### Architecture Choices  
* The code mixes **business logic** (upload, delete) with **web concerns** (setting request attributes). A cleaner separation could be achieved by moving file operations into a dedicated service.  
* The use of `ServiceFactory` rather than Spring injection is legacy; modern code would inject `MerchantService` directly.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `prepare()` | Init before any action method | – | – | Sets title, clears image attrs, loads languages |
| `deleteImage()` | Remove all branding images for a merchant | – | `String` (success) | Deletes files, clears `storelogo`, saves store |
| `display()` | Show current cart branding configuration | – | `String` (success) | Loads store, sets request images, fetches sitemap config |
| `deleteFile()` | Delete a single logo or banner | – | `String` (success) | Removes file, clears corresponding store field |
| `saveCart()` | Persist store changes + handle uploads | – | `String` (success) | Updates store, uploads files, sets request images |
| Getters/Setters (e.g., `getUploadlogoFileName()`, `setUploadlogoFileName(...)`) | Standard POJO accessors | – | – | – |

### Notable Helpers  
* **`FileUtil.getBrandingFilePath()`** – centralises the root directory for branding assets.  
* **`DynamicImage`** – encapsulates image name & path for view rendering.  
* **`FileModule.uploadFile()` / `deleteFile()`** – abstracts file system operations.  

---

## 4. Dependencies  

| Library / Framework | Role | Standard / Third‑Party |
|---------------------|------|------------------------|
| `org.apache.commons.configuration.Configuration` | Reads static config | Third‑party |
| `org.apache.commons.lang.StringUtils` | String utilities | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `org.hibernate.HibernateException` | Persistence exceptions | Third‑party |
| `com.opensymphony.xwork2.Preparable` | Struts 2 lifecycle | Third‑party |
| `com.salesmanager.central.*` | Application core classes | Application |
| `com.salesmanager.core.*` | Core services/utilities | Application |
| `SpringUtil` | Spring bean lookup | Application |
| `FileModule` | File handling abstraction | Application |

*All dependencies are either part of the SalesManager core or classic Apache/Struts libraries.*

---

## 5. Additional Notes  

### Strengths  
* Clear separation of concerns at the action level (display vs. save).  
* Robust handling of missing values (e.g., default domain, color).  
* Use of utility classes reduces duplication (e.g., `FileUtil`).  

### Potential Issues / Edge Cases  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Hard‑coded directory paths** | OS‑independence & security risk | Use `File` and `Paths` API; validate paths against traversal. |
| **Duplicate code** (upload logic in `saveCart`) | Hard to maintain | Extract to a helper method or service. |
| **No file‑type validation** | Security – arbitrary file upload | Check MIME type and allowed extensions. |
| **No size validation** | Potential disk space exhaustion | Enforce max size before upload. |
| **Missing null check on `uploadlogo`** | NPE if file omitted | Guard against `null` before accessing. |
| **`imageId` string comparisons** | Possible `NullPointerException` | Validate `imageId` before use. |
| **`conf` static field** | Not refreshed if properties change | Consider reloading on context refresh or inject via Spring. |
| **Logging** | Sensitive data may be logged (file paths) | Mask or avoid logging full paths. |
| **Internationalisation** | Hard‑coded `MessageUtil.addErrorMessage` messages may lack localisation | Use `LabelUtil` for all messages. |

### Future Enhancements  

1. **Dependency Injection** – Replace `ServiceFactory` and `SpringUtil` lookups with constructor injection.  
2. **Unit Tests** – Mock `MerchantService` and `FileModule` to test action logic in isolation.  
3. **Transactional Handling** – Wrap `saveCart` in a Spring transaction for atomicity.  
4. **Async File Handling** – For large uploads, stream files instead of loading into memory.  
5. **Audit Trail** – Log who changed branding assets and when.  

---

**Overall Assessment**  
`CartPropertiesAction` is a functional, if somewhat dated, Struts 2 action that fulfills its business requirements. The code is readable and leverages existing utilities, but it could benefit from modernizing dependency injection, tightening file‑security checks, and refactoring duplicated upload logic. Addressing the highlighted edge cases would make the component more robust in a production environment.

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
package com.salesmanager.central.cart;

import java.io.File;
import java.util.Date;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.hibernate.HibernateException;

import com.opensymphony.xwork2.Preparable;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.web.DynamicImage;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.module.impl.application.files.FileException;
import com.salesmanager.core.module.model.application.FileModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.SpringUtil;

/**
 * Manage Shopping Cart properties
 * 
 * @author Carl Samson
 * 
 */
public class CartPropertiesAction extends BaseAction implements Preparable {

	private static final long serialVersionUID = 5156288255337069381L;
	private Logger log = Logger.getLogger(CartPropertiesAction.class);
	private String contentType;

	private String caption;

	private MerchantStore merchantStore;
	private static Configuration conf = PropertiesUtil.getConfiguration();

	// image banner
	private String uploadbannerfilename;
	private String uploadbannercontenttype;
	private File uploadbanner;

	// logo
	private File uploadlogo;
	private String uploadlogofilename;// uploadFileName
	private String uploadlogocontenttype;
	
	private MerchantConfiguration siteMapConfiguration = null;

	// for managing deleteFile
	private String imageId;

	// since we are using <s:file name="upload" .../> the file name will be
	// obtained through getter/setter of <file-tag-name>FileName
	public String getUploadlogoFileName() {
		return uploadlogofilename;
	}

	public void setUploadlogoFileName(String uploadlogofilename) {
		this.uploadlogofilename = uploadlogofilename;
	}

	public String getUploadlogoContentType() {
		return uploadlogocontenttype;
	}

	public void setUploadlogoContentType(String uploadlogocontenttype) {
		this.uploadlogocontenttype = uploadlogocontenttype;
	}

	public void prepare() {
		
		super.setPageTitle("label.branding.title");

		// need to cleanup image stuff
		super.getServletRequest().removeAttribute("BANNER");
		super.getServletRequest().removeAttribute("LOGO");
		super.prepareLanguages();
		
		

	}

	public String deleteImage() throws Exception {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice.getMerchantStore(merchantid);

			java.util.Date dt = new java.util.Date();

			if (store == null) {// create default values

				MerchantUserInformation userInformation = mservice
						.getMerchantUserInformation(super.getPrincipal()
								.getRemoteUser());

				store = new MerchantStore();
				store.setMerchantId(merchantid.intValue());
				store.setStorename("");
				store.setCountry(userInformation.getUsercountrycode());
				store.setZone("0");
				store.setCurrency(CurrencyUtil.getDefaultCurrency());
				store.setStoreaddress(userInformation.getUseraddress());
				store.setStorecity(userInformation.getUsercity());
				store.setStorepostalcode(userInformation.getUserpostalcode());
				store.setWeightunitcode(Constants.LB_WEIGHT_UNIT);
				store.setSeizeunitcode(Constants.INCH_SIZE_UNIT);
				store.setSupportedlanguages("en");
				store.setStorephone("");
			}

			store.setLastModified(new java.util.Date(dt.getTime()));
			store.setStorelogo("");

			Configuration config = PropertiesUtil.getConfiguration();
			//String imgfolder = config
			//		.getString("core.branding.cart.filefolder");
			String imgfolder = FileUtil.getBrandingFilePath();
			// Check if merchant directory exist
			String directory = imgfolder + "/"
					+ String.valueOf(merchantid.intValue());
			String headerdir = directory + "/header";

			FileModule futil = (FileModule) SpringUtil.getBean("localfile");
			futil.deleteFile(merchantid, new File(headerdir));

			mservice.saveOrUpdateMerchantStore(store);

			super.setSuccessMessage();

		} catch (Exception e) {
			super.setTechnicalMessage();
			log.error(e);

		}
		return SUCCESS;
	}

	/**
	 * Displays the shopping cart properties
	 * 
	 * @return
	 * @throws Exception
	 */
	public String display() throws Exception {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			merchantStore = mservice.getMerchantStore(merchantid);

			if (merchantStore.getBgcolorcode() == 0) {
				merchantStore.setBgcolorcode(new Integer(1));// force white
			}
			

			if(StringUtils.isBlank(merchantStore.getDomainName())) {
				
				
				String serverName = super.getServletRequest().getServerName();
				int serverPort = super.getServletRequest().getServerPort();
				
				if(serverPort>0) {
					serverName = serverName + ":" + String.valueOf(serverPort);
				}
				
				merchantStore.setDomainName(serverName);
				
				
			}
			
			if(StringUtils.isBlank(merchantStore.getContinueshoppingurl())) {
				merchantStore.setContinueshoppingurl(FileUtil.getDefaultCataloguePageUrl(merchantStore, super.getServletRequest()));
			}

			if (!StringUtils.isBlank(merchantStore.getStorelogo())) {

				// set image info in the request
				DynamicImage img = new DynamicImage();
				img.setImageName(merchantStore.getStorelogo());
				//String imgfolder = conf
				//		.getString("core.branding.cart.filefolder");
				String imgfolder = FileUtil.getBrandingFilePath();
				String imgpath = new StringBuffer().append(imgfolder).append(
						"/").append(merchantid.intValue()).append("/").append(
						"header/").toString();
				img.setImagePath(imgpath);
				super.getServletRequest().setAttribute("LOGO", img);

			}

			if (!StringUtils.isBlank(merchantStore.getStorebanner())) {
				DynamicImage img = new DynamicImage();
				img.setImageName(merchantStore.getStorebanner());
				//String imgfolder = conf
				//		.getString("core.branding.cart.filefolder");
				String imgfolder = FileUtil.getBrandingFilePath();
				String imgpath = new StringBuffer().append(imgfolder).append(
						"/").append(merchantid.intValue()).append("/").append(
						"header/").toString();
				img.setImagePath(imgpath);
				super.getServletRequest().setAttribute("BANNER", img);// Dynamic
																		// Image
			}
			
			
			//sitemap
			
			ConfigurationRequest req = new ConfigurationRequest(merchantStore.getMerchantId(),ConfigurationConstants.SITEMAP);
			ConfigurationResponse resp = mservice.getConfiguration(req);
			if(resp!=null) {
				siteMapConfiguration = resp.getMerchantConfiguration(ConfigurationConstants.SITEMAP);
			}
			
			System.out.println("done");


		} catch (HibernateException e) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error(e);
		}
		return SUCCESS;

	}

	public String getCaption() {
		return caption;
	}

	public void setCaption(String caption) {
		this.caption = caption;
	}

	public String input() throws Exception {
		return SUCCESS;
	}

	/**
	 * Deletes logo and banner
	 * 
	 * @return
	 * @throws Exception
	 */
	public String deleteFile() throws Exception {

		try {

			Context ctx = super.getContext();

			if (!StringUtils.isBlank(this.getImageId())) {

				String headerdir = "";

				MerchantService mservice = (MerchantService) ServiceFactory
						.getService(ServiceFactory.MerchantService);
				MerchantStore mstore = mservice.getMerchantStore(ctx
						.getMerchantid());

				if (this.getImageId().equals("0")) {// banner
					
					String banner = mstore.getStorebanner();
					mstore.setStorebanner(null);

					//String imgfolder = conf
					//		.getString("core.branding.banner.filefolder");
					String imgfolder = FileUtil.getBrandingFilePath();
					
					String directory = imgfolder + "/"
							+ String.valueOf(ctx.getMerchantid());

					headerdir = directory + "/header/" + banner;
					

				} else if (this.getImageId().equals("1")) {// logo

					//String imgfolder = conf
					//		.getString("core.branding.cart.filefolder");
					
					String logo = mstore.getStorelogo();
					
					String imgfolder = FileUtil.getBrandingFilePath();
					String directory = imgfolder + "/"
							+ String.valueOf(ctx.getMerchantid());
					headerdir = directory + "/header/" + logo;

					mstore.setStorelogo(null);

				}

				FileModule futil = (FileModule) SpringUtil.getBean("localfile");
				futil.deleteFile(ctx.getMerchantid(), new File(headerdir));

				mservice.saveOrUpdateMerchantStore(mstore);
			}

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	public String saveCart() throws Exception {

		try {

			Context ctx = super.getContext();

			// get current store
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice
					.getMerchantStore(ctx.getMerchantid());

			java.util.Date dt = new java.util.Date();
			// boolean update = false;

			if (store == null) {

				MerchantUserInformation userInformation = mservice
						.getMerchantUserInformation(super.getPrincipal()
								.getRemoteUser());

				// Configure default values
				store = new MerchantStore();
				store.setMerchantId(ctx.getMerchantid());
				store.setStorename("");
				store.setCountry(userInformation.getUsercountrycode());
				store.setZone("0");
				store.setCurrency(CurrencyUtil.getDefaultCurrency());
				store.setStoreaddress(userInformation.getUseraddress());
				store.setStorecity(userInformation.getUsercity());
				store.setStorepostalcode(userInformation.getUserpostalcode());
				store.setWeightunitcode(Constants.LB_WEIGHT_UNIT);
				store.setSeizeunitcode(Constants.INCH_SIZE_UNIT);
				store.setSupportedlanguages("en");
				store.setStorephone("");
			}

			store.setLastModified(new java.util.Date(dt.getTime()));
			store.setBgcolorcode(merchantStore.getBgcolorcode());
			store
					.setContinueshoppingurl(merchantStore
							.getContinueshoppingurl());
			store.setDefaultLang(merchantStore.getDefaultLang());

			if (!StringUtils.isBlank(this.getUploadlogoFileName())) {

				// now copy the logo to the destination
				File f = this.getUploadlogo();

				try {

					FileModule futil = (FileModule) SpringUtil
							.getBean("localfile");
					String finalfilename = futil.uploadFile(
							ctx.getMerchantid(), "core.branding.cart", f, this
									.getUploadlogoFileName(), this
									.getUploadlogoContentType());

					store.setStorelogo(this.getUploadlogoFileName());

					DynamicImage img = new DynamicImage();
					img.setImageName(this.getUploadlogoFileName());

					// log.debug("Image path " + finalfilename);
					img.setImagePath(finalfilename);
					super.getServletRequest().setAttribute("LOGO", img);
				} catch (FileException e) {
					if (e.getType() == FileException.USER) {
						MessageUtil.addErrorMessage(super.getServletRequest(),
								e.getMessage());
					} else {
						log.error(e);
						MessageUtil.addErrorMessage(super.getServletRequest(),
								LabelUtil.getInstance().getText(
										"errors.technical"));
					}
					return SUCCESS;
				}

			}

			if (!StringUtils.isBlank(this.getUploadbannerFileName())) {

				// now copy the logo to the destination
				File f = this.getUploadbanner();

				try {

					FileModule futil = (FileModule) SpringUtil
							.getBean("localfile");
					String finalfilename = futil.uploadFile(
							ctx.getMerchantid(), "core.branding.cart", f,
							this.getUploadbannerFileName(), this
									.getUploadbannerContentType());

					store.setStorebanner(this.getUploadbannerFileName());

					DynamicImage img = new DynamicImage();
					img.setImageName(this.getUploadbannerFileName());

					// log.debug("Image path " + finalfilename);
					img.setImagePath(finalfilename);
					super.getServletRequest().setAttribute("Banner", img);
				} catch (FileException e) {
					if (e.getType() == FileException.USER) {
						MessageUtil.addErrorMessage(super.getServletRequest(),
								e.getMessage());
					} else {
						log.error(e);
						MessageUtil.addErrorMessage(super.getServletRequest(),
								LabelUtil.getInstance().getText(
										"errors.technical"));
					}
					return SUCCESS;
				}

			}
			
			if(StringUtils.isBlank(merchantStore.getDomainName())) {
				
				
				String serverName = super.getServletRequest().getServerName();
				int serverPort = super.getServletRequest().getServerPort();
				
				if(serverPort>0) {
					serverName = serverName + ":" + String.valueOf(serverPort);
				}
				
				merchantStore.setDomainName(serverName);
				
				
			}
			


			mservice.saveOrUpdateMerchantStore(store);

			super.setSuccessMessage();

		} catch (HibernateException e) {
			log.error(e);
			super.setTechnicalMessage();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}
		return SUCCESS;
	}

	public File getUploadbanner() {
		return uploadbanner;
	}

	public void setUploadbanner(File uploadbanner) {
		this.uploadbanner = uploadbanner;
	}

	public String getUploadbannerContentType() {
		return uploadbannercontenttype;
	}

	public void setUploadbannerContentType(String uploadbannercontenttype) {
		this.uploadbannercontenttype = uploadbannercontenttype;
	}

	public String getUploadbannerFileName() {
		return uploadbannerfilename;
	}

	public void setUploadbannerFileName(String uploadbannerfilename) {
		this.uploadbannerfilename = uploadbannerfilename;
	}

	public String getImageId() {
		return imageId;
	}

	public void setImageId(String imageId) {
		this.imageId = imageId;
	}

	public File getUploadlogo() {
		return uploadlogo;
	}

	public void setUploadlogo(File uploadlogo) {
		this.uploadlogo = uploadlogo;
	}

	public MerchantStore getMerchantStore() {
		return merchantStore;
	}

	public void setMerchantStore(MerchantStore merchantStore) {
		this.merchantStore = merchantStore;
	}

	public MerchantConfiguration getSiteMapConfiguration() {
		return siteMapConfiguration;
	}

	public void setSiteMapConfiguration(MerchantConfiguration siteMapConfiguration) {
		this.siteMapConfiguration = siteMapConfiguration;
	}

}



```
