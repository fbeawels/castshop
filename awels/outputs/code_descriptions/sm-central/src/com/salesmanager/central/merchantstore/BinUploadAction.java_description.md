# BinUploadAction.java

## Review

## 1. Summary
**Purpose**  
`BinUploadAction` is a Struts‑style action that allows merchants to upload and delete media files (images, flash, generic files) to a “bin” directory that is organized per‑merchant. The class also provides simple navigation helpers for displaying the bin tree and file browser UI.

**Key Components**

| Component | Role |
|-----------|------|
| `upload`, `uploadFileName`, `uploadContentType` | Holds the multipart upload information supplied by the front‑end. |
| `deleteFilePath` | Path of a file to delete (supplied by the UI). |
| `type` | UI helper for displaying the type of bin being browsed. |
| `FileModule futil` | Spring‑managed service that performs the actual file copy/delete. |
| `FileUtil.ContentCategoryType` | Enum used to map MIME types to logical categories. |
| `Context` | Holds the current merchant ID pulled from the session. |

**Notable Patterns / Libraries**

* **Spring** – used to look up the `localfile` bean (`FileModule`).
* **Log4j** – for logging.
* **Java EE / Struts** – extends `BaseAction`, uses `getServletRequest()`, `setPageTitle`, `setSuccessMessage`, etc.

---

## 2. Detailed Description
The action follows a typical Struts flow:

1. **Navigation Methods**
   * `binTreeDisplay()` and `displayFileBrowser()` only set the page title and return `SUCCESS`.  
   * These are called from the UI to show the appropriate page.

2. **Upload Flow (`uploadToBin`)**
   * Sets the page title and writes the `type` attribute to the request.
   * Validates that an upload was supplied via `isBlank()`. If not, an error message is set and the `INPUT` view is returned.
   * Retrieves the current `merchantid` from the session context.
   * Looks up the `FileModule` bean and determines the content category of the uploaded file by mapping its MIME type.
   * Depending on the category, the file is copied into one of three merchant‑specific subdirectories (`core.bin.images`, `core.bin.flash`, `core.bin.files`).
   * Errors during copy cause a technical error message and `INPUT` is returned.
   * On success, a generic success message is set and the `SUCCESS` view is returned.

3. **Delete Flow (`deleteFile`)**
   * Retrieves the `merchantid` from the session context.
   * Calls `FileModule.deleteFile()` with the merchant id and a new `File(deleteFilePath)`.
   * Returns `SUCCESS` after setting a success message.

4. **Utility Methods**
   * `isBlank()` simply checks for a missing file or filename.
   * Standard getters/setters for all properties.

**Assumptions & Constraints**

* The action assumes the caller has already performed authentication and that the `merchantid` is available in the session.
* File type validation is limited to MIME type mapping; no checks on file extensions or actual file content.
* No size or quota enforcement.
* The UI must supply a valid `deleteFilePath`; the method blindly deletes whatever path is provided.

**Architecture & Design Choices**

* **Single Responsibility** – The action is narrowly focused on request handling; heavy lifting is delegated to `FileModule`.
* **Dependency Injection** – Uses Spring to inject the file service.
* **No Service Layer** – All business logic resides in the action, which can be hard to unit‑test and violates separation of concerns.

---

## 3. Functions/Methods

