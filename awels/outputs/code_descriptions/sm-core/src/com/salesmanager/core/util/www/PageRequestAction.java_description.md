# PageRequestAction.java

## Review

## 1. Summary
`PageRequestAction` is an abstract Struts‑style action (likely part of a web‑app framework) that prepares all the data required to render a dynamic page (“paage” – probably a typo of *page*).  
The class:

1. **Parses** the current request URI to determine the logical page name.  
2. **Loads** the `MerchantStore` and `Page` entities via reference services.  
3. **Fetches** the list of portlets that belong to the page.  
4. **Pulls** merchant‑specific configuration values, parses them into `Field` objects, and makes them available in the request and an execution context.  
5. **Sets** a JSP template name (defaulting to `basic.jsp` if none defined).  

Key design elements:  
- Heavy use of service factories (`ServiceFactory.getService`) to obtain domain services (`ReferenceService`, `MerchantService`).  
- Dependency injection is minimal; services are fetched statically, which makes unit testing harder.  
- The class is abstract – concrete actions must implement the `display()` method to render the final view.

## 2. Detailed Description
### Initialization
- The action is invoked through a web request.  
- The `displayPage()` method is the entry point. It starts by determining a *pageAppender* (`/paage/` or `/fbPaage/`) based on the request URI.  
- It then removes the context path and a fixed `/integration` prefix to isolate the page identifier.

### Execution Flow
1. **Page Name Extraction**  
   ```java
   String pathnocontext = removeStart(requestURI, contextPath + "/integration" + pageAppender);
   String p = pathnocontext;
   int indexOfLastSlash = p.indexOf("/");
   if (indexOfLastSlash > 0) p = p.substring(0, indexOfLastSlash);
   setPageName(p.trim());
   ```
   The code trims the path to the first slash after the prefix, producing the logical page name.

2. **Store Retrieval**  
   The store is pulled from the request attribute `STORE` and stored in the action’s `store` field.

3. **Page & Portlet Loading**  
   ```java
   page = rservice.getPage(pageName, store.getMerchantId());
   portlets = rservice.getPortlets(page.getPageId(), store.getMerchantId());
   ```

4. **Configuration & Field Parsing**  
   - Build a `ConfigurationRequest` requesting all keys that match the page title.  
   - The `MerchantService` returns a list of `MerchantConfiguration`.  
   - `ConfigurationFieldUtil.parseFieldsValues()` turns raw configuration strings into a map of module → list of `Field` objects.  
   - All fields are flattened into the `fields` map (`fieldName → Field`) for easy access in the view.

5. **Context & Request Attributes**  
   - Execution context (`PageExecutionContext`) receives the fields map.  
   - Request attributes: `fields`, and `paageTemplate` (either the page’s `property1` or `basic.jsp`).

6. **Forward**  
   The method ends by invoking the abstract `display()` which concrete subclasses implement to forward to a JSP.

### Cleanup
No explicit cleanup is required; the action relies on the framework’s request lifecycle.

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| **displayPage()** | Main entry point; orchestrates page rendering preparation. | None | `String` (view name) | Sets request attributes, populates fields map, loads services, logs errors. |
| **display()** | Abstract; concrete actions supply the final view. | None | `String` | Depends on subclass implementation. |
| **getStore()/setStore(MerchantStore)** | Accessor for the merchant store. | `MerchantStore` | `MerchantStore` | None. |
| **getPage()/setPage(Page)** | Accessor for the page entity. | `Page` | `Page` | None. |
| **getExecutionContext()/setExecutionContext(PageExecutionContext)** | Accessor for execution context. | `PageExecutionContext` | `PageExecutionContext` | None. |
| **getPageName()/setPageName(String)** | Accessor for logical page name. | `String` | `String` | None. |
| **getFields()** | Returns map of field name → `Field`. | None | `Map<String, Field>` | None. |
| **getPortlets()** | Returns collection of portlet objects. | None | `Collection` | None. |

### Reusable/Utility Methods
- The class heavily relies on `ConfigurationFieldUtil` and the service factories, which are reusable across the application.

