# CoreFileImpl.java

## Review

## 1. Summary

`CoreFileImpl` is an **abstract** file‑upload helper that implements the `FileModule` interface.  
Its main job is to:

1. **Validate** the uploaded file against a set of rules defined in a configuration file (allowed MIME types, maximum width/height for images, file size limits).
2. **Delegate** the actual persistence of the file to an abstract `copyFile` method (implemented by concrete subclasses).

Key components & design choices  

- Uses Apache Commons `Configuration` for reading application‑wide properties.  
- Employs `org.apache.commons.lang.xwork.StringUtils` for blank checks (a legacy class).  
- Relies on `javax.imageio.ImageIO` and `java.awt.image.BufferedImage` for basic image dimension checks.  
- The class is **stateless** aside from the singleton `Configuration` and a static logger, making it thread‑safe in its current form.

Notable patterns: *Template Method* – the `uploadFile` method performs common validation logic, while subclasses supply the concrete `copyFile` implementation.

---

## 2. Detailed Description

### Flow of execution

| Stage | Description |
|-------|-------------|
| **Validation** | `uploadFile` first checks the MIME type, image dimensions (if configured), and file size against limits pulled from `PropertiesUtil.getConfiguration()`. |
| **Error handling** | If any check fails, a `FileException` (custom runtime exception) is thrown with an internationalized message retrieved via `LabelUtil`. |
| **Delegation** | After successful validation, the method calls the abstract `copyFile`, handing off the file for persistence. |
| **Return** | The concrete subclass’s `copyFile` returns a `String` (presumably a URL or path) which `uploadFile` propagates to the caller. |

### Assumptions & constraints

- **Configuration presence** – The class assumes the configuration keys (`.contenttypes`, `.maxwidth`, `.maxheight`, `.maxfilesize`) exist. Missing keys cause explicit exceptions.
- **Image files only checked for dimension** – MIME type checks are generic, but dimension validation is only applied when width/height limits are present. Non‑image files with dimensions configured will still be validated (they will simply fail on `ImageIO.read`).
- **Thread safety** – The class uses only immutable or thread‑safe members (`Configuration`, `Logger`), so concurrent uploads are safe.
- **File system handling** – The actual copy/persistence is deferred to subclasses; this class doesn’t interact with the file system beyond reading the image for dimension checks.

### Architecture

The code follows a **thin service** architecture: it acts as a validator and orchestrator, while concrete storage (local disk, S3, DB, etc.) is encapsulated in subclasses. This keeps validation logic in one place and allows for different storage strategies without duplicating checks.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `uploadFile(int merchantid, String config, File file, String fileName, String contentType)` | Validates an uploaded file and delegates persistence. | *merchantid* – tenant/merchant identifier.<br>*config* – base key for configuration properties.<br>*file* – uploaded file object.<br>*fileName* – original file name.<br>*contentType* – MIME type of the file. | `String` – the value returned by `copyFile` (likely a URL or storage key). | Throws `FileException` on validation failures. |
| `copyFile(...)` | **Abstract**. Implemented by subclasses to move/store the file. | Same as `uploadFile`. | `String` – result of the copy operation. | – |

*Reusable / Utility methods* – None; all logic lives inside `uploadFile`. Utility responsibilities (e.g., reading config, string checks) are handled via external libraries.

---

## 4. Dependencies

| Library / API | Type | Notes |
|---------------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads application properties. |
| `org.apache.commons.lang.xwork.StringUtils` | Third‑party (legacy) | Provides `isBlank`; modern code would use `org.apache.commons.lang3.StringUtils` or `String.isBlank()`. |
| `org.apache.log4j.Logger` | Third‑party | Classic Log4J; could be replaced with SLF4J+Logback for better abstraction. |
| `javax.imageio.ImageIO`, `java.awt.image.BufferedImage` | Standard Java | For image dimension checks. |
| `com.salesmanager.core.util.PropertiesUtil`, `LabelUtil` | Project‑internal | Utility helpers for configuration and i18n. |
| `com.salesmanager.core.module.model.application.FileModule` | Project‑internal | Interface declaring `copyFile`. |
| `com.salesmanager.core.module.impl.application.files.FileException` | Project‑internal | Custom runtime exception. |

No platform‑specific dependencies; the code is JVM‑agnostic.

---

## 5. Additional Notes & Recommendations

### 1. Use of raw types  
```java
List ct = new ArrayList();
```
*Issue*: Raw type usage defeats generics, potentially leading to `ClassCastException` later.  
*Fix*: `List<String> ct = new ArrayList<>();`

### 2. Legacy `StringUtils`  
`org.apache.commons.lang.xwork.StringUtils` is outdated. Switching to `org.apache.commons.lang3.StringUtils` or the standard `String.isBlank()` would reduce dependencies.

### 3. Hard‑coded property fallback  
The fallback to `core.branding.cart.maxfilesize` is a magic string. It should be documented or extracted as a constant.

### 4. Image dimension check for non‑image files  
`ImageIO.read(file)` returns `null` for unsupported formats. The code does not handle `null`, which would cause a `NullPointerException`. Wrap this in a null check.

