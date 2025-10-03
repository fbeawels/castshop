# FbDownloadProducts.java

## Review

## 1. Summary
- **Purpose**: A Spring‑managed module (`FbDownloadProducts`) that retrieves a list of downloadable products from the catalog (based on a relation type defined in the module’s configuration) and exposes them to the page execution context for rendering.
- **Key components**  
  - `display(...)` – the core method that reads the configuration, queries the catalog, localises the result, and stores it for the view.  
  - `requiresAuthorization()` – indicates whether a user must be authenticated to use the module.  
  - `submit(...)` – a no‑op placeholder for future form handling.  
- **Design patterns / frameworks**  
  - Spring’s `@Component` for dependency injection.  
  - A simple **Module** interface (`PortletModule`) which is part of a custom integration framework.  
  - Service locator pattern (`ServiceFactory.getService`) instead of constructor/field injection.

---

## 2. Detailed Description
### Flow of Execution
1. **Configuration Extraction**  
   - The module pulls a map of configuration fields from `pageContext`.  
   - It looks up the field named `"downloadRelationType"` and attempts to parse its value as an `int`.  
   - If parsing fails, a merchant‑specific log entry is written.

2. **Catalog Query**  
   - The `CatalogService` is obtained via `ServiceFactory`.  
   - `getProductRelationShip(-1, storeId, relType, locale, true)` is called. The first argument (`-1`) appears to be a placeholder for “all products”; this is a hard‑coded value that could be refactored into a constant or a meaningful method parameter.

3. **Localisation**  
   - `LocaleUtil.setLocaleToEntityCollection(prods, locale, store.getCurrency());` enriches each product entity with the appropriate language and currency data.

4. **Context Injection**  
   - The resulting product collection is put into `pageContext` under the key `"downloadproducts"`, making it available to the view layer.

5. **Authorization & Submission**  
   - `requiresAuthorization()` always returns `false`.  
   - `submit(...)` contains no logic – it is a stub.

### Assumptions & Constraints
- The configuration map **always** contains a `"downloadRelationType"` key (otherwise `field` is `null`).  
- The `getProductRelationShip` method returns a non‑`null` collection; otherwise `LocaleUtil` would throw a `NullPointerException`.  
- The `-1` passed to the service is assumed to mean “fetch all related products”; the meaning is implicit and undocumented.

### Architecture & Design Choices
- **Service Locator** (`ServiceFactory`) is used instead of Spring’s dependency injection; this can make unit testing harder and hides the module’s dependencies.  
- Raw types (`Map`, `Collection`) are used throughout, sacrificing type safety.  
- Logging is performed via a static utility (`LogMerchantUtil`); no standard SLF4J/Log4J façade is used.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `display(MerchantStore store, HttpServletRequest request, Locale locale, PageRequestAction action, PageExecutionContext pageContext)` | Reads module configuration, queries the catalog for downloadable products, localises them, and stores the result in `pageContext`. | *`store`* – merchant store context.<br>*`request`* – current HTTP request (unused).<br>*`locale`* – locale for localisation.<br>*`action`* – encapsulates store, locale and request data.<br>*`pageContext`* – context map for the page. | `void` | Populates `pageContext` with `"downloadproducts"`; logs an error if the relation type cannot be parsed. |
| `requiresAuthorization()` | Indicates whether a logged‑in user is required. | None | `false` (module is public). | None |
| `submit(MerchantStore store, HttpServletRequest request, Locale locale, PageRequestAction action, PageExecutionContext pageContext)` | Placeholder for form submission logic. | Same as `display`. | `void` | None (currently no implementation). |

---

## 4. Dependencies

| Library/Framework | Usage | Notes |
|-------------------|-------|-------|
| **Spring Framework** (`org.springframework.stereotype.Component`) | Component registration & autowiring | Standard Spring MVC/Portlet integration. |
| **Servlet API** (`javax.servlet.http.HttpServletRequest`) | Request handling (unused in this module) | Standard JEE. |
| **SalesManager Core** | `MerchantStore`, `Field`, `PortletModule`, `ServiceFactory`, `CatalogService`, `LocaleUtil`, `LogMerchantUtil`, `PageExecutionContext`, `PageRequestAction` | Custom application modules. |
| **Java Standard Library** | `java.util.*`, `java.util.Locale` | Standard. |

All external dependencies are internal to the SalesManager platform; no third‑party libraries are referenced.

