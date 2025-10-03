# FilesAction.java

## Review

## 1. Summary

**Purpose & Functionality**  
`FilesAction` is a Struts‑2 action (extending `CheckoutBaseAction`) that handles three user‑initiated file download scenarios in an e‑commerce checkout context:

1. **Access URL (`accessUrl`)** – Validates a tokenized download URL, checks expiration, and sets error messages if the link is invalid or expired.  
2. **View Files (`viewFiles`)** – Retrieves an order by ID, fetches its downloadable products, and prepares a list of `OrderProductDownload` objects for display.  
3. **Get File (`getFile`)** – Loads a specific downloadable file from a module bean (via Spring) and streams it to the client.

**Key Components**

| Component | Role |
|-----------|------|
| `OrderService` | Retrieve order data. |
| `MerchantService` | Retrieve merchant store metadata. |
| `DownloadFileModule` | Abstracts the file source; provides `InputStream` and filename. |
| `FileUtil` | Decodes the encoded URL token into order ID & date. |
| `SessionUtil` | Stores merchant store in the session for later use. |

**Notable Patterns & Libraries**

- **MVC (Struts‑2)** – Action class responds to HTTP requests, populates the request context.  
- **Dependency Injection** – `SpringUtil.getBean()` is used to obtain the module bean.  
- **Exception handling** – Custom `CoreException` and `OrderException` used to differentiate error types.  
- **Apache Commons Lang** – `StringUtils` for null/empty checks.  
- **Apache Log4j** – For error logging.  

---

## 2. Detailed Description

### Execution Flow

| Phase | What Happens | Key Methods |
|-------|--------------|-------------|
| **Initialization** | Action instance created by Struts‑2; properties (`orderId`, `downloadFiles`, etc.) set via request parameters. | `setOrderId`, `setFileMessage` |
| **`accessUrl`** | 1. Pull `fileId` & `lang` from the request.<br>2. Validate non‑blank `fileId`.<br>3. Decode the token (order ID & date).<br>4. Verify date is not in the past.<br>5. If expired, throw `CoreException`. | `FileUtil.getUrlTokens`, `DateFormat` parsing |
| **`viewFiles`** | 1. Load order by ID using `OrderService`.<br>2. Extract order products and map them by ID.<br>3. Load merchant store via `MerchantService` and store in the session.<br>4. Iterate products; for each downloadable product, set the product name on each `OrderProductDownload` instance. | `OrderService.getOrder`, `MerchantService.getMerchantStore`, session handling |
| **`getFile`** | 1. Retrieve `mod` and `fileId` from request.<br>2. Load the module bean (`DownloadFileModule`) using Spring.<br>3. Call `getFileInputStream` and `getFileName` to stream file to client. | `SpringUtil.getBean`, `DownloadFileModule` methods |
| **Error Handling** | If a `CoreException` (expired link) occurs, the action sets a friendly message and returns a special result (`DELAYEXPIRED`). For generic errors, it logs and returns `GENERICERROR`. | `super.setActionErrors`, `super.setTechnicalMessage` |

### Dependencies & Constraints

- **External Services** – Must be registered in `ServiceFactory` (`OrderService`, `MerchantService`).  
- **Spring Beans** – The `DownloadFileModule` bean must be defined with name matching the `mod` request parameter.  
- **Token Format** – The URL token must encode `ID` and `DATE` keys; the parsing logic depends on `FileUtil.getUrlTokens`.  
- **Locale** – `lang` parameter is accepted but not used; likely intended for i18n support.  

---

## 3. Functions / Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `getInputStream()` | Getter for file stream (used by Struts file download interceptor). | none | `InputStream` | none |
| `accessUrl()` | Validates download link token and checks expiration. | none (uses request params) | `String` (result code) | Sets `fileMessage`, throws `CoreException` |
| `viewFiles()` | Loads order and prepares downloadable files list for UI. | none | `String` (result) | Populates `downloadFiles`, `order`, sets session merchant store |
| `getFile()` | Streams the requested file to the client. | none | `String` (result) | Sets `inputStream`, `fileName`, handles `OrderException` |
| `getFileName()` | Getter for file name (used in UI or interceptor). | none | `String` | none |
| `setFileName(String)` | Setter for `fileName`. | `fileName` | void | updates field |
| `getFileMessage()` | Getter for user‑visible message. | none | `String` | none |
| `setFileMessage(String)` | Setter for `fileMessage`. | `fileMessage` | void | updates field |
| `getDownloadFiles()` | Getter for downloadable files collection. | none | `Collection` | none |
| `setDownloadFiles(Collection)` | Setter for collection. | `downloadFiles` | void | updates field |
| `getOrderId()` | Getter for order ID. | none | `String` | none |
| `setOrderId(String)` | Setter for order ID. | `orderId` | void | updates field |
| `getOrder()` | Getter for order entity. | none | `Order` | none |
| `setOrder(Order)` | Setter for order entity. | `order` | void | updates field |

