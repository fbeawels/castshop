# LocalFileImpl.java

## Review

## 1. Summary

`LocalFileImpl` is an implementation of two application‑specific interfaces (`ProductFileModule` & `DownloadFileModule`).  
It is responsible for:

1. **Serving downloadable product files** – retrieving the file from disk, validating the token, tracking download counts, and streaming the file to the client.  
2. **Managing files on the local filesystem** – deleting files/directories, copying/moving temporary uploads to the permanent media folder, and providing download URLs.

Key elements:

| Component | Purpose |
|-----------|---------|
| `FileUtil` | Helper for path resolution and token parsing |
| `OrderService` | Persistence of `OrderProductDownload` & `FileHistory` |
| `Configuration` | Reads properties such as max download count and media paths |
| `log` | Log4j logger for tracing & error reporting |
| `productid`, `fileName` | Instance state used during the file download flow |

The code uses straightforward procedural logic; there are no advanced patterns or external frameworks beyond Log4j and Apache Commons Configuration.

---

## 2. Detailed Description

### Flow Overview

| Step | Method | Description |
|------|--------|-------------|
| 1 | `getFileInputStream(HttpServletRequest)` | Parses `fileId` token from the request, validates expiry, loads the corresponding `OrderProductDownload`, updates download counts, fetches the file from disk, and returns a `BufferedInputStream`. |
| 2 | `handleResponse(HttpServletResponse, InputStream)` | Intended to stream the input to the HTTP response; currently empty. |
| 3 | `getFileUrl(int, long)` | Returns an internal URL for the download, delegating to `FileUtil`. |
| 4 | `deleteFile(...)` | Recursively removes a file or directory tree. |
| 5 | `copyFile(...)` | Moves an uploaded temporary file to the merchant’s permanent media directory, optionally cleaning up old files. |

### Initialization / Dependencies

* The class pulls a static `Configuration` instance from `PropertiesUtil` once at class load time.
* Logging is configured via Log4j.
* Service objects (`OrderService`) are fetched from `ServiceFactory` on demand.

### Assumptions & Constraints

| Aspect | Assumption | Impact |
|--------|------------|--------|
| **Thread‑safety** | The service layer handles synchronization. | Not guaranteed – concurrent downloads may corrupt counts. |
| **File path safety** | Merchant IDs and filenames are trusted and come from the database. | If the values can be user‑controlled, path‑traversal is possible. |
| **Disk layout** | Media root and merchant sub‑folders exist and are writable. | Failure to create directories leads to unhandled `FileException`. |
| **Token validity** | The date format used in tokens is always `yyyy-MM-dd`. | Any deviation will cause `ParseException` (unhandled). |

### Architectural Choices

* **Separation of concerns** – business logic (service calls) is decoupled from file I/O via helper classes (`FileUtil`).
* **Configuration‑driven** – paths and limits are externalized.
* **Synchronous API** – no async I/O or streaming frameworks; relies on Java IO streams.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects | Notes |
|--------|---------|------------|---------|--------------|-------|
| `setProductId(long)` | Sets instance product ID. | `productid` | void | Updates field | Simple setter |
| `getFileName()` | Retrieves last requested file name. | none | `String` | none | Used by callers for display |
| `getFileInputStream(HttpServletRequest)` | Main download logic. | `HttpServletRequest` | `InputStream` | Updates download counters, may throw `CoreException`, `OrderException` | **Potential leaks** – caller must close the stream |
| `handleResponse(HttpServletResponse, InputStream)` | Expected to write the stream to response. | `HttpServletResponse`, `InputStream` | void | none | **Not implemented** – placeholder |
| `getFileUrl(int, long)` | Builds internal download URL. | `merchantId`, `downloadId` | `String` | logs errors | Swallows exception and returns empty string |
| `deleteFile(int, File, String)` | Deletes a file under a given folder. | `merchantid`, `File`, `folder` | `boolean` | Recursively deletes files | Always returns `true` – bug |
| `deleteFile(int, File)` | Recursively deletes file or directory tree. | `merchantid`, `File` | `boolean` | Deletes files/directories | Does not report real delete status |
| `copyFile(int, String, File, String, String)` | Moves an uploaded file to merchant’s media folder. | `merchantid`, `config`, `File`, `fileName`, `contentType` | `String` (destination path) | Creates dirs, cleans up, moves file | Deletes temp file twice, error handling minimal |
| `getProductid()` | Getter for product ID. | none | `long` | none | Duplicate of `getProductId` |
| `setProductid(long)` | Another setter for product ID (typo). | `productid` | void | Updates field | Redundant with `setProductId` |
| `setFileName(String)` | Setter for `fileName`. | `fileName` | void | Updates field | Simple setter |