---

## 5. Additional Notes

### Strengths
- Clear separation of concerns: the module only handles data retrieval and context population.  
- Uses existing services (`CatalogService`) and utilities (`LocaleUtil`) which promotes reuse.

### Potential Issues & Edge Cases
1. **Raw Types**  
   - `Map fields` and `Collection prods` are raw; this hides compile‑time type safety and can lead to `ClassCastException` at runtime.  
   - Recommendation: use generics, e.g. `Map<String, Field>` and `Collection<Product>`.

2. **Hard‑coded `-1`**  
   - The first argument to `getProductRelationShip` is a magic number. It would be clearer to declare a constant or expose it as a configurable parameter.

3. **Missing Null Checks**  
   - `pageContext.getFromExecutionContext("fields")` could return `null`.  
   - `pageContext.addToExecutionContext` may silently overwrite existing data.  
   - `getProductRelationShip` may return `null` or throw; no exception handling is present.

4. **Logging**  
   - Uses a static utility instead of a conventional logging framework (SLF4J, Log4j).  
   - The log message concatenates the merchant ID but the format could be clearer.  
   - Consider using parameterised logging (`log.warn("... {}", e.getMessage());`).

5. **Service Acquisition**  
   - The module pulls `CatalogService` from a static `ServiceFactory`.  
   - This approach makes unit testing harder; injecting the service via constructor or field injection would improve testability.

6. **Internationalisation of Log Message**  
   - The error message is hard‑coded in English. If the system supports multiple locales, the message should be externalised.

7. **Unimplemented `submit`**  
   - The method is a stub; if the module is later extended to handle form submissions, the signature should align with the rest of the framework’s expectations.

### Suggested Enhancements
- **Inject `CatalogService` via Spring** (`@Autowired` constructor) and remove `ServiceFactory`.  
- Replace raw types with generics.  
- Extract the relation‑type parsing into a small helper method for clarity and reuse.  
- Add unit tests that mock `CatalogService` and verify that the correct collection is placed into `pageContext`.  
- Consider making the relation type configurable via `pageContext` or a dedicated configuration property, rather than a hard‑coded fallback.  
- Implement `submit` if needed or remove it from the interface to avoid accidental calls.

---

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Jan 12, 2011 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.integration.modules;

import java.util.Collection;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.springframework.stereotype.Component;

import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.system.Field;
import com.salesmanager.core.module.model.integration.PortletModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.LogMerchantUtil;
import com.salesmanager.core.util.www.PageExecutionContext;
import com.salesmanager.core.util.www.PageRequestAction;

/**
 * Get downloadable products from FB dowload product list
 * displays items as regular items but switches detail buttons
 * with download link
 * @author Carl Samson
 *
 */
@Component("downloadproducts")
public class FbDownloadProducts implements PortletModule {

	public void display(MerchantStore store, HttpServletRequest request,
			Locale locale, PageRequestAction action,
			PageExecutionContext pageContext) {
		
		
		//get configuration for this module from execution context
		Map fields = (Map)pageContext.getFromExecutionContext("fields");
		
		//get field by fieldName
		Field field = (Field)fields.get("downloadRelationType");
		
		int relType = CatalogConstants.PRODUCT_RELATIONSHIP_FBPAGE_DOWNLOADS;
		
		if(field!=null) {
			
			try {
				relType = Integer.parseInt(field.getFieldValue());
			} catch (Exception e) {
				LogMerchantUtil.log(store.getMerchantId(), "Invalid value for FB Product display (relationType, the value should be numeric");
			}
		}
			
			
		
		CatalogService cService = (CatalogService) ServiceFactory
		.getService(ServiceFactory.CatalogService);
		
		Collection prods = cService
			.getProductRelationShip(
				-1,
				action.getStore().getMerchantId(),
				relType,
				action.getLocale().getLanguage(), true);
		
		
		LocaleUtil.setLocaleToEntityCollection(prods, locale, store.getCurrency());
		
		pageContext.addToExecutionContext("downloadproducts", prods);
		
	

	}

	public boolean requiresAuthorization() {
		// TODO Auto-generated method stub
		return false;
	}

	public void submit(MerchantStore store, HttpServletRequest request,
			Locale locale, PageRequestAction action,
			PageExecutionContext pageContext) {
		// TODO Auto-generated method stub

	}

}



```
