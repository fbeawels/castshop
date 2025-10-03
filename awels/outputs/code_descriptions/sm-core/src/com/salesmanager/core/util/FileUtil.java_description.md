# FileUtil.java

## Review

## 1. Summary  

`FileUtil` is a utility façade that builds URLs, file paths, and decrypts tokens for a multi‑tenant e‑commerce platform (SalesManager).  
Key responsibilities:

| Feature | Purpose |
|---------|---------|
| URL generation | Builds links for invoices, order downloads, admin password reset, and catalogue pages. |
| Path resolution | Provides absolute and relative paths for media (images, flash, branding, product files, sitemap, downloads). |
| Token parsing | Decrypts “fileId” style tokens into map structures. |
| Content type categorisation | Determines whether a MIME type is an image, flash or generic file. |

**Design patterns / libraries**  
* Singleton‑like static façade – all methods are `static`.  
* Uses the **Apache Commons Configuration** API for property loading (`Configuration`).  
* **Apache Commons Lang** (`StringUtils`).  
* `EncryptionUtil` and `ReferenceUtil` are custom helpers for encryption and URI construction.  
* Service locator pattern via `ServiceFactory` to obtain `MerchantService`.  

The code is tightly coupled to the platform’s configuration structure and business logic (merchant IDs, order IDs, user IDs, etc.).

---

## 2. Detailed Description  

### Execution Flow  

1. **Configuration** – The first line of the class loads the media base path (`IMAGE_PATH`) from `PropertiesUtil.getConfiguration()`.  
2. **URL helpers** – Each `getXxxUrl` method:
   * Pulls the relevant domain/URI fragments from configuration.
   * Retrieves the merchant’s `MerchantStore` via `MerchantService`.
   * Builds a *token* string containing identifiers and a calculated expiry date.
   * Encrypts the token with a fixed key from `SecurityConstants`.
   * Concatenates the base URL, action path, and query parameters.
3. **Token parsers** – `getInvoiceTokens`, `getUrlTokens`, `getFileDownloadFileTokens`:
   * Decrypt the token.
   * Tokenise by `|`.
   * Map to a `Map<String,String>` with specific keys.
4. **Path helpers** – Methods such as `getLargeProductImagePath` or `getFileTreeBinPath` simply read configuration and compose strings with merchant IDs and image prefixes.
5. **Content type** – `getContentCategoryType` checks a MIME string against configured lists.

### Dependencies & Assumptions  

| Dependency | Nature | Comments |
|------------|--------|----------|
| `PropertiesUtil.getConfiguration()` | Third‑party (likely a wrapper around Apache Commons Configuration) | Assumes a single shared `Configuration` instance. |
| `MerchantService` | Application service | Retrieved via a `ServiceFactory` which acts as a Service Locator. |
| `EncryptionUtil` | Custom encryption utility | Uses a key from `SecurityConstants.idConstant`. The encryption is deterministic; no IV or salt is used. |
| `ReferenceUtil` | Custom URI builder | Provides base URLs (`buildCartUri`, `buildCentralUri`, etc.). |
| `DateUtil` | Custom date formatting | Assumes ISO‑8601 or a specific format. |
| `HttpServletRequest` | Servlet API | Only used in `getDefaultCataloguePageUrl`. |