*Reusable utilities*: `FileUtil.getUrlTokens` and the use of `SessionUtil.setMerchantStore` are generic helpers that can be used elsewhere in the application.

---

## 4. Dependencies

| Library / Framework | Purpose | Standard / 3rd‑party |
|---------------------|---------|----------------------|
| `org.apache.commons.lang.StringUtils` | String null/blank checks | 3rd‑party |
| `org.apache.log4j.Logger` | Logging | 3rd‑party |
| `com.salesmanager.core.*` | Core entities (`Order`, `OrderProduct`, etc.), services (`OrderService`, `MerchantService`), utilities (`FileUtil`, `SessionUtil`) | In‑house |
| `com.salesmanager.checkout.*` | Base action class | In‑house |
| `org.apache.struts2` | Struts‑2 action framework (implied by extending `CheckoutBaseAction`) | In‑house / Struts |
| `org.springframework` (via `SpringUtil`) | Bean lookup for modules | 3rd‑party |

No native Java EE APIs are used directly; the code relies on a custom service factory pattern.

---

## 5. Additional Notes

### Strengths
- **Clear separation of concerns** – Each action method focuses on a single operation.  
- **Use of custom exceptions** – Distinguishes business errors (`CoreException`, `OrderException`) from generic system errors.  
- **Reusability** – Utility methods (`FileUtil`, `SessionUtil`) are well abstracted.  

### Weaknesses & Edge Cases
1. **Hard‑coded Result Strings**  
   The action returns literal strings like `"DELAYEXPIRED"`, `"GENERICERROR"`, etc. In a full Struts configuration these must be mapped; missing mappings will result in runtime errors.

2. **Missing Locale Handling**  
   The `lang` parameter is read but never used. If i18n support is required, the code should pass it to `FileUtil` or locale‑aware services.

3. **Null Checks & Logging**  
   - In `getFile()`, the module bean is fetched before checking `mod` against null. If `mod` is null, `SpringUtil.getBean(null)` may throw an exception.  
   - The error log for `fileId` being null writes only a message; the method continues executing and may produce NPEs.

4. **Date Comparison Logic**  
   The expiration check compares parsed date to `new Date(today.getTime())` which is effectively `new Date()`; this will treat any date *before* the current instant as expired. It may be acceptable but should be documented.

5. **Thread Safety**  
   The action holds instance variables (`downloadFiles`, `order`, etc.) that are reused across requests in Struts‑2. If the action is shared across threads (depending on configuration), this could cause concurrency issues. Struts‑2 usually creates a new instance per request, but documentation should confirm.

6. **Exception Specificity**  
   Catching `Exception` broadly in `accessUrl` and `viewFiles` can mask other runtime errors. It is safer to catch only the expected exceptions (e.g., `ParseException`, `CoreException`).

7. **Resource Leak**  
   `inputStream` is never closed; it is expected to be closed by the Struts‑2 file download interceptor. Verify that the interceptor handles stream closing correctly.

### Potential Enhancements
- **Validation Framework** – Use Struts validation annotations or a custom validator instead of manual `StringUtils.isBlank` checks.  
- **Internationalization** – Pull the `lang` parameter into the request context and use it to localize error messages.  
- **Logging Granularity** – Add more contextual logs (e.g., order ID, merchant ID) to aid troubleshooting.  
- **Unit Tests** – Create unit tests for each method, mocking services and the Spring bean retrieval.  
- **Refactor to Service Layer** – Move business logic (order retrieval, file permission checks) into a dedicated service to keep the action thin.  
- **Security Improvements** – Verify that the `fileId` token cannot be tampered with; consider using JWT or HMAC for stronger integrity.  

Overall, the code is straightforward and follows established patterns for Struts‑2 actions. Addressing the above concerns will improve robustness, maintainability, and security.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 25, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.checkout.files;

