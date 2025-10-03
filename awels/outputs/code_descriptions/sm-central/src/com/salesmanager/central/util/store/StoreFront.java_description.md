# StoreFront.java

## Review

## 1. Summary
**Purpose**  
`StoreFront` is a helper/utility class that allows a merchant to switch the visual template (module) of their storefront and then retrieve a detailed description of that module’s configuration.

**Key Components**
- **`selectTemplate(String moduleName, String countryCode)`** – the sole public method that:
  1. Reads the current user session and merchant context.
  2. Persists the chosen template for the merchant.
  3. Loads the module’s configuration (e.g., image sizes, banner width).
  4. Builds a human‑readable description string and attaches it to a `CoreModuleService` object.
  5. Returns the populated `CoreModuleService` (or `null` on error).

**Design Patterns / Libraries**
- **Service Factory** – a simple factory pattern (`ServiceFactory.getService(...)`) for obtaining service instances.
- **Web Context Factory (DWR)** – `WebContextFactory.get().getHttpServletRequest()` to access the servlet request in a non‑controller class.
- **Dependency Injection via Session** – `Context` is retrieved from the HTTP session; this is a lightweight approach but couples the code tightly to the session store.

---

## 2. Detailed Description

### Execution Flow
1. **Session Retrieval**  
   The method obtains the `HttpServletRequest` via DWR’s `WebContextFactory`. It then pulls the `Context` bean from the session (`ProfileConstants.context`).

2. **Merchant Store Update**  
   Using `MerchantService`, the merchant’s `MerchantStore` is fetched, its template module is set to the requested `moduleName`, and the store is persisted.

3. **Module Configuration Fetch**  
   - `ReferenceService` supplies the core module configuration (`CoreModuleService`) for the supplied `countryCode` and `moduleName`.
   - It also retrieves a collection of `ModuleConfiguration` objects for the selected module.

4. **Description Building**  
   A `StringBuilder` is used to concatenate various configuration properties (image width/height, banner width, slider image size) with translated labels obtained via `LabelUtil`. The final string is set into the `CoreModuleService` description field.

5. **Return Value**  
   The fully populated `CoreModuleService` is returned. If any exception occurs, it is logged, and `null` is returned.

### Assumptions & Constraints
- The method assumes that the HTTP session always contains a `Context` instance.
- It also assumes that the merchant store update will succeed; errors are swallowed and the method returns `null`.
- No validation of `moduleName` or `countryCode` is performed.
- The code relies on the DWR `WebContextFactory` to provide the request – this ties the utility to the DWR framework.
- Logging is done with `org.apache.log4j.Logger`.

### Architecture & Design Choices
- **Centralised Service Retrieval** – The use of a `ServiceFactory` centralises service creation but still relies on static look‑ups, which can hinder testing and flexibility.
- **Hard‑coded String Literals** – The label keys and configuration keys are hard‑coded, making future changes brittle.
- **StringBuilder for HTML** – The description contains raw HTML (`</br>`) which mixes presentation concerns into a service layer.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `public CoreModuleService selectTemplate(String moduleName, String countryCode)` | Persists a merchant’s template choice, fetches configuration, builds a descriptive string, and returns the module service. | `moduleName` – name of the template module.<br>`countryCode` – ISO country code for locale‑specific configuration. | `CoreModuleService` populated with module description (or `null` if an error occurs). | 1. Updates `MerchantStore` in the database.<br>2. Writes to the Log4j log on exception.<br>3. Mutates the returned `CoreModuleService` object. |

### Helper Utilities
- `LabelUtil.getInstance().getText(lang, key)` – retrieves internationalised labels.
- `ServiceFactory.getService(...)` – static factory for obtaining service instances.

---

## 4. Dependencies

| Library / Framework | Type | Notes |
|---------------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard Java EE | Core servlet API. |
| `org.apache.log4j.Logger` | Third‑party | Legacy logging framework. |
| `uk.ltd.getahead.dwr.WebContextFactory` | Third‑party | DWR (Direct Web Remoting) framework. |
| `com.salesmanager.core.*` | Third‑party | Domain‑specific services, entities, and constants. |
| `com.salesmanager.central.*` | In‑house | Central context and constants. |
| `java.util.*` | Standard Java | Collections utilities. |

**Platform Specifics**
- Requires a servlet container (e.g., Tomcat, Jetty) to provide the HTTP request and session.
- Depends on the DWR framework being correctly initialised in the web application.

---

## 5. Additional Notes & Recommendations

### Edge Cases & Potential Issues
- **Null Session / Context** – If the session does not contain `ProfileConstants.context`, a `NullPointerException` will be thrown. Consider graceful handling.
- **Transaction Management** – Updating the `MerchantStore` and then fetching configurations are separate operations. In a failure scenario, the store may be updated but the method returns `null`. Wrap operations in a transactional boundary if supported by the framework.
- **Error Propagation** – Swallowing the exception and returning `null` can make debugging difficult. Returning a detailed error object or throwing a checked exception would be clearer.
- **Hard‑coded HTML** – The description string contains raw `<br>` tags; if the output is consumed by a non‑HTML context, it may cause display issues.