| Method | Purpose | Parameters / Inputs | Returns | Side‑Effects |
|--------|---------|---------------------|---------|--------------|
| `binTreeDisplay()` | Show bin tree UI | None | `String` (`SUCCESS`) | Sets page title |
| `displayFileBrowser()` | Show file browser UI | None | `String` (`SUCCESS`) | Sets page title |
| `uploadToBin()` | Handle file upload | None (reads fields) | `String` (`SUCCESS` / `INPUT`) | Sets messages, writes file via `FileModule` |
| `deleteFile()` | Delete a file | None (uses `deleteFilePath`) | `String` (`SUCCESS`) | Deletes file via `FileModule` |
| `isBlank()` | Check that upload and filename are present | None | `boolean` | None |
| `getUpload() / setUpload(File)` | Getter/Setter for uploaded file | `File` | `File` | None |
| `getUploadContentType() / setUploadContentType(String)` | MIME type | `String` | `String` | None |
| `getUploadFileName() / setUploadFileName(String)` | Original file name | `String` | `String` | None |
| `getFileUploadType() / setFileUploadType(String)` | UI helper, not used | `String` | `String` | None |
| `getDeleteFilePath() / setDeleteFilePath(String)` | Path to delete | `String` | `String` | None |
| `getType() / setType(String)` | UI helper | `String` | `String` | None |

**Reusable/Utility Methods**

* `isBlank()` is a simple validation helper but could be expanded into a more generic validator.

---

## 4. Dependencies

| External | Type | Remarks |
|----------|------|---------|
| `com.salesmanager.central.BaseAction` | Core library | Provides Struts helpers, request/response handling. |
| `com.salesmanager.central.profile.Context` | Core library | Holds session‑specific data like merchant ID. |
| `com.salesmanager.central.profile.ProfileConstants` | Core library | Key for session attribute lookup. |
| `com.salesmanager.core.module.impl.application.files.FileException` | Core library | Custom exception for file operations. |
| `com.salesmanager.core.module.model.application.FileModule` | Core library | Service that copies/deletes files. |
| `com.salesmanager.core.util.FileUtil` | Core library | Determines content type enum. |
| `com.salesmanager.core.util.SpringUtil` | Core library | Provides Spring bean lookup. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `java.io.File` | JDK | Standard file representation. |

No other third‑party libraries are used. The code is platform‑agnostic beyond requiring a servlet container and a Spring context.

---

## 5. Additional Notes & Recommendations

### 5.1 Security & Validation
* **Path Traversal** – `deleteFilePath` is used verbatim to instantiate a `File`. An attacker could supply `"../../etc/passwd"` to delete arbitrary files. Always sanitize paths or restrict deletions to a known directory tree.
* **Filename Sanitization** – `uploadFileName` is stored directly; if the file system treats it literally, malicious names (e.g., `../evil.jsp`) could overwrite files outside the intended directory.
* **MIME / Extension Validation** – Relying solely on MIME type is weak; a user can spoof the type. Consider checking file extensions and using libraries like Apache Tika to validate content.
* **Size / Quota Checks** – No limits on file size or merchant storage quotas. This could lead to disk exhaustion.

### 5.2 Robustness
* **Null Checks** – `deleteFilePath` is not validated for null. Add a guard clause to avoid `NullPointerException`.
* **Exception Handling** – The upload methods catch generic `Exception` for FLASH and FILE types. Catching `FileException` consistently (as with IMAGE) would be cleaner and more explicit.
* **Duplicate File Names** – The current `copyFile` implementation likely overwrites existing files. Decide on a policy (e.g., append timestamp or prompt the user).

### 5.3 Design Improvements
* **Separate Service Layer** – Move file handling logic to a dedicated service (e.g., `BinService`) that the action calls. This would make unit testing easier and keep the action thin.
* **Use Enums for Bin Types** – Instead of hard‑coding string paths (`core.bin.images` etc.), define an enum mapping each category to its directory.
* **Internationalization** – Use constants or resource keys for all messages (`error.upload.required`, `error.bin.upload.invalid.type`).

### 5.4 Future Enhancements
1. **Bulk Upload / Delete** – Support multiple file uploads in one request.
2. **Metadata Management** – Store additional information (e.g., MIME type, size, upload timestamp) in a database for better tracking.
3. **Access Control** – Verify that the requesting merchant has rights to delete files that belong to them.
4. **Unit & Integration Tests** – Mock `FileModule` to verify upload/delete logic without touching the filesystem.

--- 