import java.io.InputStream;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.checkout.CheckoutBaseAction;
import com.salesmanager.core.CoreException;
import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.OrderProduct;
import com.salesmanager.core.entity.orders.OrderProductDownload;
import com.salesmanager.core.module.model.application.DownloadFileModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.order.OrderException;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class FilesAction extends CheckoutBaseAction {

	private Logger log = Logger.getLogger(FilesAction.class);

	private Collection downloadFiles;

	private InputStream inputStream;
	private String fileName;

	private String fileMessage;

	private String orderId = null;

	private Order order = null;

	public InputStream getInputStream() {
		return inputStream;
	}

	public String accessUrl() {

		try {

			// parse file download request
			String fileId = getServletRequest().getParameter("fileId");
			String lang = getServletRequest().getParameter("lang");

			if (StringUtils.isBlank(fileId)) {
				List msg = new ArrayList();
				msg.add(getText("error.downloadurl.invalid"));
				super.setActionErrors(msg);
				return "GENERICERROR";
			}

			Map fileInfo = FileUtil.getUrlTokens(fileId);

			String order = (String) fileInfo.get("ID");
			String date = (String) fileInfo.get("DATE");

			orderId = order;

			// Compare the date
			Date today = new Date();
			DateFormat d = new SimpleDateFormat("yyyy-MM-dd");
			Date dt = null;

			dt = d.parse(date);

			if (dt.before(new Date(today.getTime()))) {
				// expired
				CoreException excpt = new CoreException(
						ErrorConstants.DELAY_EXPIRED);
				throw excpt;
			}

		} catch (Exception e) {

			if (e instanceof CoreException) {
				String message = getText("error.downloadurl.expired");
				this.setFileMessage(message);
				return "DELAYEXPIRED";
			} else {
				log.error(e);
				super.setTechnicalMessage();
				return "GENERICERROR";
			}

		}

		return SUCCESS;

	}

	public String viewFiles() {

		try {

			if (this.getOrderId() == null) {
				super.setTechnicalMessage();
				return "GENERICERROR";
			}

			// need MerchantStore, create a Locale

			OrderService oservice = (OrderService) ServiceFactory
					.getService(ServiceFactory.OrderService);

			order = oservice.getOrder(Long.parseLong(this.getOrderId()));

			Set products = order.getOrderProducts();
			Iterator i = products.iterator();
			Map productsMap = new HashMap();
			List opList = new ArrayList();
			while (i.hasNext()) {
				OrderProduct op = (OrderProduct) i.next();
				productsMap.put(op.getOrderProductId(), op);
				opList.add(op);
			}

			// MerchantStore
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice.getMerchantStore(order
					.getMerchantId());
			SessionUtil.setMerchantStore(store, getServletRequest());
			getServletRequest().setAttribute("MERCHANT__STORE", store);

			Set st1 = order.getOrderProducts();

			if (st1 != null && st1.size() > 0) {
				Iterator opit = st1.iterator();
				while (opit.hasNext()) {
					OrderProduct op = (OrderProduct) opit.next();

					if (op.getDownloads() != null
							&& op.getDownloads().size() > 0) {
						// check if download expired or downloadcount==0

						Set opdSet = op.getDownloads();

						downloadFiles = opdSet;

						if (downloadFiles != null && downloadFiles.size() > 0) {
							Iterator dfIterator = downloadFiles.iterator();
							while (dfIterator.hasNext()) {
								OrderProductDownload opd = (OrderProductDownload) dfIterator
										.next();
								OrderProduct opp = (OrderProduct) productsMap
										.get(opd.getOrderProductId());
								if (opp != null) {
									opd.setProductName(opp.getProductName());
								} else {
									opd.setProductName(opd
											.getOrderProductFilename());
								}
							}
						}

					}

				}
			}

		} catch (Exception e) {
			if (e instanceof CoreException) {
				String message = getText("error.downloadurl.expired");
				this.setFileMessage(message);
				return "DELAYEXPIRED";
			} else {
				log.error(e);
				super.setTechnicalMessage();
				return "GENERICERROR";
			}
		}

		return SUCCESS;

	}

	public String getFile() {

		try {

			String mod = getServletRequest().getParameter("mod");
			String fileid = getServletRequest().getParameter("fileId");
			if (fileid == null) {
				log.error("fileId is null !!!");
			} else {

				DownloadFileModule module = (DownloadFileModule) SpringUtil
						.getBean(mod);

				if (mod == null) {
					log.error("Mod is null from the URL");
					super.setTechnicalMessage();
					return "GENERICERROR";
				}

				inputStream = module.getFileInputStream(getServletRequest());
				fileName = module.getFileName();

			}

		} catch (Exception e) {
			if (e instanceof OrderException) {
				String message = getText("label.order.download.maximumdownloadreached");
				this.setFileMessage(message);
				return "MAXIMUMDOWNLOADREACHED";
			} else {
				log.error(e);
				super.setTechnicalMessage();
				return "GENERICERROR";
			}
		}

		return SUCCESS;

	}

	public String getFileName() {
		return fileName;
	}

	public void setFileName(String fileName) {
		this.fileName = fileName;
	}

	public String getFileMessage() {
		return fileMessage;
	}

	public void setFileMessage(String fileMessage) {
		this.fileMessage = fileMessage;
	}

	public Collection getDownloadFiles() {
		return downloadFiles;
	}

	public void setDownloadFiles(Collection downloadFiles) {
		this.downloadFiles = downloadFiles;
	}

	public String getOrderId() {
		return orderId;
	}

	public void setOrderId(String orderId) {
		this.orderId = orderId;
	}

	public Order getOrder() {
		return order;
	}

	public void setOrder(Order order) {
		this.order = order;
	}

}



```