The code assumes:
* All configuration keys exist; missing keys trigger `NullPointerException` or malformed URLs.
* `customer` and `information` objects are non‑null where expected; otherwise null‑check logic is missing in some places (e.g., `getInvoiceUrl` uses `customer.getCustomerLang()` without null‑checking `customer` after the early `if (customer != null)`).
* The token format is strictly “value1|value2|…”. Any deviation causes the parser to throw a generic `Exception`.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `getInvoiceUrl(Order, Customer)` | Builds a URL to view an invoice. | `Order`, `Customer` | `String` URL | Reads config, encrypts token |
| `getDefaultCataloguePageUrl(MerchantStore, HttpServletRequest)` | Builds default catalogue page URL. | `MerchantStore`, `HttpServletRequest` | `String` URL | Reads config |
| `getAdminPasswordResetUrl(MerchantUserInformation, MerchantStore)` | Builds URL for admin password reset. | `MerchantUserInformation`, `MerchantStore` | `String` URL | Reads config, encrypts token |
| `getOrderDownloadFileUrl(Order, Customer)` | URL to download order files. | `Order`, `Customer` | `String` URL | Reads config, encrypts token |
| `getMediaPath()` | Returns media base directory. | None | `String` | Reads config |
| `getInternalDownloadFileUrl(int, long)` | URL to download a file by its internal ID. | `merchantId`, `downloadId` | `String` URL | Reads config, encrypts token |
| `getInvoiceTokens(String)` | Decrypts and parses invoice token. | `token` | `Map<String,String>` | Throws `Exception` on error |
| `getUrlTokens(String)` | Decrypts and parses generic URL token. | `token` | `Map<String,String>` | Throws `Exception` on error |
| `getFileDownloadFileTokens(String)` | Decrypts and parses file‑download token. | `token` | `Map<String,String>` | Throws `Exception` on error |
| `getStoreLogoPath(int, String)` | Builds path to a store’s logo. | `merchantId`, `storeLogo` | `String` | Reads config |
| `getLargeProductImagePath(int, String)` | Path for large product image. | `merchantId`, `productImage` | `String` | Reads config |
| `getSmallProductImagePath(int, String)` | Path for small product image. | `merchantId`, `productImage` | `String` | Reads config |
| `getProductImagePath(int, String)` | Path for original product image. | `merchantId`, `productImage` | `String` | Reads config |
| `getFileTreeBinPath()` | Root path for merchant file tree. | None | `String` | Reads config |
| `getFileTreeBinPathForImages(int)` | Path for merchant images. | `merchantId` | `String` | Calls `getFileTreeBinPath` |
| `getFileTreeBinPathForFlash(int)` | Path for merchant flash files. | `merchantId` | `String` | Calls `getFileTreeBinPath` |
| `getBrandingFilePath()`, `getProductFilePath()`, `getSiteMapFilePath()` | Absolute paths for various file types. | None | `String` | Reads config |
| `getSiteMapUrl()` | URL for sitemap. | None | `String` | Reads config |
| `getDownloadFilePath()` | Absolute download directory. | None | `String` | Reads config |
| `getBinServerUrl()` / `getBinServerUrl(int, boolean)` | Base URL for binary server. | (optional) `merchantId`, `isImage` | `String` | Reads config |
| `getContentCategoryType(String)` | Determines MIME category. | `fileContentType` | `ContentCategoryType` | Reads config |

**Reusable utilities**  
`EncryptionUtil` and `ReferenceUtil` are used by many methods; if either fails, all URL generators break.

---

## 4. Dependencies  

| Library / API | Purpose | Standard / Third‑party | Notes |
|---------------|---------|------------------------|-------|
| `org.apache.commons.configuration.Configuration` | Property access | Third‑party (Commons Configuration) | Centralised config. |
| `org.apache.commons.lang.StringUtils` | String safety | Third‑party (Commons Lang) | Basic null‑check helpers. |
| `javax.servlet.ServletContext`, `HttpServletRequest` | Servlet integration | Standard (Java EE) | Only used in one method. |
| `java.util` classes (Calendar, Date, Map, HashMap, StringTokenizer) | Standard utilities | Standard | `StringTokenizer` is legacy; could use `String.split`. |
| `com.salesmanager.core.*` | Domain entities & services | Application-specific | Tight coupling. |
| `EncryptionUtil`, `ReferenceUtil`, `DateUtil` | Custom helpers | Application-specific | Encryption is deterministic and potentially insecure. |

No external frameworks (Spring, Hibernate, etc.) are directly referenced, but the code assumes a broader application context.

---

## 5. Additional Notes & Recommendations  

