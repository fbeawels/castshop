# GenericFileUploadAction.java

## Review

## 1. Summary
**Purpose**  
`GenericFileUploadAction` is a Struts‑2 action that handles CSV uploads for three domain objects: categories, products, and manufacturers. It validates the incoming file, delegates the actual parsing/processing to an `IFileUploadService`, and reports success or detailed line‑level errors back to the UI.

**Key Components**  
| Component | Role |
|-----------|------|
| `BaseAction` | Provides common Struts functionality (i18n, request/session access, message handling). |
| `IFileUploadService` | Service interface (obtained via `ServiceFactory`) that does the heavy lifting of parsing the CSV and persisting data. |
| `CSVConstants` | Holds constants such as the CSV file suffix and a generic error message. |
| `ProfileConstants.merchant` | Session attribute key used to pass the merchant id to the service. |
| `Logger` | Logs upload events and errors. |

**Design Patterns / Libraries**  
* **Factory** – `ServiceFactory.getService("fileUpload")` creates the file‑upload service.  
* **Strategy** – Different upload methods (`uploadCategories`, `uploadProducts`, `uploadManufacturers`) call the same service API but with different method names.  
* **Dependency Injection (implicit)** – The action obtains its service via a static factory instead of constructing it directly.  
* **Logging** – Uses Apache Log4j.  
* **Struts‑2** – Action extends `BaseAction` and returns Struts constants (`INPUT`, `ERROR`, `SUCCESS`).  

---

## 2. Detailed Description
### Execution Flow
1. **Input Validation** – `isBlank()` checks that a file and its name are present.  
2. **File Type Validation** – `isValidFile()` ensures the file name ends with `.csv`.  
3. **Logging** – The content type of the upload is logged for audit purposes.  
4. **Service Acquisition** – `ServiceFactory` fetches an `IFileUploadService` instance.  
5. **Upload Invocation** – Depending on the method (`uploadCategories`, `uploadProducts`, `uploadManufacturers`) the service’s corresponding upload method is called, passing the file and, when required, the merchant id from the session.  
6. **Error Handling** – The service returns a `Map<Integer, List<String>>` where the key is the CSV line number and the value a list of error strings. If the map is not empty, `processErrors()` turns each entry into a human‑readable message and stores it via `addErrorMessages()`.  
7. **Result** – On success, `setSuccessMessage()` is called and the action returns `SUCCESS`. If any exception is thrown, `setTechnicalMessage()` records a generic technical error.  

### Assumptions & Constraints
* The HTTP request contains a multipart file upload with `upload`, `uploadFileName`, and `uploadContentType` fields populated by the Struts file upload interceptor.  
* The merchant id is stored in the session under `ProfileConstants.merchant` and is an `Integer`.  
* The `IFileUploadService` is responsible for file I/O and persistence; the action does not perform any direct database or file system operations.  
* The application uses Log4j 1.x (not Log4j 2 or SLF4J).  

### Architectural Notes
* The action class follows the **Thin Controller** pattern – it delegates business logic to a service layer.  
* The duplicated code in the three upload methods suggests a missed opportunity for **DRY (Don’t Repeat Yourself)** refactoring; a private helper that takes a functional interface or strategy could eliminate redundancy.  
* The class is request‑scoped (typical for Struts actions) so thread‑safety concerns are minimal.  

---

