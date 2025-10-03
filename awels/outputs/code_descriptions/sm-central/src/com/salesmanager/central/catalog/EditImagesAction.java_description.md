# EditImagesAction.java

## Review

## 1. Summary

**Purpose**  
`EditImagesAction` is a Struts‑style action class that allows merchants to upload, display and delete product images. It interacts with the core catalog services to fetch a product, validates uploaded files, persists images on disk, and updates the product entity with the new image names.

**Key components**  

| Component | Role |
|-----------|------|
| `Product` | Entity representing a catalog product |
| `CatalogService` | CRUD operations on products |
| `ReferenceService` & `MerchantService` | Retrieve configuration & merchant store information |
| `ProductImageUtil` | Handles the actual image file manipulation (resizing, naming, etc.) |
| `FileModule` | Deletes image files from the file system |
| `DynamicImage` | Wraps image metadata for rendering in the view |

**Design patterns / libraries**

* **Service Locator / Factory** – `ServiceFactory.getService(...)`
* **Dependency Injection** – `SpringUtil.getBean("localfile")` retrieves a Spring bean
* **Struts‑2 Action** – extends `BaseAction`, uses request attributes and `getText()` for i18n
* **Commons Configuration** – loads application properties
* **Apache Commons IO/Lang** – file handling, string utilities

---

## 2. Detailed Description

### Initialization

* Static block reads configuration values (`maxfilesize`, `maximagesize`, `image content‑types`) from `properties`.
* The values are stored in static fields; if missing or malformed, defaults are used and an error is logged.

### Execution flow

1. **`displayImages()`**  
   * Sets page title.  
   * Retrieves the product by ID and verifies that it belongs to the current merchant.  
   * For each of the four possible image slots, if an image exists it creates a `DynamicImage` with the correct prefix and path and places it in the request attributes (`DYNIMG1` … `DYNIMG4`).  
   * Sets fixed image dimensions (`200x200`) and the product ID in request scope.  
   * Returns `SUCCESS` for rendering the JSP.

2. **`saveImages()`**  
   * Again verifies product ownership.  
   * Validates that uploaded files exist and match allowed content‑types and size limits.  
   * For each file, calls `ProductImageUtil.uploadProductImages(...)` which is responsible for the actual file copy/resize operations.  
   * After all uploads, iterates over `uploadFileName` to assign the new file names to the first empty image slots (`productImage1…4`).  
   * Persists the updated product and displays a success message.

3. **`deleteImage()`**  
   * Loads the product, checks authorization, and parses the `imageId` (1‑4).  
   * Clears the corresponding `productImageX` field.  
   * Deletes the three image variations (original, small, large) from disk via `FileModule`.  
   * Persists the product and shows a success message.

### Assumptions & Constraints

* The action assumes a maximum of four images per product.  
* The image naming convention is `{productId}-{originalFileName}`.  
* The action trusts that `uploadFileName`, `uploadContentType` arrays align with `upload` array indices.  
* All image files are stored under `/product/{merchantId}/` relative to the base product file path.  
* It relies on Struts request attributes for view rendering – no JSON API.

---

## 3. Functions / Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `displayImages()` | Prepares request attributes for the image view. | None | `String` (view name) | Sets page title, request attributes, validates merchant |
| `saveImages()` | Validates, uploads, and assigns images to a product. | None | `String` | Modifies product entity, writes files, updates DB, sets flash message |
| `deleteImage()` | Removes one of the four product images. | None | `String` | Deletes files, clears DB field, updates DB, sets flash message |
| `getProduct()/setProduct(Product)` | Bean property accessors. | `Product` | `Product` | None |
| `getUpload()/setUpload(File[])` | Accessors for uploaded files. | `File[]` | `File[]` | None |
| `getUploadFileName()/setUploadFileName(String[])` | Accessors for file names. | `String[]` | `String[]` | None |
| `getUploadContentType()/setUploadContentType(String[])` | Accessors for MIME types. | `String[]` | `String[]` | None |
| `getImageId()/setImageId(String)` | Accessor for the image slot identifier. | `String` | `String` | None |

---

## 4. Dependencies

| External | Type | Notes |
|----------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Handles property files |
| `org.apache.commons.lang.StringUtils` | Third‑party | String utilities |
| `org.apache.log4j.Logger` | Third‑party | Logging |
| `com.salesmanager.*` | Internal | Core framework services, entities, utilities |
| `SpringUtil` | Internal | Spring bean resolution |
| `FileModule` | Internal | File system abstraction (local/remote) |

All dependencies are standard for a Java EE web application using Struts and Spring.

---

## 5. Additional Notes & Recommendations

### 5.1 Code Duplication