```java
originalImage = ImageIO.read(file);
if(originalImage == null) {
    throw new FileException(LabelUtil.getInstance().getText(
        "errors.unsupportedimagefile"));
}
```

### 5. Exception handling consistency  
The method declares `throws FileException` but also catches generic `Exception` and re‑wraps it. Consider defining a dedicated `ConfigurationException` for property parsing errors.

### 6. Logging  
Currently, the class logs nothing. Adding trace or debug logs around key validation steps can help diagnose issues in production.

### 7. Resource leaks  
`ImageIO.read(file)` opens an `InputStream`. Although the file is not closed explicitly, the underlying stream is closed by ImageIO internally. In Java 7+, consider using try‑with‑resources if you open streams manually.

### 8. Thread‑local configuration cache  
`PropertiesUtil.getConfiguration()` may perform IO each call; if heavy, cache the configuration statically or rely on the existing library's caching.

### 9. Return type of `copyFile`  
The method signature returns `String`. It is unclear whether this is a path, URL, or identifier. Document this contract.

### 10. Future enhancements  
- **Content‑length header validation**: For large files, you might want to stream rather than load into memory.  
- **Asynchronous upload**: Delegate `copyFile` to an async executor for non‑blocking uploads.  
- **Metadata extraction**: Extract EXIF data or file hash for duplicate detection.  
- **Unit tests**: Mock configuration and subclass `copyFile` to test validation logic.  
- **Internationalization**: Centralize error keys rather than embedding them in code.

---

### Summary of Issues & Suggested Fixes

| Issue | Severity | Suggested Fix |
|-------|----------|---------------|
| Raw types in list | Low | Use generics (`List<String>`) |
| Legacy StringUtils | Low | Replace with Lang3 or standard `String.isBlank()` |
| Null image on `ImageIO.read` | Medium | Check for null and throw informative exception |
| Hard‑coded fallback key | Low | Define constant or document rationale |
| No logging | Low | Add debug statements for validation steps |
| Exception handling | Medium | Separate config parsing errors; avoid catching `Exception` broadly |
| Resource leaks | Low | Ensure streams are closed (try‑with‑resources if applicable) |
| Unclear `copyFile` contract | Low | Document expected return value |

With these adjustments, the code will be more robust, easier to maintain, and better aligned with modern Java practices.

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

import java.awt.image.BufferedImage;
import java.io.File;
import java.util.ArrayList;
import java.util.List;
import java.util.StringTokenizer;

import javax.imageio.ImageIO;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.xwork.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.module.model.application.FileModule;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.PropertiesUtil;

public abstract class CoreFileImpl implements FileModule {

	private Configuration conf = PropertiesUtil.getConfiguration();
	private static Logger log = Logger.getLogger(CoreFileImpl.class);

	public String uploadFile(int merchantid, String config, File file,
			String fileName, String contentType) throws FileException {

		/** Check content type **/
		String imgct = conf.getString(config + ".contenttypes");
		if (imgct != null) {

			List ct = new ArrayList();
			StringTokenizer st = new StringTokenizer(imgct, ";");
			while (st.hasMoreTokens()) {
				ct.add(st.nextToken());
			}

			// check content type
			if (!ct.contains(contentType)) {
				throw new FileException(LabelUtil.getInstance().getText(
						"errors.unsupported.file ")
						+ contentType);
			}
		}

		/** if an image check size **/
		String imgwsz = conf.getString(config + ".maxwidth");
		String imghsz = conf.getString(config + ".maxheight");
		
		
		
		if (imgwsz != null && imghsz != null) {

			int wseize = 0;
			int hseize = 0;

			BufferedImage originalImage = null;

			try {
				wseize = Integer.parseInt(imgwsz);
				hseize = Integer.parseInt(imghsz);

			} catch (Exception e) {
				throw new FileException(e);
			}

			try {

				originalImage = ImageIO.read(file);
				int width = originalImage.getWidth();
				int height = originalImage.getHeight();

				if (width > wseize || height > hseize) {
					throw new FileException(LabelUtil.getInstance().getText(
							"errors.filedimensiontoolarge"));
				}

			} catch (FileException fe) {
				throw fe;
			} catch (Exception e) {
				throw new FileException(e);
			}

		}

		// Check file size
		long fsize = file.length();
		String smaxfsize = conf.getString(config + ".maxfilesize");
		if(StringUtils.isBlank(smaxfsize)) {
			smaxfsize = conf.getString("core.branding.cart.maxfilesize");
		}
		if (smaxfsize == null) {
			throw new FileException(FileException.ERROR, "Properties " + config
					+ ".maxfilesize not defined");
		}
		long maxsize = 0;
		try {
			maxsize = Long.parseLong(smaxfsize);

		} catch (Exception e) {
			throw new FileException(e);
		}

		if (fsize > maxsize) {
			throw new FileException(LabelUtil.getInstance().getText(
					"errors.filetoolarge"));
		}

		return copyFile(merchantid, config, file, fileName, contentType);

	}

}



```