## 3. Functions/Methods
| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `uploadCategories()` | Handles CSV upload for categories. | None | `"input" | "error" | "success"` (Struts result names) | Writes to logs, interacts with session, sets user messages. |
| `uploadProducts()` | Handles CSV upload for products. | None | Same result names | Same side effects as above. |
| `uploadManufacturers()` | Handles CSV upload for manufacturers. | None | Same result names | Same side effects. |
| `processErrors(Map<Integer, List<String>>)` | Converts the error map into a list of error strings and passes them to the UI. | `errorMap` – mapping of line numbers to error details. | void | Adds error messages via `addErrorMessages()`. |
| `isBlank()` | Checks whether the file and its name are present. | None | `boolean` | None. |
| `isValidFile()` | Validates that the file has a CSV suffix. | None | `boolean` | None. |
| `getFileUploadType() / setFileUploadType(String)` | Getter/Setter for the (unused) file upload type. | None / `fileUploadType` | `String` / void | None. |
| `getUpload() / setUpload(File)` | File getter/setter (populated by Struts). | None / `upload` | `File` / void | None. |
| `getUploadContentType() / setUploadContentType(String)` | Content‑type getter/setter. | None / `uploadContentType` | `String` / void | None. |
| `getUploadFileName() / setUploadFileName(String)` | File‑name getter/setter. | None / `uploadFileName` | `String` / void | None. |

**Reusable / Utility Methods**  
* `processErrors` can be reused by any upload action that receives a line‑based error map.  
* `isBlank` and `isValidFile` encapsulate common validation logic that could be shared across multiple actions.

---

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Classic Log4j 1.x. |
| `com.salesmanager.central.BaseAction` | Internal | Struts action base class providing i18n, request/session utilities, and message helpers. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Factory for obtaining services. |
| `com.salesmanager.core.service.catalog.CatalogException` | Internal | Custom exception for catalog‑related errors. |
| `com.salesmanager.core.util.file.IFileUploadService` | Internal | Service interface for file uploads. |
| `com.salesmanager.core.util.file.csv.CSVConstants` | Internal | Holds constants like `.csv` suffix and error message. |
| `com.salesmanager.central.profile.ProfileConstants` | Internal | Contains session attribute keys. |
| Java Standard Library | Standard | `File`, `Map`, `List`, `Iterator`, etc. |

**Platform Assumptions**  
* Running in a servlet container that supports Struts‑2.  
* Log4j configuration is provided externally.  
* Session state is available (`getServletRequest().getSession()`).

---

## 5. Additional Notes & Recommendations
### Strengths
* Clear separation of concerns: the action focuses on request handling, while the service layer does the heavy lifting.  
* User feedback is detailed—errors include the CSV line number and specific problem descriptions.  
* Uses i18n (`getText`) for user‑facing messages.  

### Weaknesses / Edge Cases
1. **Code Duplication** – All three upload methods perform identical validation and error handling. Extract a common private method that accepts the specific service call (e.g., a `Consumer<File>` or a functional interface).  
2. **Limited File Validation** – `isValidFile()` only checks the suffix; it does not verify that the file exists, is readable, or that the MIME type matches. Consider adding size limits or a MIME check.  
3. **Unused Field** – `fileUploadType` is never read; either remove it or integrate it into validation logic.  
4. **Null‑Pointer Risk** – If `ServiceFactory.getService("fileUpload")` returns `null`, subsequent calls will throw NPE. Add a guard or throw a meaningful exception.  
5. **Session Dependence** – The merchant id is extracted directly from the session without null‑checking. A missing session attribute could lead to a `ClassCastException`.  
6. **Logging** – The content type is logged but not the file name or size; consider adding those for audit purposes.  
7. **Exception Granularity** – All `CatalogException`s are treated the same. If the service can throw different subclasses (e.g., `InvalidCSVException`, `DatabaseException`), differentiate messages accordingly.  

### Future Enhancements
* **Refactor to a Generic Upload Handler** – Create a single `upload(String type)` method that dispatches to the appropriate service method based on an enum.  
* **Unit Tests** – Mock `IFileUploadService` to test validation, error processing, and message setting without needing an actual file system or servlet container.  
* **Enhanced Validation** – Add checks for file size, MIME type, and existence before delegating to the service.  
* **Use SLF4J** – Switch to SLF4J for future‑proofing logging.  
* **Security** – Ensure the uploaded file is stored in a safe, temporary location and that no path traversal is possible.  