* `displayImages()` repeats the same block for each image slot; consider extracting a helper that builds a `DynamicImage` and sets the attributes.
* `saveImages()` contains four almost identical blocks assigning filenames to image slots. This can be replaced with a loop that scans the first empty slot.

### 5.2 Error Handling

* When `product` is null after `cservice.getProduct()`, `super.setAuthorizationMessage()` is called but `displayImages()` is recursively invoked, leading to an infinite recursion. It should return `INPUT` or an error view instead.
* `deleteImage()` recursively calls `displayImages()` before any validation; this might not be necessary and could cause unwanted side effects.
* `maximagesize` is set once in the static block; if the configuration changes at runtime it won’t be reflected.

### 5.3 Concurrency & Thread Safety

* Static configuration maps (`imgctypes`) and size fields are immutable after init, so they are thread‑safe.
* However, the action instance fields (`upload`, `uploadFileName`, etc.) are per‑request, so no cross‑request contamination.

### 5.4 Validation Logic

* The content‑type check uses `imgctypes.containsKey(c)`. If the MIME type list is incomplete (e.g., contains `image/jpeg` but file uploads come as `image/pjpeg`), uploads may be rejected incorrectly.
* The file size check uses `u.length()` which is the size of the temporary file on disk; if the upload is performed with chunking or streaming, this might not be accurate.

### 5.5 Naming Conventions

* Method names follow JavaBean conventions, but class name `EditImagesAction` could be more specific (`ProductImageAction`).
* Use of magic numbers (`1-4`) to index image slots could be replaced with an enum or constant array to improve readability.

### 5.6 Performance

* `ProductImageUtil.uploadProductImages()` is called for each file but the method signature suggests it already handles all variants; ensure it does not re‑scan the entire product each time.
* File deletion in `deleteImage()` manually constructs three paths. Consider centralizing deletion logic in a helper method.

### 5.7 Future Enhancements

| Feature | Benefit |
|---------|---------|
| **Dynamic image slots** | Support more than four images without code changes |
| **RESTful API** | Enable integration with front‑end frameworks or mobile apps |
| **Async upload** | Improve UX by allowing uploads in the background |
| **Validation messages** | Centralize error messages in a constants file or properties |
| **Unit tests** | Mock services and test validation logic |

---

### Bottom Line

The `EditImagesAction` class implements the required CRUD functionality for product images with a clear separation between business logic (`CatalogService`) and presentation logic (`DynamicImage` and request attributes). However, the code suffers from significant duplication, subtle recursion bugs, and a lack of abstraction for repeated patterns. Refactoring to extract helper methods, improving validation, and adding unit tests would greatly increase maintainability and robustness.

## Code Critique



## Code Preview

