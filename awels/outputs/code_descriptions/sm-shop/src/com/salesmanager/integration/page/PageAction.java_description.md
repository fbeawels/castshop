# PageAction.java

## Review

## 1. Summary  

**Purpose** – `PageAction` is a Struts‑style action that handles rendering of a merchant’s page (e.g., a Facebook page).  
**Key responsibilities**  
| Component | Role |
|-----------|------|
| `display()` | Main entry point that parses the request URI, looks up the corresponding `Page` record, pulls in the configured `Portlet`s, and decides how to render the page. |
| `page` field | Holds the `Page` instance that is ultimately displayed. |
| `getPage()/setPage()` | Standard bean accessors used by the view layer. |

**Notable patterns & libraries**  
* **Factory pattern** – `ServiceFactory.getService(...)` obtains a `ReferenceService`.  
* **Command pattern** – `display()` encapsulates a series of actions that result in a view name (`SUCCESS` or `ERROR`).  
* **Struts/Action** – Extends `PageRequestAction`, presumably a custom base class providing `getServletRequest()` and `getServletResponse()`.  
* **Commons‑Lang / Log4j** – Used for string manipulation and logging.  

The code is tightly coupled to the `com.salesmanager` domain model and does not use generics, which makes it harder to reason about types safely.

---

## 2. Detailed Description  

1. **URI Normalisation**  
   * `pathnocontext` removes the context path from the request URI.  
   * The first segment after the context path is taken as the *page identifier* (`p`).  
   * No sanity checks are performed – if the URI does not contain a `/`, `indexOf("/")` returns `-1` and a `StringIndexOutOfBoundsException` will be thrown.  

2. **MerchantStore Retrieval**  
   * The current `MerchantStore` is read from the request attributes (`"MERCHANTSTORE"`).  
   * The code assumes the attribute is present and non‑null; a missing store will lead to a `NullPointerException` when `store.getMerchantId()` is called.  

3. **Page Lookup**  
   * `ReferenceService` is obtained via `ServiceFactory`.  
   * `getPage(p, store.getMerchantId())` fetches the page definition from the DB.  
   * If the page is `null`, the method returns `ERROR` (presumably a Struts result).  

4. **Portlet Collection & Dispatch**  
   * `getPortlets(page.getPageId(), store.getMerchantId())` returns a raw `Collection` of portlet objects.  
   * The loop casts each element to `Portlet` and checks the portlet type.  
   * The actual handling for module and label types is omitted – placeholders remain.  

5. **Return Value**  
   * If everything succeeds, `SUCCESS` is returned.  
   * The method declares `throws Exception` but never throws any checked exception itself.  

The class is essentially a thin controller that orchestrates service calls and decides the view name. Cleanup or transaction handling is not shown (likely managed by the framework).

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `display()` | Main action handler. Parses request, retrieves data, dispatches portlets, and returns a result string. | None | `String` (`SUCCESS` or `ERROR`) | Reads request attributes, writes to `page` field. |
| `getPage()` | Bean getter for the `page` field. | None | `Page` | None |
| `setPage(Page page)` | Bean setter for the `page` field. | `Page` | void | Sets the internal field. |

### Reusable/Utility Methods  
* None – the class contains only the action logic.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | 3rd‑party | Provides `removeStart`; could be replaced with Java 11+ `String.startsWith` and substring. |
| `org.apache.log4j.Logger` | 3rd‑party | Classic Log4j logger; may be legacy (Log4j 1.x). |
| `com.salesmanager.core.*` | 3rd‑party | Domain model (`Page`, `Portlet`, `MerchantStore`), services (`ReferenceService`, `ServiceFactory`). |
| `com.salesmanager.core.util.www.PageRequestAction` | 3rd‑party | Base action class providing request/response access. |
| `java.util.Collection` | JDK | Used raw; could be generified (`Collection<Portlet>`). |

All dependencies are third‑party, with the exception of the JDK collections.

---

## 5. Additional Notes  

### Edge‑cases & Robustness  
1. **URI Parsing** – If the request URI does not contain a `/` after the context path, `indexOf("/")` returns `-1`, causing a crash.  
2. **Null Checks** – `store`, `page`, and the returned collection are never checked for `null`.  
3. **Type Safety** – Using raw `Collection` forces unchecked casts.  
4. **Exception Handling** – The method declares `throws Exception` but does not catch or wrap any runtime exceptions.  
5. **Logging** – No log statements are present; useful for debugging failures.  

### Design Improvements  
* **Use generics**: `Collection<Portlet> portlets = rservice.getPortlets(...);`  
* **Add defensive coding**: validate URI, ensure `store` and `page` are non‑null, handle missing portlets gracefully.  
* **Separate concerns**: move portlet dispatch logic into a dedicated service or helper class; keep the action thin.  
* **Logging**: log important milestones (e.g., page not found, number of portlets loaded).  
* **Modern string utilities**: replace `StringUtils` with Java 11+ `String` methods.  
* **Dependency injection**: inject `ReferenceService` rather than fetching via a static factory.  
* **Result constants**: replace magic strings (`SUCCESS`, `ERROR`) with constants or an enum for clarity.  

### Future Enhancements  
* Implement the placeholders for module and label portlet types.  
* Cache page/portlet data to reduce DB load.  
* Support additional portlet types (e.g., dynamic widgets).  
* Add unit tests for the action logic (mocking the service layer).  
* Migrate to a more recent logging framework (Log4j 2 or SLF4J).  

Overall, the class is functional but would benefit from modern Java practices, better error handling, and clearer separation of responsibilities.

## Code Critique



## Code Preview

```java
package com.salesmanager.integration.page;

import java.util.Collection;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Page;
import com.salesmanager.core.entity.reference.Portlet;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.www.PageRequestAction;

public class PageAction extends PageRequestAction {
	private Logger log = Logger.getLogger(PageAction.class);
	
	private Page page = null;
	
	public String display() throws Exception {
		

			String pathnocontext = StringUtils.removeStart(super
					.getServletRequest().getRequestURI(), super.getServletRequest()
					.getContextPath()
					+ "/integration/");
			
			String p = pathnocontext
					.substring(0, pathnocontext.indexOf("/"));
			
			MerchantStore store =(MerchantStore) super.getServletRequest().getAttribute("MERCHANTSTORE");
			
			//get page from the database
			
			ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
			
			
			//get Page
			page = rservice.getPage(p, store.getMerchantId());//facebook page
			
			if(page==null) {
				return ERROR;
			}
			
			
			//invoke portlets
			Collection configuredPortlets = rservice.getPortlets(page.getPageId(), store.getMerchantId());
			
			for(Object o: configuredPortlets) {
				
				Portlet portlet = (Portlet)o;
				
				if(portlet.getPortletType().intValue()==LabelConstants.PORTLET_TYPE_MODULE) {

					//invoke module
					
				} else if(portlet.getPortletType().intValue()==LabelConstants.PORTLET_TYPE_LABEL) {
					

				}
				
			}
			

		
		return SUCCESS;
		
		
	}

	public Page getPage() {
		return page;
	}

	public void setPage(Page page) {
		this.page = page;
	}

}



```