**Reusable/Utility methods**  
None beyond the two `deleteFile` overloads and `copyFile`. All methods perform side‑effects tied to the file system.

---

## 4. Dependencies

| Library | Category | Usage |
|---------|----------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reading configuration properties |
| `org.apache.log4j.Logger` | Logging | Application logging |
| `javax.servlet.http.HttpServletRequest/Response` | Servlet API | HTTP request/response handling |
| `com.salesmanager.core.*` | Internal | Core services, entities, utilities |
| `java.io.*` | JDK | File I/O |
| `java.text.*` | JDK | Date parsing |
| `java.util.*` | JDK | Collections, dates |

All dependencies are either standard Java or third‑party libraries that are widely used. No platform‑specific code is present.

---

## 5. Additional Notes & Recommendations

### 5.1 Security

* **Path traversal** – `merchantId` and `fileName` are concatenated directly into paths. If these values can be influenced by the user, an attacker could read or delete arbitrary files. Validate or sanitize these values (e.g., ensure they are numeric and contain no slashes).
* **Token expiration** – The expiry check uses `date.before(new Date(today.getTime()))`. This will treat any past date as expired, but does not account for timezone or clock skew. Consider using `java.time` APIs and a configurable grace period.

### 5.2 Resource Management

* `getFileInputStream` opens a `FileInputStream` wrapped in a `BufferedInputStream` but never closes it. The caller must remember to close it; otherwise, file handles may leak. Use **try‑with‑resources** or document the contract clearly.
* `handleResponse` is a stub. When implemented, it should also close the input stream in a finally block.

### 5.3 Error Handling

* **Swallowed exceptions** – `getFileUrl` logs the error but returns an empty string, which may cause confusing downstream behaviour. Prefer to throw a checked exception or return `Optional<String>`.
* **Deletion logic** – `deleteFile` always returns `true` regardless of whether `file.delete()` succeeded. The method should return the actual result of the delete operation.
* **Copying** – `copyFile` deletes the source file after `renameTo` succeeds. While `renameTo` moves the file, a double delete is harmless but unnecessary. In failure scenarios, the source file is deleted regardless of the failure, potentially losing data.

### 5.4 Concurrency

* The class is not thread‑safe. Multiple concurrent downloads for the same file may race when updating `downloadCount` and `FileHistory`. Wrap these updates in a database transaction or use atomic database fields.
* Static `conf` is immutable, so it is safe, but the instance fields `productid` and `fileName` could be shared across threads if the same object is reused.

### 5.5 Code Style & Maintenance

* **Duplicate setters** (`setProductId` vs `setProductid`) create confusion. Remove the redundant method.
* **Empty finally blocks** (`deleteFile`, `copyFile`) are misleading. Remove them or add real cleanup.
* **Hard‑coded separators** (`"/"`) should use `File.separator` for cross‑platform compatibility.
* **Magic strings** – e.g., `"core.product.file.downloadmaxcount"` – should be constants or retrieved via a configuration helper.

### 5.6 Potential Enhancements