```java
/*
 * Provided by CSTI Consulting 
 * Following GNU LESSER GENERAL PUBLIC LICENSE
 * You may obtain more details at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-2011 Consultation CS-TI inc. 
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
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.StringTokenizer;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.central.web.DynamicImage;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.module.model.application.FileModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.ProductImageUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.SpringUtil;

/**
 * Upload and manage product images
 * @author Carl Samson
 *
 */
public class EditImagesAction extends BaseAction {
	
	private File[] upload;
    private String[] uploadFileName;
    private String[] uploadContentType;

    
    private static Logger log = Logger.getLogger(EditImagesAction.class);
    
	// image validation
	private static long maximagesize;
	private static long maxfilesize;
	private static Map imgctypes = new HashMap();
	private static Configuration conf = PropertiesUtil.getConfiguration();
	
    private Product product;
    
    //id of image to be deleted
    private String imageId = null;
    
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
    
    

    public String displayImages() throws Exception {
    	
    	super.setPageTitle("label.product.images");
    	
    	//get the product
		CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);
		
		Product p = cservice.getProduct(this.getProduct().getProductId());
		if(p==null || p.getMerchantId()!=super.getContext().getMerchantid()) {
			super.setAuthorizationMessage();
			displayImages();
			return INPUT;
		}
		
		p.setLocale(super.getLocale());
    	
		this.setProduct(p);
		
    	//do this for 4 images
    	if (!StringUtils.isBlank(product.getProductImage1())) {
    		
    		
			DynamicImage img = new DynamicImage();
			img
					.setImageName(new StringBuffer()
							.append(conf
									.getString("core.product.image.large.prefix"))
									.append("-").append(
									this.getProduct().getProductImage1())
							.toString());

			img.setImagePath(FileUtil.getProductFilePath()
					+ "/" + super.getContext().getMerchantid() + "/");
			
			
			super.getServletRequest().setAttribute("DYNIMG1", img);
			super.getServletRequest().setAttribute("uploadimagename1",
					product.getProductImage1());
			
    	}
    	
    	if (!StringUtils.isBlank(product.getProductImage2())) {
    		
    		
			DynamicImage img = new DynamicImage();
			img
					.setImageName(new StringBuffer()
							.append(conf
									.getString("core.product.image.large.prefix"))
									.append("-").append(
									this.getProduct().getProductImage2())
							.toString());

			img.setImagePath(FileUtil.getProductFilePath()
					+ "/" + super.getContext().getMerchantid() + "/");
			
			
			super.getServletRequest().setAttribute("DYNIMG2", img);
			super.getServletRequest().setAttribute("uploadimagename2",
					product.getProductImage2());
			
    	}
    	
    	if (!StringUtils.isBlank(product.getProductImage3())) {
    		
    		
			DynamicImage img = new DynamicImage();
			img
					.setImageName(new StringBuffer()
							.append(conf
									.getString("core.product.image.large.prefix"))
									.append("-").append(
									this.getProduct().getProductImage3())
							.toString());

			img.setImagePath(FileUtil.getProductFilePath()
					+ "/" + super.getContext().getMerchantid() + "/");
			
			
			super.getServletRequest().setAttribute("DYNIMG3", img);
			super.getServletRequest().setAttribute("uploadimagename3",
					product.getProductImage3());
			
    	}
    	
    	if (!StringUtils.isBlank(product.getProductImage4())) {
    		
    		
			DynamicImage img = new DynamicImage();
			img
					.setImageName(new StringBuffer()
							.append(conf
									.getString("core.product.image.large.prefix"))
									.append("-").append(
									this.getProduct().getProductImage4())
							.toString());

			img.setImagePath(FileUtil.getProductFilePath()
					+ "/" + super.getContext().getMerchantid() + "/");
			
			
			super.getServletRequest().setAttribute("DYNIMG4", img);
			super.getServletRequest().setAttribute("uploadimagename4",
					product.getProductImage4());
			
    	}
    	
    	super.getServletRequest().setAttribute("imagewidth","200");
    	super.getServletRequest().setAttribute("imageheight","200");
    	super.getServletRequest().setAttribute("product.productId",String.valueOf(p.getProductId()));
    	
    	return SUCCESS;
    	
    }
    
	
	public String saveImages() throws Exception {
		
		super.setPageTitle("label.product.images");
		
		displayImages();
		
		//get product first
		CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);
		
		Product p = cservice.getProduct(this.getProduct().getProductId());
		if(p==null || p.getMerchantId()!=super.getContext().getMerchantid()) {
			super.setAuthorizationMessage();
			return INPUT;
		}
		
		this.getProduct().setMerchantId(p.getMerchantId());
		
		p.setLocale(super.getLocale());
		
		
		
		
		if(upload==null || upload.length==0) {
			return INPUT;
		}
		
		boolean hasError = false;
		

		
		//validate images
		if (this.getUploadContentType()!=null && this.getUploadContentType().length>0) {
			
			for (String c: uploadContentType) {
				if (!imgctypes.containsKey(c)) {
					super
							.addActionError(
									getText("error.message.product.image.invalidfiletype")
											+ " "
											+ getText("label.product.uploadimage"));
					hasError = true;
				}
	        }
		}

		if (this.getUpload()!=null && this.getUpload().length>0) {
			
			
			for (File u: upload) {

				if (u.length() > this.maximagesize) {

					super.addActionError(
							getText("error.message.product.image.file") + " "
									+ getText("label.product.uploadimage"));
					hasError = true;
				}
	        }
		}
		
		if(hasError) {
			return INPUT;
		}
		
		//upload pictures
		

		
		
		ReferenceService rservice = (ReferenceService) ServiceFactory
		.getService(ServiceFactory.ReferenceService);
		MerchantService service = (MerchantService) ServiceFactory
		.getService(ServiceFactory.MerchantService);
		MerchantStore mStore = service.getMerchantStore(super.getContext()
		.getMerchantid());
		Map<String, String> moduleConfigMap = rservice
		.getModuleConfigurationsKeyValue(mStore
				.getTemplateModule(), mStore.getCountry());


		for (int i = 0; i< this.getUpload().length;i++) {
			
			File f = this
			.getUpload()[i];
			
			ProductImageUtil imageSpecifications = new ProductImageUtil();
			imageSpecifications.uploadProductImages(
			f, this.getUploadFileName()[i],
			this.getUploadContentType()[i],
			this.getProduct(), moduleConfigMap);
			
			
		}
		
		for(int i=0;i<uploadFileName.length;i++) {
			
			if(i==0) {
				if(StringUtils.isBlank(p.getProductImage1())) {
					p.setProductImage1(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage2())) {
					p.setProductImage2(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage3())) {
					p.setProductImage3(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage4())) {
					p.setProductImage4(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
			}
			
			if(i==1) {
				if(StringUtils.isBlank(p.getProductImage1())) {
					p.setProductImage1(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage2())) {
					p.setProductImage2(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage3())) {
					p.setProductImage3(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage4())) {
					p.setProductImage4(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
			}
			
			if(i==2) {
				if(StringUtils.isBlank(p.getProductImage1())) {
					p.setProductImage1(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage2())) {
					p.setProductImage2(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage3())) {
					p.setProductImage3(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage4())) {
					p.setProductImage4(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
			}
			
			if(i==3) {
				if(StringUtils.isBlank(p.getProductImage1())) {
					p.setProductImage1(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage2())) {
					p.setProductImage2(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage3())) {
					p.setProductImage3(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
				
				if(StringUtils.isBlank(p.getProductImage4())) {
					p.setProductImage4(new StringBuffer().append(p.getProductId()).append("-").append(
							uploadFileName[i]).toString());
					continue;
				}
			}
			
			
		}
		
		
		cservice.saveOrUpdateProduct(p);
		
		super.setSuccessMessage();
		
		
		
		return SUCCESS;
		
		
	}
	
	public String deleteImage() throws Exception {
		


			// get the product firts
			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			this.displayImages();

			CatalogService cservice = (CatalogService)ServiceFactory.getService(ServiceFactory.CatalogService);
			
			Product p = cservice.getProduct(this.getProduct().getProductId());
			if(p==null || p.getMerchantId()!=super.getContext().getMerchantid()) {
				super.setAuthorizationMessage();
				return INPUT;
			}
			
			if(StringUtils.isBlank(this.getImageId())) {
				log.error("Image id is null");
				super.setTechnicalMessage();
				return INPUT;
			}

			String folder = null;
			String imagename = null;
			
			int iId = 0;
			
			try {
				iId = Integer.parseInt(this.getImageId());
			} catch (Exception e) {
				log.error("Image id is not numeric " + this.getImageId());
				super.setTechnicalMessage();
				return INPUT;
			}
			
			String imageName = null;
			
			switch (iId) {
			
				case 1:
					imageName = p.getProductImage1();
					p.setProductImage1(null);
					
					break;
			
				case 2:
					imageName = p.getProductImage2();
					p.setProductImage2(null);
					
					break;
			
				case 3:
					imageName = p.getProductImage3();
					p.setProductImage3(null);
					
					break;
			
				case 4:
					imageName = p.getProductImage4();
					p.setProductImage4(null);
					
					break;
					
				
			
			
			}

			if(imageName==null) {
				log.error("Invalid image id " + this.getImageId());
				super.setTechnicalMessage();
				return INPUT;
			}

			ProductImageUtil imutil = new ProductImageUtil();



			folder = FileUtil.getProductFilePath()
					+ "/"
					+ ctx.getMerchantid() + "/";


			cservice.saveOrUpdateProduct(p);

				FileModule fh = (FileModule) SpringUtil.getBean("localfile");
				fh.deleteFile(ctx.getMerchantid(), new File(new StringBuffer()
						.append(folder).append(imageName).toString()));// delete
																		// regular
				fh
						.deleteFile(
								ctx.getMerchantid(),
								new File(
										new StringBuffer()
												.append(folder)
												.append(
														conf
																.getString("core.product.image.small.prefix"))
												.append("-").append(imageName)
												.toString()));
				fh
						.deleteFile(
								ctx.getMerchantid(),
								new File(
										new StringBuffer()
												.append(folder)
												.append(
														conf
																.getString("core.product.image.large.prefix"))
												.append("-").append(imageName)
												.toString()));



			super.setSuccessMessage();

			return SUCCESS;


		
		
	}
	
	

	
	public Product getProduct() {
		return product;
	}
	public void setProduct(Product product) {
		this.product = product;
	}


	public File[] getUpload() {
		return upload;
	}


	public void setUpload(File[] upload) {
		this.upload = upload;
	}


	public String[] getUploadFileName() {
		return uploadFileName;
	}


	public void setUploadFileName(String[] uploadFileName) {
		this.uploadFileName = uploadFileName;
	}


	public String[] getUploadContentType() {
		return uploadContentType;
	}


	public void setUploadContentType(String[] uploadContentType) {
		this.uploadContentType = uploadContentType;
	}


	public String getImageId() {
		return imageId;
	}


	public void setImageId(String imageId) {
		this.imageId = imageId;
	}


}



```