### 5.1 Security & Token Handling  
* **Deterministic encryption**: `EncryptionUtil.encrypt` appears to use a static key without IV or salt. This makes tokens predictable and vulnerable to replay or brute‑force attacks.  
* **Token lifetime**: Expiry dates are computed server‑side but not validated anywhere. The consumer of the URL must validate the timestamp.  
* **Hard‑coded key**: `SecurityConstants.idConstant` is a static string; rotating it would require changing code or configuration.  

### 5.2 Error Handling  
* Methods that build URLs often throw generic `Exception`. Better to throw a domain‑specific unchecked exception (e.g., `FileUtilException`) or propagate the underlying cause.  
* Missing configuration values cause `NullPointerException` or malformed URLs; add explicit checks and informative errors.

### 5.3 Null‑Safety  
* Several places assume non‑null inputs (`customer.getCustomerLang()`, `information.getUserlang()`). Defensive programming would reduce NPE risk.  
* `getInvoiceUrl` checks `if (customer != null)` only for language; however, the code later accesses `customer.getCustomerId()` regardless.

### 5.4 Code Style & Maintainability  
* **String concatenation**: Uses `StringBuilder` extensively but could be simplified with Java 8+ `String.format` or `UriComponentsBuilder` (Spring).  
* **Hard‑coded delimiters**: The token separator `|` is magic. Define a constant (`TOKEN_DELIMITER`).  
* **Legacy classes**: `StringTokenizer` can be replaced with `String.split`.  
* **Configuration loading**: The class reads configuration multiple times; caching or passing a `Configuration` object could improve performance.  
* **Magic numbers**: The download expiration days (`2`) are hard‑coded; should be read from config or a constant.

### 5.5 Extensibility  
* **Multiple media types**: Currently only images and flash are handled. Adding new media types would require extending `ContentCategoryType` and updating `getContentCategoryType`.  
* **Multi‑language URLs**: The `lang` parameter is appended but the logic for defaulting to user language is inconsistent. A dedicated `UrlBuilder` service could centralise this logic.  
* **Testing**: The static nature makes unit testing difficult. Extracting configuration and service lookups into interfaces would allow mocking.

### 5.6 Suggested Refactor Points  

| Area | Action |
|------|--------|
| **Token Generation / Validation** | Create a dedicated `TokenService` that encapsulates encryption, expiry calculation, and validation. |
| **Path Construction** | Use a `PathBuilder` that accepts a `MerchantStore` and an enum of media type, returning the absolute or relative path. |
| **Error Handling** | Replace generic `Exception` with a hierarchy of unchecked exceptions. |
| **Null Safety** | Add `Objects.requireNonNull` checks and guard clauses. |
| **Constants** | Centralise delimiters, default values, and configuration keys in a `FileUtilConstants` interface. |
| **Configuration** | Inject `Configuration` via constructor or a service locator pattern; avoid static lookup in every method. |
| **Logging** | Add SLF4J logging for debug / error scenarios. |

---

### 5.7 Edge Cases Not Handled  

| Scenario | Current Behavior | Recommended Fix |
|----------|------------------|-----------------|
| Token contains fewer than required parts | `StringTokenizer` silently stops, resulting in empty strings, which triggers an exception later. | Validate token length before parsing. |
| Expiry date in the past | URL still works until consumer checks; could be accepted or rejected. | Validate timestamp and return 403/404 if expired. |
| Missing config key (`core.bin.images.contenttypes`) | NPE or empty string, leading to `INVALID` category. | Provide default content type list. |
| Very long file names or IDs | String concatenation may produce excessively long URLs. | Apply URL encoding. |
| Duplicate keys in configuration | Overwrites earlier values, unpredictable behavior. | Document and enforce unique keys. |

---

### 5.8 Future Enhancements  

