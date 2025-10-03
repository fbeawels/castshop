# PropertiesUtil.java

## Review

## 1. Summary  
`PropertiesUtil` is a tiny, static helper that centralises loading of application configuration files for the **Sales Manager** core module. It uses Apache Commons Configuration to read key/value pairs from:

1. `sm-core-config.properties` – the main configuration file.  
2. `struts.properties` – a Struts‑2 configuration file whose `struts.custom.i18n.resources` list is merged into the main config.

The class exposes the merged `Configuration` instance via `getConfiguration()`. Log4j is used for diagnostics.

> **Key points**  
> * Static initialisation guarantees a single, thread‑safe instance.  
> * The runtime directory is optionally specified with the `-DsmRuntimeDirectory` system property.  
> * If the file is missing, a fallback to the class‑path version is attempted.  
> * The code relies on Apache Commons Configuration (`Configuration`, `FileConfiguration`, `PropertiesConfiguration`) and Log4j.

---

## 2. Detailed Description  

### 2.1 Core flow  
| Step | What happens | Why |
|------|--------------|-----|
| **Static block executes** | The first time `PropertiesUtil` is referenced | Sets up the shared `config` object. |
| **Determine runtime directory** | Reads `smRuntimeDirectory` system property | Allows the application to point at a custom config location. |
| **Load main properties** | If the property is present and the file exists → `config.setFile(file)` & `config.load()`. Otherwise, fall back to `sm-core-config.properties` on the class‑path. | Guarantees at least one set of properties is loaded. |
| **Set `smRuntimeDirectory` in config** | After loading from a custom directory, the property is stored back into the `config` object. | Makes the directory value available to other parts of the application. |
| **Load bundle (Struts) properties** | Reads `struts.properties` from the class‑path, extracts `struts.custom.i18n.resources`, and merges it into the main config. | Allows Struts i18n resources to be used globally. |
| **Error handling** | Any exception during the load process is logged via Log4j. | Prevents the application from crashing on mis‑configured files. |

### 2.2 Assumptions & Constraints  
* The application expects the main properties file to be named **`sm-core-config.properties`**.  
* If the `smRuntimeDirectory` system property is supplied, the file must exist there; otherwise the class‑path version is used.  
* The Struts properties file must be present on the class‑path.  
* The class does **not** provide any API for reloading or updating the configuration at runtime.  
* `config` is mutable – callers can modify it via the returned `Configuration` instance.  

### 2.3 Architecture & Design Choices  
* **Static Singleton** – The use of a static block is a simple, thread‑safe way to guarantee a single configuration instance without an explicit `Singleton` pattern.  
* **Apache Commons Configuration** – Provides a flexible API for loading `.properties` files, merging lists, and later reading typed values.  
* **Loose coupling** – The class itself is isolated; all business logic is elsewhere.  
* **Minimal dependency footprint** – Only two third‑party libraries (`commons-configuration` and `log4j`).  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `private PropertiesUtil()` | Private constructor to prevent instantiation | – | – | – |
| `public static Configuration getConfiguration()` | Provides global access to the loaded configuration | – | `Configuration` (the static `config` instance) | No side‑effects; callers may mutate the returned object. |

*All other logic lives inside the static initializer block.*

---

## 4. Dependencies  

| Library | Type | Notes |
|---------|------|-------|
| `org.apache.commons.configuration` (`Configuration`, `FileConfiguration`, `PropertiesConfiguration`) | Third‑party | Handles property file parsing, merging, and list handling. |
| `org.apache.log4j.Logger` | Third‑party | Used for logging. No SLF4J abstraction. |
| JDK (java.io.File, java.util.List) | Standard | Basic I/O and collections. |

*Platform‑specific assumptions:* The code expects a Unix‑style file separator (`/`) when constructing the configuration file path. On Windows, `File` will translate to backslashes, but the concatenation uses `/`. This works on Windows too because `File` normalises, but could be clarified.

---

## 5. Additional Notes  

### 5.1 Strengths  
* **Simplicity** – Easy to understand and maintain.  
* **Fallback logic** – Gracefully degrades to class‑path config if custom file is missing.  
* **Merge logic** – Automatically incorporates Struts i18n resources into the main config.  