### Suggested Enhancements
1. **Parameter Validation** – Validate `moduleName` and `countryCode` against known values to avoid illegal configuration lookups.
2. **Logging Details** – Log the chosen module and merchant ID to aid auditability.
3. **Return Type Refactor** – Instead of returning `null` on error, return an `Optional<CoreModuleService>` or a custom result wrapper that includes status and error messages.
4. **Internationalisation** – Externalise configuration key constants and label keys into a properties file or enum for easier maintenance.
5. **Unit Testability** – Abstract away the `ServiceFactory` and `WebContextFactory` via interfaces or dependency injection, allowing mocking in tests.
6. **Use of Modern Logging** – Consider SLF4J with a modern logging implementation to decouple the logging API from Log4j.
7. **Remove DWR Dependency** – If possible, obtain the request via a standard servlet filter or pass it explicitly, reducing framework coupling.
8. **HTML Escaping** – If the description will be displayed in a UI, escape or sanitize any user‑supplied values to avoid XSS.

### Code‑Specific Refactor Snippet
```java
private CoreModuleService buildServiceDescription(Context ctx, Collection<ModuleConfiguration> configs) {
    StringBuilder sb = new StringBuilder();
    sb.append(LabelUtil.getInstance().getText(ctx.getLang(), "module.storefront.modulename"))
      .append(' ').append(ctx.getTemplateModule()).append("<br/>");

    for (ModuleConfiguration conf : configs) {
        String key = conf.getId().getConfigurationKey();
        String value = conf.getConfigurationValue();
        String labelKey = switch (key) {
            case ConfigurationConstants.LARGEIMAGEWIDTH_CONFIGURATION_KEY  -> "module.storefront.minimumimagewidth";
            case ConfigurationConstants.LARGEIMAGEHEIGHT_CONFIGURATION_KEY -> "module.storefront.minimumimageheight";
            case ConfigurationConstants.BANNER_CONFIGURATION_KEY          -> "module.storefront.bannerwidth";
            case ConfigurationConstants.SLIDER_CONFIGURATION_KEY          -> "module.storefront.sliderimagesize";
            default -> null;
        };
        if (labelKey != null) {
            sb.append(LabelUtil.getInstance().getText(ctx.getLang(), labelKey))
              .append(' ').append(value).append(" px<br/>");
        }
    }
    return sb.toString();
}
```
Using a switch expression improves readability and reduces repetitive code.

---

**Overall**, `StoreFront` is a compact utility that ties together session state, persistence, and configuration lookup to produce a user‑friendly module description. While functional, the implementation could benefit from better error handling, decoupling from framework specifics, and a cleaner separation between business logic and presentation formatting.

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
package com.salesmanager.central.util.store;

import java.util.ArrayList;
import java.util.Collection;
import java.util.List;

import javax.servlet.http.HttpServletRequest;

import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.ModuleConfiguration;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.LabelUtil;

public class StoreFront {

	private Logger log = Logger.getLogger(StoreFront.class);

	public CoreModuleService selectTemplate(String moduleName,
			String countryCode) {

		try {

			HttpServletRequest req = WebContextFactory.get()
					.getHttpServletRequest();

			// get context
			Context ctx = (Context) req.getSession().getAttribute(
					ProfileConstants.context);

			// get actual template

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			MerchantStore store = mservice
					.getMerchantStore(ctx.getMerchantid());

			store.setTemplateModule(moduleName);
			mservice.saveOrUpdateMerchantStore(store);

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			CoreModuleService service = rservice.getCoreModuleService(
					countryCode, moduleName);
			
			List modules = new ArrayList();
			modules.add(store.getTemplateModule());
			
			//get module configuration
			Collection<ModuleConfiguration> moduleConf = rservice.getModuleConfigurations(modules);

			
			StringBuilder templateInformation = new StringBuilder();
			templateInformation.append(LabelUtil.getInstance().getText(ctx.getLang(),
					"module.storefront.modulename")).append(" ").append(store.getTemplateModule()).append("</br>");
			
			if(moduleConf!=null && moduleConf.size()>0) {
				
				
				for(Object o : moduleConf) {
					
					ModuleConfiguration conf = (ModuleConfiguration)o;
					
					if(conf.getId().getConfigurationKey().equals(ConfigurationConstants.LARGEIMAGEWIDTH_CONFIGURATION_KEY)) {
						
						templateInformation.append(LabelUtil.getInstance().getText(ctx.getLang(),
						"module.storefront.minimumimagewidth")).append(" ").append(conf.getConfigurationValue()).append(" px</br>");
						
					}
					
					if(conf.getId().getConfigurationKey().equals(ConfigurationConstants.LARGEIMAGEHEIGHT_CONFIGURATION_KEY)) {
						
						templateInformation.append(LabelUtil.getInstance().getText(ctx.getLang(),
						"module.storefront.minimumimageheight")).append(" ").append(conf.getConfigurationValue()).append(" px</br>");
					}
					
					if(conf.getId().getConfigurationKey().equals(ConfigurationConstants.BANNER_CONFIGURATION_KEY)) {
						
						templateInformation.append(LabelUtil.getInstance().getText(ctx.getLang(),
						"module.storefront.bannerwidth")).append(" ").append(conf.getConfigurationValue()).append("px </br>");
						
					}
					
					if(conf.getId().getConfigurationKey().equals(ConfigurationConstants.SLIDER_CONFIGURATION_KEY)) {
						
						templateInformation.append(LabelUtil.getInstance().getText(ctx.getLang(),
						"module.storefront.sliderimagesize")).append(" ").append(conf.getConfigurationValue()).append("</br>");
						
					}
					
					
				}
				
				
			}
			
			


			//String title = LabelUtil.getInstance().getText(ctx.getLang(),
			//		"module." + moduleName + ".title");

			
			service.setCoreModuleServiceDescription(templateInformation.toString());

			return service;

		} catch (Exception e) {
			log.error(e);
			return null;
		}

	}

}



```