1. **Internationalisation** – Move language handling into a dedicated service.  
2. **Rate limiting** – Throttle URL generation to prevent abuse.  
3. **Audit logging** – Log each generated URL with user context for traceability.  
4. **Unit tests** – Create comprehensive tests for each helper using mocks for `Configuration`, `ServiceFactory`, and `EncryptionUtil`.  
5. **Documentation** – Add Javadoc comments describing token format, expiration policy, and error semantics.  

---

**Overall Verdict**  
`FileUtil` serves a clear purpose within the SalesManager ecosystem and is functionally sufficient for building URLs and paths. However, its current implementation is fragile: it relies heavily on global configuration, uses deterministic encryption, and lacks robust error handling. Refactoring the token logic into a dedicated service, centralising configuration access, and adding defensive coding practices would greatly improve security, testability, and maintainability.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 24, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.util;

import java.util.Calendar;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.StringTokenizer;

import javax.servlet.ServletContext;
import javax.servlet.http.HttpServletRequest;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.constants.SecurityConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;

public class FileUtil {

	private final static String IMAGE_PATH = PropertiesUtil.getConfiguration()
			.getString("core.store.mediaurl", "/media");

	public enum ContentCategoryType {
		IMAGE, FLASH, INVALID, FILE
	}

	public static String getInvoiceUrl(Order order, Customer customer)
			throws Exception {

		Configuration conf = PropertiesUtil.getConfiguration();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(order.getMerchantId());

		// ***build fileid***
		StringBuffer urlconstruct = new StringBuffer();
		StringBuffer invoiceurl = new StringBuffer();

		// order id and, expiration date and language
		urlconstruct.append(order.getOrderId()).append("|").append(
				customer.getCustomerId());

		String lang = conf.getString("core.system.defaultlanguage", "en");
		if (customer != null) {
			lang = customer.getCustomerLang();
		}

		String file = EncryptionUtil.encrypt(EncryptionUtil
				.generatekey(SecurityConstants.idConstant), urlconstruct
				.toString());

		invoiceurl.append(ReferenceUtil.buildCartUri(store)).append(
				conf.getString("core.salesmanager.core.viewInvoiceAction"))
				.append("?fileId=").append(file);

		return invoiceurl.toString();

	}
	
	public static String getDefaultCataloguePageUrl(MerchantStore store, HttpServletRequest request) throws Exception {
		
		
		
		
		String storeDomain = ReferenceUtil.getUnSecureDomain(store);
		
		
		Configuration conf = PropertiesUtil.getConfiguration();
		String shop = conf.getString("core.salesmanager.catalog.url");
		
		return new StringBuilder().append(storeDomain).append(shop).toString();
	}
	
	public static String getAdminPasswordResetUrl(MerchantUserInformation information, MerchantStore store) throws Exception {
		
		Configuration conf = PropertiesUtil.getConfiguration();
		
		StringBuffer urlconstruct = new StringBuffer();
		StringBuffer downloadurl = new StringBuffer();
		Calendar endDate = Calendar.getInstance();
		endDate.add(Calendar.DATE, conf.getInt(
				"core.product.file.downloadmaxdays", 2)); // add 2 days
		Date denddate = endDate.getTime();
		String sedate = DateUtil.formatDate(denddate);

		// order id and, expiration date and language
		urlconstruct.append(information.getMerchantUserId()).append("|").append(
				sedate);

		String lang = conf.getString("core.system.defaultlanguage", "en");
		if (information != null) {
			lang = information.getUserlang();
		}

		String file = EncryptionUtil.encrypt(EncryptionUtil
				.generatekey(SecurityConstants.idConstant), urlconstruct
				.toString());

		downloadurl.append(ReferenceUtil.buildCentralUri(store)).append("/anonymous" +
				conf.getString("core.salesmanager.core.resetPasswordAction"))
				.append("?urlId=").append(file).append("&lang=").append(lang);

		return downloadurl.toString();
		
		
	}