### 5.2 Weaknesses & Edge Cases  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw `List` usage** | Unchecked type safety; compiler warnings. | Use generics: `List<String> lst = bundleConfigs.getList("...");` |
| **No reload capability** | Cannot refresh configuration at runtime. | Expose a `reload()` method or use `FileConfiguration`'s reload feature. |
| **Mutability leakage** | Callers can alter the shared `config`, potentially breaking assumptions. | Return an immutable view (`new ImmutableConfiguration(config)`) or document immutability. |
| **Hard‑coded file names** | Not flexible for other environments. | Allow injection of file names or patterns via system properties. |
| **Error handling is minimal** | All exceptions are logged but not propagated, potentially hiding startup failures. | Throw a runtime exception after logging to fail fast. |
| **Platform‑dependent path construction** | Using `/` may be confusing on Windows. | Use `File.separator` or `new File(runtimedirectory, "sm-core-config.properties")`. |
| **Potential double loading of default config** | When `runtimedirectory` is null, `config` is created with `new PropertiesConfiguration()` then immediately re‑created with `new PropertiesConfiguration("sm-core-config.properties")`. | Skip the first empty instance; just instantiate with the file path. |
| **`smRuntimeDirectory` not set when using fallback** | The property is absent in the config when the default file is used. | Always set the property, even when loading from class‑path. |
| **Struts file missing** | `bundleConfigs.load()` throws an exception, breaking init. | Check file existence before loading or catch `ConfigurationException`. |

### 5.3 Potential Enhancements  

1. **Configuration Reload** – Add a `public static void reload()` method that re‑initialises `config` from the same sources.  
2. **Immutability** – Wrap the returned `Configuration` in an immutable decorator to prevent accidental modification.  
3. **Extensibility** – Accept a list of property file names or a configuration directory via system properties, making the utility more generic.  
4. **Logging Framework** – Replace Log4j with SLF4J to decouple from a specific logging implementation.  
5. **Unit Tests** – Add tests that simulate different `smRuntimeDirectory` scenarios, missing files, and malformed properties.  

---

### 5.4 Verdict  
`PropertiesUtil` is a functional, low‑overhead helper that serves its purpose within the Sales Manager core. While it is adequate for simple use cases, the code could benefit from modern Java practices (generics, immutability, better error handling) and additional flexibility (reload, custom file paths). These improvements would make the utility more robust, easier to maintain, and safer for use in a larger, multi‑module application.

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

import java.io.File;
import java.util.List;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.configuration.FileConfiguration;
import org.apache.commons.configuration.PropertiesConfiguration;
import org.apache.log4j.Logger;

/**
 * Manage Apache Configuration file type to be used
 * 
 * @author Carl Samson
 * 
 */
public class PropertiesUtil {

	// private static Configuration config = null;
	private static FileConfiguration config = null;
	private static Logger log = Logger.getLogger(PropertiesUtil.class);

	static {

		try {

			/**
			 * To externalize properties create a -D system property specifying
			 * the directory to be scanned for retrieving
			 * sm-core-config.properties
			 */

			config = new PropertiesConfiguration();

			String runtimedirectory = System.getProperty("smRuntimeDirectory");
			if (runtimedirectory == null) {
				log
						.warn("smRuntimeDirectory not specified, will get sm-core-config.properties from classpath");
				config = new PropertiesConfiguration(
						"sm-core-config.properties");// hope it is in the
														// classpath
				config.load();
			} else {
				log.info("Loading properties from " + runtimedirectory
						+ "/sm-core-config.properties");
				String configurationfile = runtimedirectory
						+ "/sm-core-config.properties";
				File file = new File(configurationfile);
				if (file.exists()) {
					config.setFile(file);
					config.load();
					config.setProperty("smRuntimeDirectory", runtimedirectory);
				} else {
					log.error(configurationfile + " does not exist");
					config = new PropertiesConfiguration(
							"sm-core-config.properties");// hope it is in the
															// classpath
					config.load();
				}
			}

			// load bundles
			FileConfiguration bundleConfigs = new PropertiesConfiguration(
					"struts.properties");
			bundleConfigs.load();

			List lst = bundleConfigs.getList("struts.custom.i18n.resources");
			if (lst != null) {
				config.addProperty("struts.custom.i18n.resources", lst);
			}

		} catch (Exception e) {
			log.error(e);
		}
	}

	private PropertiesUtil() {

	}

	public static Configuration getConfiguration() {

		return config;
	}

}



```