1. **Streaming API** – Replace blocking IO with NIO or Servlet 3.1 async streaming for better scalability.
2. **Checksum verification** – Validate file integrity during copy.
3. **Audit logging** – Record download events with timestamps and user identities.
4. **Configuration helper** – Abstract `conf.getInt()` / `getString()` into typed methods to reduce boilerplate.
5. **Unit tests** – Add tests for token validation, download count increment, path generation, and cleanup logic.

---

### 5.7 Summary of Issues

| Issue | Severity | Suggested Fix |
|-------|----------|---------------|
| Unimplemented `handleResponse` | Medium | Implement streaming logic or remove method. |
| Potential path traversal | High | Validate `merchantId` & `fileName`. |
| Unclosed InputStream | Medium | Caller must close; document or use try‑with‑resources. |
| Delete always returns true | Medium | Return `file.delete()` result. |
| Double delete in `copyFile` | Low | Remove redundant delete. |
| Duplicate setter methods | Low | Remove `setProductid`. |
| Swallowing exceptions in `getFileUrl` | Medium | Throw or return `Optional`. |
| Race conditions on download counts | High | Use transactional updates or atomic DB fields. |

By addressing the above points, the module would become more robust, secure, and easier to maintain.

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
package com.salesmanager.core.module.impl.application.files;

import java.io.BufferedInputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import com.salesmanager.core.CoreException;
import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.entity.orders.FileHistory;
import com.salesmanager.core.entity.orders.OrderProductDownload;
import com.salesmanager.core.module.model.application.DownloadFileModule;
import com.salesmanager.core.module.model.application.ProductFileModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.order.OrderException;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.util.FileUtil;
import com.salesmanager.core.util.PropertiesUtil;

/**
 * Implementation for managing the creation and copy of files localy on the
 * server
 * 
 * @author Carl Samson
 * 
 */