## 4. Dependencies
| Library / Class | Type | Notes |
|-----------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Apache Commons Lang – string utilities. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.core.entity.*` | Domain | Entities for merchant, page, field. |
| `com.salesmanager.core.service.*` | Domain | Service interfaces (`ReferenceService`, `MerchantService`). |
| `com.salesmanager.core.service.ServiceFactory` | Domain | Static factory for service lookup. |
| `com.salesmanager.core.util.ConfigurationFieldUtil` | Utility | Parses configuration strings into fields. |
| `BaseAction` | Inherited | Likely a framework base class providing `getServletRequest()` and other utilities. |

- All external dependencies are third‑party Java libraries; the code does **not** reference any platform‑specific APIs beyond the servlet request/response.

## 5. Additional Notes

### Strengths
- Centralizes page‑related data loading in one place.
- Decouples configuration parsing via a dedicated utility.
- Allows different page actions to reuse the same preparation logic.

### Weaknesses & Edge Cases
1. **Hard‑coded Path Parsing** – The logic that strips `/integration` and identifies the page name assumes a very specific URL format. Any deviation (e.g., trailing slashes, query strings, or different integration paths) will break the extraction.
2. **Static Service Lookup** – Using `ServiceFactory.getService()` everywhere prevents dependency injection and makes unit testing more cumbersome.
3. **Unparameterized Collections** – `Map fields = new HashMap();` and `Collection portlets = new ArrayList();` should use generics (`Map<String,Field>` and `Collection<Portlet>`).
4. **Typo “paage”** – The code consistently uses the misspelled term `paageTemplate` and `pageAppender`. This could cause confusion or bugs when referring to JSP names.
5. **Error Handling** – A generic catch block logs the error and returns `"MINIMALERROR"`. It would be better to propagate a meaningful error or use the framework’s error handling.
6. **Null Checks** – The code assumes non‑null values for `store`, `page`, and service calls; in production a null pointer could surface unexpectedly.

### Future Enhancements
- **Refactor URL parsing** into a dedicated helper or a configuration bean that can be overridden.
- **Inject services** via constructor or setter injection (or use a DI framework like Spring).
- **Use generics** everywhere to improve type safety.
- **Add unit tests** for `displayPage()` by mocking the request and services.
- **Handle optional fields** gracefully (e.g., missing `property1` or null page).
- **Rename all “paage” references** to `page` to avoid confusion.
- **Improve logging** to include request context and page name on errors.

Overall, the class provides a useful foundation for page actions but would benefit from modernizing its dependency handling, type safety, and resilience to URL changes.

## Code Critique



## Code Preview

```java
package com.salesmanager.core.util.www;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Page;
import com.salesmanager.core.entity.system.Field;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.ConfigurationFieldUtil;


public abstract class PageRequestAction extends BaseAction {
	
	private Logger log = Logger.getLogger(PageRequestAction.class);
	
	private MerchantStore store;
	private Page page;
	private PageExecutionContext executionContext;
	
	private String pageName;
	
	private Map<String,Field> fields = new HashMap();//String->module,Map<String,Field>
	
	private Collection portlets = new ArrayList();
	
	
	public String displayPage() {
		
		
		
		try {
			

		
		String pageAppender = "/paage/";
		//int charToRemove = 6;
		if(super
				.getServletRequest().getRequestURI().contains("/fbPaage/")) {
			pageAppender = "/fbPaage/";
			//charToRemove = 8;
		}
		
		String pathnocontext = StringUtils.removeStart(super
				.getServletRequest().getRequestURI(), super.getServletRequest()
				.getContextPath() + "/integration" + pageAppender);
		
		
		
		String p = pathnocontext;
		//		.substring(0, pathnocontext.length());
		
		//should have /fbPage/<PAGE>/
		int indexOfLastSlash = p.indexOf("/");
		if(indexOfLastSlash>0) {
			p = p.substring(0,indexOfLastSlash);
		}

		
		this.setPageName(p.trim());
		
		MerchantStore store =(MerchantStore) super.getServletRequest().getAttribute("STORE");
		
		this.setStore(store);
		
		ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		
		page = rservice.getPage(this.getPageName(), store.getMerchantId());
		
		
		//get configured portlets
		portlets = rservice.getPortlets(getPage().getPageId(), getStore().getMerchantId());
		
		//get fields
		MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
		ConfigurationRequest configRequest = new ConfigurationRequest(store.getMerchantId(),true,ConfigurationFieldUtil.getMerchantConfigurationKeyLike(page.getTitle()));
		ConfigurationResponse configResponse = mservice.getConfiguration(configRequest);
		
		List<MerchantConfiguration> configs = configResponse.getMerchantConfigurationList();
		Map<String,List<Field>> fieldValues = new HashMap();//String->module,List<Field>
		
		if(configs!=null && configs.size()>0) {
			List sArrayList = new ArrayList();
			for(Object o: configs) {
				MerchantConfiguration conf = (MerchantConfiguration)o;
				sArrayList.add(conf.getConfigurationValue());
			}
			fieldValues = ConfigurationFieldUtil.parseFieldsValues(sArrayList);
			for(Object oo: fieldValues.keySet()) {
				String module = (String)oo;
				List fieldsList = (List)fieldValues.get(module);
				for(Object ooo: fieldsList) {
					Field f = (Field)ooo;
					fields.put(f.getName(), f);
				}
			}
		}
		
		
		this.getExecutionContext().addToExecutionContext("fields", fields);
		super.getServletRequest().setAttribute("fields", fields);

		if(page != null && !StringUtils.isBlank(page.getProperty1())) {
			super.getServletRequest().setAttribute("paageTemplate", page.getProperty1());
		} else {
			super.getServletRequest().setAttribute("paageTemplate", "basic.jsp");
		}
		
		return display();
		
		} catch (Exception e) {
			log.error(e);
			return "MINIMALERROR";
		}
		
		
		
		
		
	}
	
	public abstract String display() throws Exception;
	
	public MerchantStore getStore() {
		return store;
	}
	public void setStore(MerchantStore store) {
		this.store = store;
	}
	public Page getPage() {
		return page;
	}
	public void setPage(Page page) {
		this.page = page;
	}
	public PageExecutionContext getExecutionContext() {
		return executionContext;
	}
	public void setExecutionContext(PageExecutionContext executionContext) {
		this.executionContext = executionContext;
	}

	public String getPageName() {
		return pageName;
	}

	public void setPageName(String pageName) {
		this.pageName = pageName;
	}

	public Map<String, Field> getFields() {
		return fields;
	}

	public Collection getPortlets() {
		return portlets;
	}

}



```