	public static String getOrderDownloadFileUrl(Order order, Customer customer)
			throws Exception {

		Configuration conf = PropertiesUtil.getConfiguration();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(order.getMerchantId());

		// ***build fileid***
		StringBuffer urlconstruct = new StringBuffer();
		StringBuffer downloadurl = new StringBuffer();
		Calendar endDate = Calendar.getInstance();
		endDate.add(Calendar.DATE, conf.getInt(
				"core.product.file.downloadmaxdays", 2)); // add 2 days
		Date denddate = endDate.getTime();
		String sedate = DateUtil.formatDate(denddate);

		// order id and, expiration date and language
		urlconstruct.append(order.getOrderId()).append("|").append("|").append(
				sedate);

		String lang = conf.getString("core.system.defaultlanguage", "en");
		if (customer != null) {
			lang = customer.getCustomerLang();
		}

		String file = EncryptionUtil.encrypt(EncryptionUtil
				.generatekey(SecurityConstants.idConstant), urlconstruct
				.toString());

		downloadurl.append(ReferenceUtil.buildCheckoutUri(store)).append(
				conf.getString("core.salesmanager.core.viewFilesAction"))
				.append("?fileId=").append(file).append("&lang=").append(lang);

		return downloadurl.toString();

	}
	
	public static String getMediaPath() {
		Configuration conf = PropertiesUtil.getConfiguration();
		return conf.getString("core.bin.mediapath");
	}

	public static String getInternalDownloadFileUrl(int merchantId,
			long downloadId) throws Exception {

		Configuration conf = PropertiesUtil.getConfiguration();

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(merchantId);

		// ***build fileid***
		StringBuffer urlconstruct = new StringBuffer();
		StringBuffer downloadurl = new StringBuffer();
		Calendar endDate = Calendar.getInstance();
		endDate.add(Calendar.DATE, conf.getInt(
				"core.product.file.downloadmaxdays", 2)); // add 2 days
		Date denddate = endDate.getTime();
		String sedate = DateUtil.formatDate(denddate);

		urlconstruct.append(downloadId).append("|").append(sedate).append("|")
				.append(merchantId);

		String file = EncryptionUtil.encrypt(EncryptionUtil
				.generatekey(SecurityConstants.idConstant), urlconstruct
				.toString());

		downloadurl.append(ReferenceUtil.buildCheckoutUri(store)).append(
				conf.getString("core.salesmanager.core.downloadFileAction"))
				.append("?fileId=").append(file).append("&mod=productfile");

		return downloadurl.toString();

	}

	/**
	 * Decrypt & Parses a url with tokens for displaying invoice page
	 * 
	 * @param token
	 * @return
	 * @throws Exception
	 */
	public static Map<String, String> getInvoiceTokens(String token)
			throws Exception {

		if (StringUtils.isBlank(token)) {
			throw new Exception("token (url parameter) is empty");
		}

		String decrypted = EncryptionUtil.decrypt(EncryptionUtil
				.generatekey(SecurityConstants.idConstant), token);

		StringTokenizer st = new StringTokenizer(decrypted, "|");
		String orderId = "";
		String customerId = "";

		int i = 0;
		while (st.hasMoreTokens()) {
			String t = st.nextToken();
			if (i == 0) {
				orderId = t;
			} else if (i == 1) {
				customerId = t;
			} else {
				break;
			}
			i++;
		}

		if (StringUtils.isBlank(orderId) || StringUtils.isBlank(customerId)) {
			throw new Exception("Invalid URL parameters for getInvoiceTokens "
					+ token);
		}

		Map response = new HashMap();
		response.put("order.orderId", orderId);
		response.put("customer.customerId", customerId);

		return response;
	}