public class LocalFileImpl extends CoreFileImpl implements ProductFileModule,
		DownloadFileModule {

	private static Configuration conf = PropertiesUtil.getConfiguration();

	private static Logger log = Logger.getLogger(LocalFileImpl.class);

	private long productid;

	private String fileName;

	public void setProductId(long productid) {
		this.productid = productid;
	}

	public String getFileName() {
		return fileName;
	}

	public InputStream getFileInputStream(HttpServletRequest request)
			throws Exception {

		// parse token

		String fileid = request.getParameter("fileId");

		Map fileInfo = FileUtil.getFileDownloadFileTokens(fileid);

		String fileId = (String) fileInfo.get("ID");
		String date = (String) fileInfo.get("DATE");
		String merchantId = (String) fileInfo.get("MERCHANTID");

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
			// String lbl =
			// LabelUtil.getInstance().getText("message.error.download.delayexpired1");

		}

		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);

		OrderProductDownload download = oservice
				.getOrderProductDownload(new Long(fileId));

		this.setFileName(download.getOrderProductFilename());

		int maxcount = conf.getInt("core.product.file.downloadmaxcount", 5);

		// String filename = "";

		FileHistory fh = oservice.getFileHistory(Integer.parseInt(merchantId),
				download.getFileId());

		if (fh == null) {
			log
					.warn("Trying to update non existing file history [attrid "
							+ download.getFileId() + "][merchantid "
							+ merchantId + "]");
		} else {
			int downloadcount = fh.getDownloadCount();
			int newcount = downloadcount + 1;
			fh.setDownloadCount(newcount);
			oservice.saveOrUpdateFileHistory(fh);
		}

		int downloadcount = download.getDownloadCount();
		if (downloadcount == maxcount) {
			OrderException oe = new OrderException("Maximum download reached",
					ErrorConstants.MAXIMUM_ORDER_PRODUCT_DOWNLOAD_REACHED);
			throw oe;

		}

		int newcount = downloadcount + 1;
		download.setDownloadCount(newcount);
		oservice.saveOrUpdateOrderProductDownload(download);

		//String downloadPath = conf.getString("core.product.file.filefolder");
		String downloadPath = FileUtil.getDownloadFilePath();

		File file = new File(downloadPath + "/" + merchantId + "/"
				+ download.getOrderProductFilename());

		BufferedInputStream bis = new BufferedInputStream(new FileInputStream(
				file));

		return bis;
	}

	public void handleResponse(HttpServletResponse response, InputStream fis)
			throws IOException {

	}

	public String getFileUrl(int merchantId, long downloadId)
			throws FileException {

		try {
			return FileUtil.getInternalDownloadFileUrl(merchantId, downloadId);
		} catch (Exception e) {
			log.error(e);
		}
		return "";

	}

	public boolean deleteFile(int merchantid, File file, String folder) {

		if (folder != null) {
			String newFile = folder + "/" + file.getName();
			file = new File(newFile);
		}
		return deleteFile(merchantid, file);
	}

	public boolean deleteFile(int merchantid, File file) {

		try {

			boolean direxist = file.exists();

			if (direxist) {
				if (file.isDirectory()) {
					String[] children = file.list();
					for (int i = 0; i < children.length; i++) {
						boolean success = deleteFile(merchantid, new File(file,
								children[i]));
						if (!success) {
							return false;
						}
					}
				}
			}

			// The directory is now empty so delete it
			boolean delete = true;
			try {
				file.delete();
			} catch (Exception e) {
				log.warn("File " + file.getName() + " does not exist");
			}
			return delete;

		} finally {

		}
	}

	public String copyFile(int merchantid, String config, File file,
			String fileName, String contentType) throws FileException {

		try {

			String filefolder = conf.getString(config + ".filefolder");
			if (filefolder == null) {
				throw new FileException(FileException.ERROR, "Properties "
						+ config + ".filefolder not defined");
			}

			// Check if merchant directory exist
			
			StringBuffer dirPath = new StringBuffer();
			dirPath.append(FileUtil.getMediaPath());
			dirPath.append(filefolder);
			dirPath.append("/");
			dirPath.append(String.valueOf(merchantid));
			String directory = dirPath.toString();
			
			//String directory = filefolder + "/" + String.valueOf(merchantid);
			String dir = conf.getString(config + ".dirname");

			String destinationdir = directory;

			if (dir != null) {
				destinationdir = directory + "/" + dir;
			}

			// do we need to clear
			String clear = conf.getString(config + ".cleanup", "false");

			if (clear.equals("true")) {
				// will delete the directory if it exist
				this.deleteFile(merchantid, new File(destinationdir));
			}

			// check if directory/merchantid exists
			boolean exists = (new File(directory)).exists();
			if (!exists) {
				// create the directory merchant id
				File merchantdir = new File(directory);
				boolean successcreate = merchantdir.mkdir();
				if (!successcreate) {
					throw new FileException(FileException.ERROR,
							"Can't create directory " + merchantdir);
				}
			}

			// create the destination directory
			File destfile = new File(destinationdir);
			boolean exists2 = (new File(destinationdir)).exists();
			if (!exists2) {
				boolean successcreatefinal = destfile.mkdir();
				if (!successcreatefinal) {
					throw new FileException(FileException.ERROR,
							"Can't create directory " + destfile);
				}
			}

			// check if file exist in destination
			File destfile2 = new File(destfile, fileName);
			boolean destfile2exist = destfile2.exists();
			if (destfile2exist) {
				// remove it
				destfile2.delete();
			}

			// Move file to new directory
			// boolean successcopy = file.renameTo(new File(destfile,
			// file.getName()));
			boolean successcopy = file.renameTo(new File(destfile, fileName));
			if (!successcopy) {
				file.delete();
				throw new FileException(FileException.ERROR, "Can't move file "
						+ file.getName() + " to destfile");
			}

			// delete temp file
			file.delete();
			return destinationdir + "/" + fileName;

		} finally {

		}
	}

	public long getProductid() {
		return productid;
	}

	public void setProductid(long productid) {
		this.productid = productid;
	}

	public void setFileName(String fileName) {
		this.fileName = fileName;
	}

}



```
