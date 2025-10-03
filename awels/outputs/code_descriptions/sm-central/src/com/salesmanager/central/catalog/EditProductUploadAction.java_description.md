# EditProductUploadAction.java

## Review

## 1. Summary  
**Purpose** – `EditProductUploadAction` is a Struts 2 action that handles the upload of product‑specific files (e.g., downloadable assets). It provides two primary entry points:  
- `showUploadForm()` – prepares the form by loading the current product and any existing download information.  
- `uploadProduct()` – validates the submitted file, checks its size against a configurable maximum, and persists the upload via `CatalogService`.  

**Key components**  
| Component | Role |
|-----------|------|
| `BaseAction` | Provides common Struts utilities (request, locale, messages, page title). |
| `ServletContextAware` | Allows injection of the servlet context (currently unused). |
| `CatalogService` | Business service that fetches and persists `Product` and its download metadata. |
| `LabelUtil` / `MessageUtil` | Internationalisation and user‑visible messaging. |
| `PropertiesUtil` / `Configuration` | Reads configuration values (e.g., `core.product.file.maxfilesize`). |
| `Logger` | Logging (Log4j). |

The code follows a classic Struts 2 action‑pattern, using request attributes to pass data back to the view and a simple static‑initialiser for the max‑file‑size value.

---

## 2. Detailed Description  
### Flow of execution  
1. **Initialization** – The class is instantiated by the Struts 2 framework per HTTP request.  
2. **`showUploadForm()`**  
   * Sets a generic upload page title.  
   * Retrieves the `Context` from the HTTP session to obtain user‑specific information.  
   * Checks that a `Product` object with a valid ID is available.  
   * Loads the full `Product` from `CatalogService`, attaches it to the action, and places its ID on the request.  
   * Attempts to fetch an existing `ProductAttributeDownload` and, if present, exposes the original filename.  
   * Returns `SUCCESS` (i.e., forwards to the upload JSP); otherwise returns `"unauthorized"`.  
3. **`uploadProduct()`**  
   * Again sets the upload page title.  
   * Validates that a file and filename are present; if not, falls back to `showUploadForm()` (returns `SUCCESS`).  
   * Checks the file size against the configured maximum; if exceeded, adds a field error and returns `SUCCESS`.  
   * Retrieves the target product, re‑loads it, and invokes `persistUploadProduct()` to store the file and its metadata.  
   * On success, registers a confirmation message; on any exception, logs the error and registers a generic technical error.  
   * Returns `SUCCESS` in all code paths (the view is responsible for interpreting the messages).  

### Assumptions & Constraints  
* The action expects a pre‑populated `Product` bean (usually set by a preceding action or URL parameter).  
* `MAXFILESIZE` is read once at class load time; subsequent changes to the property file will not affect running instances.  
* File upload handling is performed by Struts 2’s built‑in file upload support; the code does **not** perform MIME‑type validation or virus scanning.  
* No concurrent safety concerns are addressed (multiple uploads for the same product could race).  

### Design choices  
* **Static configuration**: Simpler to implement but less flexible for hot‑reload.  
* **Error handling**: Uses `MessageUtil` for user‑facing messages; logs all exceptions.  
* **Action scope**: Keeps request‑level data in the `HttpServletRequest` rather than in the action, which is typical for Struts 2 but somewhat redundant given the action’s own fields.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `showUploadForm()` | Prepares the upload form, loads product and any existing download. | None (uses `product` field). | `String` action result (`SUCCESS` or `"unauthorized"`). | Sets request attributes (`product.productId`, `uploadfilename`), logs errors. |
| `uploadProduct()` | Handles the actual file upload, validation, persistence. | None (uses `product`, `uploadfile`, `uploadfileFileName`, `uploadfileContentType`). | `String` action result (`SUCCESS`). | Adds field errors, message alerts, persists file via `CatalogService`, logs. |
| `getUploadfile()`, `setUploadfile(File)` | File getter/setter. | `File` | `File` | None |
| `getUploadfileContentType()`, `setUploadfileContentType(String)` | Content‑type getter/setter. | `String` | `String` | None |
| `getUploadfileFileName()`, `setUploadfileFileName(String)` | Filename getter/setter. | `String` | `String` | None |
| `getProduct()`, `setProduct(Product)` | Product getter/setter. | `Product` | `Product` | None |
| `getServletContext()`, `setServletContext(ServletContext)` | Context getter/setter (required by `ServletContextAware`). | `ServletContext` | `ServletContext` | None |
| **Static block** | Reads `core.product.file.maxfilesize` from the configuration. | None | None | Sets `MAXFILESIZE`; logs any errors. |