	/**
	 * Decrypt & Parses a url with tokens for displaying url page
	 * needs 2 tokens, an id (first) and ten a dat
	 * @param tokens
	 * @return
	 * @throws Exception
	 */
	public static Map<String, String> getUrlTokens(String token)
			throws Exception {

		String decrypted = EncryptionUtil.decrypt(EncryptionUtil
				.generatekey(SecurityConstants.idConstant), token);

		StringTokenizer st = new StringTokenizer(decrypted, "|");
		String id = "";
		String date = "";

		int i = 0;
		while (st.hasMoreTokens()) {
			String t = st.nextToken();
			if (i == 0) {
				id = t;
			} else if (i == 1) {
				date = t;
			} else {
				break;
			}
			i++;
		}

		if (StringUtils.isBlank(id) || StringUtils.isBlank(date)) {
			throw new Exception(
					"Invalid URL parameters for FileUtil.getUrlTokens "
							+ token);
		}

		Map response = new HashMap();
		response.put("ID", id);
		response.put("DATE", date);

		return response;
	}

	/**
	 * Decrypt and parses file token
	 * 
	 * @param tokens
	 * @return
	 * @throws Exception
	 */
	public static Map<String, String> getFileDownloadFileTokens(String token)
			throws Exception {

		String decrypted = EncryptionUtil.decrypt(EncryptionUtil
				.generatekey(SecurityConstants.idConstant), token);

		StringTokenizer st = new StringTokenizer(decrypted, "|");
		String downloadId = "";
		String date = "";
		String merchantId = "";

		int i = 0;
		while (st.hasMoreTokens()) {
			String t = st.nextToken();
			if (i == 0) {
				downloadId = t;
			} else if (i == 1) {
				date = t;
			} else if (i == 2) {
				merchantId = t;
			} else {
				break;
			}
			i++;
		}

		if (StringUtils.isBlank(downloadId) || StringUtils.isBlank(date)
				|| StringUtils.isBlank(merchantId)) {
			throw new Exception(
					"Invalid URL parameters for FileUtil.getFileDownloadFileTokens "
							+ token);
		}

		Map response = new HashMap();
		response.put("ID", downloadId);
		response.put("DATE", date);
		response.put("MERCHANTID", merchantId);

		return response;

	}

	public static String getStoreLogoPath(int merchantId, String storeLogo) {


		Configuration config = PropertiesUtil.getConfiguration();
		return new StringBuffer().append(IMAGE_PATH).append(
				config.getString("core.store.brandingsuri")).append(merchantId)
				.append("/header/").append(storeLogo).toString();

	}



	public static String getLargeProductImagePath(int merchantId,
			String productImage) {
		if (StringUtils.isBlank(productImage)) {
			return "";
		} else {

			Configuration config = PropertiesUtil.getConfiguration();


			return new StringBuffer()
					.append(IMAGE_PATH)
					.append(config.getString("core.products.images.uri"))
					.append("/")
					.append(merchantId)
					.append("/")
					.append(config.getString("core.product.image.large.prefix"))
					.append("-").append(productImage).toString();
		}
	}

	public static String getSmallProductImagePath(int merchantId,
			String productImage) {



		if (StringUtils.isBlank(productImage)) {
			return "";
		} else {
			Configuration config = PropertiesUtil.getConfiguration();
			return new StringBuffer()
					.append(IMAGE_PATH)
					.append(config.getString("core.products.images.uri"))
					.append("/")
					.append(merchantId)
					.append("/")
					.append(config.getString("core.product.image.small.prefix"))
					.append("-").append(productImage).toString();
		}
	}

	/**
	 * returns original image
	 * 
	 * @param merchantId
	 * @param productImage
	 * @return
	 */
	public static String getProductImagePath(int merchantId, String productImage) {
		if (StringUtils.isBlank(productImage)) {
			return "";
		} else {


			Configuration config = PropertiesUtil.getConfiguration();
			return new StringBuffer().append(IMAGE_PATH).append(
					config.getString("core.products.images.uri")).append("/")
					.append(merchantId).append("/").append(productImage)
					.toString();
		}
	}