**Overall**, the class performs its core task of delegating file operations to a service bean and handling basic UI navigation. However, it lacks critical security checks, robust error handling, and clean separation of concerns. Addressing the points above would greatly improve reliability, maintainability, and safety.

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

import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.module.impl.application.files.FileException;
import com.salesmanager.core.module.model.application.FileModule;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.FileUtil.ContentCategoryType;

public class BinUploadAction extends BaseAction {

	private static final long serialVersionUID = 1L;
	private Logger log = Logger.getLogger(BinUploadAction.class);
	private File upload;
	private String uploadContentType;
	private String uploadFileName;
	private String fileUploadType;
	private String deleteFilePath;
	private String type;

	public String binTreeDisplay() {
		super.setPageTitle("label.media.binmanagement");
		return SUCCESS;
	}

	public String displayFileBrowser() {
		super.setPageTitle("label.media.binmanagement");
		return SUCCESS;

	}

	public String uploadToBin() {
		
		super.setPageTitle("label.media.binmanagement");

		super.getServletRequest().setAttribute("Type", this.getType());
		if (isBlank()) {
			setErrorMessage(getText("error.upload.required"));
			return INPUT;
		}
		Context ctx = super.getContext();
		Integer merchantid = ctx.getMerchantid();
		FileModule futil = (FileModule) SpringUtil.getBean("localfile");
		ContentCategoryType contentType = FileUtil
				.getContentCategoryType(uploadContentType);
		if (ContentCategoryType.IMAGE.equals(contentType)) {

			try {
				futil.copyFile(merchantid, "core.bin.images", getUpload(),
						getUploadFileName(), getUploadContentType());
			} catch (FileException e) {
				log.error(e);
				super.setTechnicalMessage();
				return INPUT;
			}
		} else if (ContentCategoryType.FLASH.equals(contentType)) {
			try {
				futil.copyFile(merchantid, "core.bin.flash", getUpload(),
						getUploadFileName(), getUploadContentType());
			} catch (Exception e) {
				log.error(e);
				super.setTechnicalMessage();
				return INPUT;
			}
		} else if (ContentCategoryType.FILE.equals(contentType)) {
			try {
				futil.copyFile(merchantid, "core.bin.files", getUpload(),
						getUploadFileName(), getUploadContentType());
			} catch (Exception e) {
				log.error(e);
				super.setTechnicalMessage();
				return INPUT;
			}
		} else {
			setErrorMessage(getText("error.bin.upload.invalid.type"));
			return INPUT;
		}
		super.setSuccessMessage();
		return SUCCESS;
	}

	public String deleteFile() {
		
		super.setPageTitle("label.media.binmanagement");
		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();
		FileModule futil = (FileModule) SpringUtil.getBean("localfile");
		futil.deleteFile(merchantid, new File(deleteFilePath));
		super.setSuccessMessage();
		return SUCCESS;
	}

	public boolean isBlank() {
		return (upload == null || uploadFileName == null);
	}

	public File getUpload() {
		return upload;
	}

	public void setUpload(File upload) {
		this.upload = upload;
	}

	public String getUploadContentType() {
		return uploadContentType;
	}

	public void setUploadContentType(String uploadContentType) {
		this.uploadContentType = uploadContentType;
	}

	public String getUploadFileName() {
		return uploadFileName;
	}

	public void setUploadFileName(String uploadFileName) {
		this.uploadFileName = uploadFileName;
	}

	public String getFileUploadType() {
		return fileUploadType;
	}

	public void setFileUploadType(String fileUploadType) {
		this.fileUploadType = fileUploadType;
	}

	public String getDeleteFilePath() {
		return deleteFilePath;
	}

	public void setDeleteFilePath(String deleteFilePath) {
		this.deleteFilePath = deleteFilePath;
	}

	public String getType() {
		return type;
	}

	public void setType(String type) {
		this.type = type;
	}

}



```