Overall, the code is functional and follows standard Struts conventions, but refactoring to reduce duplication and adding robust validation would improve maintainability and reliability.

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
package com.salesmanager.central.util.upload;

import java.io.File;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogException;
import com.salesmanager.core.util.file.IFileUploadService;
import com.salesmanager.core.util.file.csv.CSVConstants;

public class GenericFileUploadAction extends BaseAction {

	private static final long serialVersionUID = 1L;
	private static Logger logger = Logger
			.getLogger(GenericFileUploadAction.class);
	private File upload;
	private String uploadContentType;
	private String uploadFileName;
	private String fileUploadType;

	public String uploadCategories() {
		try {
			if (isBlank()) {
				setMessage(getText("error.upload.required"));
				return INPUT;
			}
			if (!isValidFile()) {
				setMessage(CSVConstants.INVALID_FILE_EXCEPTION_MSG);
				return ERROR;
			}
			logger.info("The content type uploaded is:" + uploadContentType);
			Map<Integer, List<String>> errorMap = null;
			IFileUploadService uploadService = (IFileUploadService) ServiceFactory
					.getService("fileUpload");
			errorMap = uploadService.uploadCategory(getUpload(),
					(Integer) getServletRequest().getSession().getAttribute(
							ProfileConstants.merchant));
			if (!errorMap.isEmpty()) {
				processErrors(errorMap);
				return ERROR;
			}
		} catch (CatalogException e) {
			logger.error("Error occurred while uploading", e);
			setTechnicalMessage();
		}
		setSuccessMessage();
		return SUCCESS;
	}

	public void processErrors(Map<Integer, List<String>> errorMap) {
		List<String> errorList = new ArrayList<String>();
		for (Iterator<Integer> it = errorMap.keySet().iterator(); it.hasNext();) {
			Integer lineNo = it.next();
			errorList.add("Error at Line:  " + lineNo + " "
					+ errorMap.get(lineNo).toString());
		}
		addErrorMessages(errorList);
	}

	public boolean isBlank() {
		return (upload == null || uploadFileName == null);
	}

	public String uploadProducts() {
		if (isBlank()) {
			setMessage(getText("error.upload.required"));
			return INPUT;
		}
		if (!isValidFile()) {
			setMessage(CSVConstants.INVALID_FILE_EXCEPTION_MSG);
			return ERROR;
		}
		logger.info("The content type uploaded is:" + uploadContentType);
		Map<Integer, List<String>> errorMap = null;
		IFileUploadService uploadService = (IFileUploadService) ServiceFactory
				.getService("fileUpload");
		errorMap = uploadService.uploadProducts(getUpload(),
				(Integer) getServletRequest().getSession().getAttribute(
						ProfileConstants.merchant));
		if (!errorMap.isEmpty()) {
			processErrors(errorMap);
			return ERROR;
		}
		setSuccessMessage();
		return SUCCESS;
	}

	public String uploadManufacturers() {
		if (isBlank()) {
			setMessage(getText("error.upload.required"));
			return INPUT;
		}
		if (!isValidFile()) {
			setMessage(CSVConstants.INVALID_FILE_EXCEPTION_MSG);
			return ERROR;
		}
		logger.info("The content type uploaded is:" + uploadContentType);
		Map<Integer, List<String>> errorMap = null;
		IFileUploadService uploadService = (IFileUploadService) ServiceFactory
				.getService("fileUpload");
		errorMap = uploadService.uploadManufacturers(getUpload());
		if (!errorMap.isEmpty()) {
			processErrors(errorMap);
			return ERROR;
		}
		setSuccessMessage();
		return SUCCESS;
	}

	public boolean isValidFile() {
		if (!uploadFileName.endsWith(CSVConstants.CSV_SUFFIX)) {
			return false;
		}
		return true;
	}

	public String getFileUploadType() {
		return fileUploadType;
	}

	public void setFileUploadType(String fileUploadType) {
		this.fileUploadType = fileUploadType;
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
}



```