	/**
	 * getFileTreeBinPath returns the Path to the bin folder of the merchant(specific to
	 * merchantId) and also according to the type i.e. if isImage is true then
	 * it gives the image path of that particular merchant else it gives the
	 * shock-wave path.
	 * @param context
	 *            ServletContext
	 * @param merchantId
	 *            int
	 * @param isImage
	 *            boolean
	 * @return String path
	 */
	public static String getFileTreeBinPath() {
		Configuration config = PropertiesUtil.getConfiguration();
		StringBuilder builder = new StringBuilder().append(getMediaPath()).append(config
				.getString("core.bin.filetree.filefolder"));
		return builder.toString();
	}

	public static String getFileTreeBinPathForImages(int merchantId) {
		StringBuilder builder = new StringBuilder();
		builder.append(getFileTreeBinPath()).append("/").append(
				"images").append("/").append(merchantId)
				.append("/");
		return builder.toString();
	}
	
	public static String getFileTreeBinPathForFlash(int merchantId) {
		StringBuilder builder = new StringBuilder();
		builder.append(getFileTreeBinPath()).append("/").append(
				"flash").append("/").append(merchantId)
				.append("/");
		return builder.toString();
	}
	
	/**
	 * Absolute file path
	 * @return
	 */
	public static String getBrandingFilePath() {
		Configuration config = PropertiesUtil.getConfiguration();
		return new StringBuilder().append(getMediaPath()).append(config.getString("core.branding.cart.filefolder")).toString();
	}
	
	/**
	 * Absolute file path
	 * @return
	 */
	public static String getProductFilePath() {
		Configuration config = PropertiesUtil.getConfiguration();
		return new StringBuilder().append(getMediaPath()).append(config.getString("core.product.image.filefolder")).toString();
	}
	
	/**
	 * Absolute file path
	 * @return
	 */
	public static String getSiteMapFilePath() {
		Configuration config = PropertiesUtil.getConfiguration();
		return new StringBuilder().append(getMediaPath()).append(config.getString("core.sitemap.filefolder")).toString();
	}
	
	public static String getSiteMapUrl() {
		Configuration config = PropertiesUtil.getConfiguration();
		StringBuilder builder = new StringBuilder().append(IMAGE_PATH).append(
				config.getString("core.sitemap.uri")).append("/");
		return builder.toString();
	}
	
	
	/**
	 * Download files path (absolute path)
	 * @return
	 */
	public static String getDownloadFilePath() {
		Configuration config = PropertiesUtil.getConfiguration();
		String downloadPath = config.getString("core.download.path");
		return downloadPath;
	}

	public static String getBinServerUrl() {
		Configuration config = PropertiesUtil.getConfiguration();
		StringBuilder builder = new StringBuilder().append(IMAGE_PATH).append(
				config.getString("core.bin.uri")).append("/");
		return builder.toString();
	}

	public static String getBinServerUrl(int merchantId, boolean isImage) {
		Configuration config = PropertiesUtil.getConfiguration();
		StringBuilder builder = new StringBuilder().append(IMAGE_PATH).append(
				config.getString("core.bin.uri")).append("/").append(
				(isImage ? "images" : "flash")).append("/").append(merchantId)
				.append("/");
		return builder.toString();
	}

	public static ContentCategoryType getContentCategoryType(
			String fileContentType) {
		Configuration config = PropertiesUtil.getConfiguration();
		String imageContentTypes = config
				.getString("core.bin.images.contenttypes");
		String flashContentType = config.getString("core.shockwaveformat");
		String filesContentType = config
				.getString("core.bin.files.contenttypes");
		if (fileContentType == null || fileContentType.trim() == "") {
			return ContentCategoryType.INVALID;
		}
		if (imageContentTypes.toLowerCase().contains(
				fileContentType.toLowerCase())) {
			return ContentCategoryType.IMAGE;
		} else if (flashContentType.toLowerCase().equals(
				fileContentType.toLowerCase())) {
			return ContentCategoryType.FLASH;
		} else if (filesContentType.toLowerCase().contains(
				filesContentType.toLowerCase())) {
			return ContentCategoryType.FILE;
		} else {
			return ContentCategoryType.INVALID;
		}
	}

}



```
