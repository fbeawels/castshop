# TemplateResourceBundleLoader.java

## Review

## 1. Summary  
**Purpose**  
`TemplateResourceBundleLoader` is a servlet that pre‑loads message resource bundles for all “store‑front template” modules when the web application starts. By calling `LocalizedTextUtil.addDefaultResourceBundle()` for each module it ensures that the XWork2 `LocalizedTextUtil` has all locale‑specific properties files available for later use (e.g., in Struts or XWork actions).  

**Key Components**  
| Class | Role |
|-------|------|
| `TemplateResourceBundleLoader` | Servlet that performs the load during `init()` |
| `ReferenceService` | Service that retrieves `CoreModuleService` objects from the data store |
| `CoreModuleService` | Entity describing a module; its name is used to build the bundle key |
| `LocalizedTextUtil` | XWork2 helper that registers resource bundles by name |
| `CatalogConstants.STORE_FRONT_TEMPLATES_CODE` | Constant that identifies the module category |
| `Constants.ALLCOUNTRY_ISOCODE` | Country code used to fetch all modules (regardless of locale) |

**Design patterns / frameworks**  
* The servlet uses the **Singleton Service Factory** pattern (`ServiceFactory.getService(...)`) to obtain services.  
* It relies on **Struts/XWork** (`LocalizedTextUtil`) for internationalization.  
* Logging is handled via **log4j**.

---

## 2. Detailed Description  
1. **Servlet Lifecycle**  
   * The servlet extends `HttpServlet` and implements `Servlet`.  
   * `init(ServletConfig)` forwards to the no‑arg `init()` after calling `super.init()`.  
   * The no‑arg `init()` is overridden to perform the bundle loading logic.  
2. **Bundle Loading Logic**  
   * A `ReferenceService` instance is fetched from `ServiceFactory`.  
   * `getCoreModules()` is called with the store‑front templates code and the special country code `ALLCOUNTRY_ISOCODE`, returning a `Collection` of `CoreModuleService` objects.  
   * For each module, the bundle key is constructed as `catalog-<moduleName>`.  
   * `LocalizedTextUtil.addDefaultResourceBundle(fileName)` registers the bundle.  
3. **Error Handling**  
   * All operations are wrapped in a broad `try/catch (Exception)`.  Errors are logged at `error` level, but the servlet never throws a `ServletException`, so a failure to load bundles will not abort the application startup.  
4. **Cleanup**  
   * No explicit cleanup; the servlet relies on container shutdown for resource release.  

**Assumptions & Constraints**  
* The service factory and all services are expected to be correctly configured and available during servlet initialization.  
* Module names do not contain characters that would break resource bundle naming conventions.  
* The `LocalizedTextUtil` registry is static; re‑loading the same bundle will overwrite the previous entry.  
* The servlet does not support dynamic module registration after startup.  

---

## 3. Functions/Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|------------|---------|--------|---------|--------------|
| `public void init(ServletConfig config)` | overrides `HttpServlet.init` | Passes the config to the superclass and then delegates to the no‑arg `init()`. | `ServletConfig config` | None | Initializes the servlet; prepares it for use. |
| `public void init()` | no‑arg | Main entry point that loads all template bundles. | None | None | Registers each module’s resource bundle with `LocalizedTextUtil`; logs progress or errors. |
| `private static Logger log` | field | Log messages using log4j. | None | None | Logging only. |

*No public API beyond the servlet lifecycle methods.*  

---

## 4. Dependencies  

| Library / Framework | Type | Notes |
|---------------------|------|-------|
| `javax.servlet` | Standard API | Servlet container requirement. |
| `org.apache.log4j.Logger` | Third‑party | Classic logging framework. |
| `com.opensymphony.xwork2.util.LocalizedTextUtil` | Third‑party | XWork2 (Struts2) text localization helper. |
| `com.salesmanager.core` packages (`ReferenceService`, `CoreModuleService`, `ServiceFactory`, `CatalogConstants`, `Constants`) | Internal | Business‑logic services of the SalesManager application. |
| `java.util.Collection`, `Iterator` | Standard | Basic Java collections. |

All dependencies are typical for a Java EE web application using Struts2 / XWork.

---

## 5. Additional Notes  

### Strengths  
* **Clear separation of concerns** – the servlet only handles initialization; business logic remains in services.  
* **Simple logging** – progress and error information is available in the server logs.  
* **Reusable pattern** – other modules could adopt the same approach to load their own bundles.  

### Weaknesses & Edge Cases  
1. **Exception swallowing** – any exception during bundle loading is logged but ignored, potentially masking configuration errors.  
2. **Hard‑coded bundle naming convention** – if module names contain dots or other illegal characters, the generated bundle key may be invalid.  
3. **No cache validation** – if module bundles change while the application is running, the servlet will not reload them.  
4. **Thread‑safety** – while the servlet is only initialized once, the use of a static `LocalizedTextUtil` registry must be thread‑safe (it is in XWork, but still worth noting).  
5. **No graceful degradation** – the servlet continues to run even if no bundles are found; this may lead to missing translations at runtime.  

### Potential Enhancements  
* **Throw a `ServletException`** when a critical error occurs (e.g., no modules found), so that the container can decide to stop the application.  
* **Validate bundle names** before registration.  
* **Support dynamic reloading** by exposing a `reload()` method or a scheduled job that checks for changes.  
* **Add unit tests** that mock the `ReferenceService` to verify that `addDefaultResourceBundle` is called with the correct keys.  
* **Externalize the module category and country code** as servlet init‑params, making the servlet reusable for other module types.  

Overall, the servlet fulfills its intended purpose with minimal code, but could benefit from stronger error handling and a more flexible configuration approach.

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
package com.salesmanager.common;

import java.util.Collection;
import java.util.Iterator;

import javax.servlet.ServletConfig;
import javax.servlet.ServletException;

import org.apache.log4j.Logger;

import com.opensymphony.xwork2.util.LocalizedTextUtil;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.reference.ReferenceService;

public class TemplateResourceBundleLoader extends
		javax.servlet.http.HttpServlet implements javax.servlet.Servlet {

	private static Logger log = Logger
			.getLogger(TemplateResourceBundleLoader.class);

	public void init(ServletConfig config) throws ServletException {
		super.init(config);
		this.init();
	}

	public void init() throws ServletException {

		try {

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			Collection coll = rservice.getCoreModules(
					CatalogConstants.STORE_FRONT_TEMPLATES_CODE,
					Constants.ALLCOUNTRY_ISOCODE);

			if (coll == null || coll.size() == 0) {
				log.warn("No template bundle found");
			} else {
				Iterator i = coll.iterator();
				while (i.hasNext()) {
					CoreModuleService cms = (CoreModuleService) i.next();
					String fileName = "catalog-" + cms.getCoreModuleName();
					log.info("Loading messages from " + fileName);
					LocalizedTextUtil.addDefaultResourceBundle(
							fileName);

				}
			}

		} catch (Exception e) {
			log.error(e);
		}

	}

}



```