**Utility / Re‑usable**  
`MAXFILESIZE` and the static config block could be extracted to a helper or injected via Spring for easier testing and dynamic updates.

---

## 4. Dependencies  

| Library / Package | Usage | Standard / Third‑Party |
|-------------------|-------|------------------------|
| `org.apache.commons.configuration.Configuration` | Reads application properties | Third‑party |
| `org.apache.commons.lang.StringUtils` | String safety checks | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `org.apache.struts2.util.ServletContextAware` | Injection of `ServletContext` | Third‑party (Struts 2) |
| `com.salesmanager.central.*` | Base action, profile context, constants | In‑house |
| `com.salesmanager.core.*` | Entity models (`Product`, `ProductAttributeDownload`), services (`CatalogService`), utilities (`LabelUtil`, `MessageUtil`, `PropertiesUtil`) | In‑house |

No native JDK classes beyond `java.io.File` and `javax.servlet.ServletContext` are used.

---

## 5. Additional Notes  

### Strengths  
* Clear separation of concerns: business logic stays in `CatalogService`.  
* Uses Struts 2 conventions for file upload and request attribute handling.  
* Internationalised messaging via `LabelUtil`.  

### Weaknesses & Edge Cases  
1. **Inconsistent error paths** – `uploadProduct()` falls back to `showUploadForm()` when no file is submitted but still returns `SUCCESS`. The view may treat this as a successful upload.  
2. **Null‑pointer risk** – The action assumes `product` is non‑null and has a valid ID. If the session is stale or the user manipulates the form, a `NullPointerException` could be thrown before reaching the catch blocks.  
3. **File validation** – No checks on file type, size overflow beyond `long`, or duplicate filenames. Potential overwrites or storage issues.  
4. **Static configuration** – `MAXFILESIZE` is loaded only once; if the config file is updated at runtime, the action will not reflect the change.  
5. **Unused `ServletContext`** – The injected context is never used; consider removing the interface implementation or leveraging it for path resolution.  
6. **Logging level** – Errors are logged at `error` level but sometimes a `warn` might be more appropriate (e.g., user‑visible validation errors).  
7. **Thread‑safety** – The action is request‑scoped, but the static `MAXFILESIZE` is shared. This is safe for reads but could be problematic if the value is updated dynamically.  

### Suggested Enhancements  
* **Validate the product and user permissions** before proceeding; return a clear HTTP status or error view if unauthorized.  
* **Move configuration** to a Spring bean or a dynamic `@ConfigurationProperties` class so the value can be refreshed without redeploy.  
* **Improve file handling** – generate unique filenames, validate MIME types, and limit total upload size.  
* **Add unit tests** for the action methods, mocking `CatalogService` and `MessageUtil`.  
* **Remove unused `ServletContext`** or use it for resolving the upload directory.  
* **Handle file upload exceptions** separately (e.g., `FileUploadException`) to provide more specific user feedback.  

Overall, the action fulfills its basic purpose but would benefit from tighter error handling, more robust file validation, and a more dynamic configuration strategy.

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

import javax.servlet.ServletContext;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.apache.struts2.util.ServletContextAware;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttributeDownload;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.PropertiesUtil;

public class EditProductUploadAction extends BaseAction implements
		ServletContextAware {

	private String uploadfilefilename;
	private File uploadfile;
	private String uploadfilecontenttype;

	private ServletContext servletContext;

	private Product product;

	private static Logger log = Logger.getLogger(EditProductUploadAction.class);

	private static Long MAXFILESIZE = null;

	private static Configuration conf = PropertiesUtil.getConfiguration();

	static {
		try {
			Long newmaxfilesize = conf.getLong("core.product.file.maxfilesize");
			if (newmaxfilesize != null) {
				MAXFILESIZE = newmaxfilesize;
			}
		} catch (Exception e) {
			log.error(e);
		}
	}

	public String showUploadForm() {
		
		super.setPageTitle("label.generic.uploadfile");

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);

		if (this.getProduct() != null && this.getProduct().getProductId() > 0) {

			long lproductid = this.getProduct().getProductId();

			Product product;
			try {
				CatalogService catalogservice = (CatalogService) ServiceFactory
						.getService(ServiceFactory.CatalogService);
				product = catalogservice.getProduct(lproductid);

				this.setProduct(product);
				super.getServletRequest().setAttribute("product.productId",
						this.getProduct().getProductId());

				ProductAttributeDownload pda = catalogservice
						.getProductDownload(lproductid);

				if (pda != null) {
					super.getServletRequest().setAttribute("uploadfilename",
							pda.getProductAttributeFilename());
				}

			} catch (Exception e) {
				log.error(e);
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),"errors.technical"));
			}

			return SUCCESS;

		} else {
			return "unauthorized";
		}

	}

	public String uploadProduct() {
		
		super.setPageTitle("label.generic.uploadfile");

		try {

			if (this.getUploadfile() == null
					|| this.getUploadfileFileName() == null) {
				this.showUploadForm();
				return SUCCESS;
			}

			if (this.getUploadfile() != null
					&& !StringUtils.isBlank(this.getUploadfileFileName())
					&& this.MAXFILESIZE != null) {
				java.io.File f = this.getUploadfile();

				if (f.length() > this.MAXFILESIZE) {

					super.addFieldError("uploadfile",
							getText("error.message.product.file.file") + " "
									+ getText("label.product.uploadfile"));
					return SUCCESS;
				}
			}

			CatalogService catalogservice = null;
			if (this.getProduct() != null
					&& this.getProduct().getProductId() > 0) {

				long lproductid = this.getProduct().getProductId();

				Product product;
				try {
					catalogservice = (CatalogService) ServiceFactory
							.getService(ServiceFactory.CatalogService);
					product = catalogservice.getProduct(lproductid);
					this.setProduct(product);

					super.getServletRequest().setAttribute("product.productId",
							this.getProduct().getProductId());

					catalogservice.persistUploadProduct(this.getProduct(), this
							.getUploadfile(), this.getUploadfileFileName(),
							this.getUploadfileContentType());

					super.getServletRequest().setAttribute("uploadfilename",
							this.getUploadfileFileName());

				} catch (Exception e) {
					log.error(e);
					MessageUtil
							.addErrorMessage(super.getServletRequest(),
									LabelUtil.getInstance().getText(super.getLocale(),
											"errors.technical"));
					return SUCCESS;

				}

			} else {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),"errors.technical"));
				return SUCCESS;
			}

			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
		}

		return SUCCESS;

	}

	public File getUploadfile() {
		return uploadfile;
	}

	public void setUploadfile(File uploadfile) {
		this.uploadfile = uploadfile;
	}

	public String getUploadfileContentType() {
		return uploadfilecontenttype;
	}

	public void setUploadfileContentType(String uploadfilecontenttype) {
		this.uploadfilecontenttype = uploadfilecontenttype;
	}

	public String getUploadfileFileName() {
		return uploadfilefilename;
	}

	public void setUploadfileFileName(String uploadfilefilename) {
		this.uploadfilefilename = uploadfilefilename;
	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public ServletContext getServletContext() {
		return servletContext;
	}

	public void setServletContext(ServletContext servletContext) {
		this.servletContext = servletContext;
	}

}



```
