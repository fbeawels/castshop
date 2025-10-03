# TopCategoriesTag.java

## Review

## 1. Summary  
**Purpose** – `TopCategoriesTag` is a custom JSP tag that renders the top‑level product categories of a merchant’s store.  
**Key Components**  
| Component | Role |
|-----------|------|
| `CatalogService` | Loads category data from the database. |
| `CacheModule` | Caches the category list per locale to reduce database traffic. |
| `MerchantStore` | Holds merchant‑specific data (domain, etc.) used for URL construction. |
| `ReferenceUtil` | Builds secure / unsecured domain URLs. |
| `PageContext` | Supplies tag attributes and injects the tag body into the page. |

The tag follows a very simple **“read‑then‑render”** pattern: fetch categories (with caching), iterate, set a handful of attributes, and invoke the body once per category. No design patterns beyond basic MVC separation are evident; the code relies heavily on the underlying Spring infrastructure for bean resolution.

---

## 2. Detailed Description  

### Initialization  
1. **Service acquisition** – `CatalogService` is fetched through `ServiceFactory.getService(...)`.  
2. **Request context** – The current `HttpServletRequest`, `HttpSession`, and a locale attribute (retrieved via `request.getAttribute("LOCALE")`) are obtained.  
3. **Merchant information** – `MerchantStore` is fetched from the session with `SessionUtil.getMerchantStore(request)`.  

### Caching Logic  
* Try to pull the list of top‑level categories from the cache using a key composed of `Constants.CACHE_CATEGORIES_TOP + locale.getLanguage()`.  
* If the cache lookup fails or the key is missing, query the database via `cservice.getSubCategoriesByParentCategoryAndLang(...)` and, if a non‑null list is returned, store it back in the cache.

### Rendering Loop  
For each category in the list:  

| Condition | Action |
|-----------|--------|
| `index == maxCategories` | Stop processing further categories. |
| `categoryId == 0` or `!category.isVisible()` | Skip the category. |
| Otherwise | |
| - Set page‑context attributes: `category`, `contextPath`, `securedDomain`, `unSecuredDomain`. |
| - Increment `index`. |
| - If `index == lineBreakQuantity` (default = 4) set attribute `break` to `"<br>"`. |
| - Invoke the body of the tag (`getJspBody().invoke(null)`), which renders whatever JSP snippet the developer placed inside the tag. |

The tag ends silently; if any exception occurs during processing it is logged but otherwise swallowed.

### Cleanup  
There is no explicit cleanup logic – the tag relies on the container to destroy the `PageContext` and session after the page is rendered.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `doTag()` | Entry point for the custom tag. Executes the rendering logic. | none | writes to the JSP response | logs errors, sets page‑context attributes, invokes tag body |
| `getMerchantId()` | Getter for `merchantId`. | none | int | none |
| `setMerchantId(int)` | Setter for tag attribute `merchantId`. | merchantId | none | updates field |
| `getMaxCategories()` | Getter for `maxCategories`. | none | int | none |
| `setMaxCategories(int)` | Setter for tag attribute `maxCategories`. | maxCategories | none | updates field |

**Reusable / Utility** – None; all logic is tightly coupled to the tag’s responsibilities.

---

## 4. Dependencies  

| Library / Framework | Type | Notes |
|---------------------|------|-------|
| `javax.servlet.jsp.tagext.SimpleTagSupport` | Standard (Servlet/JSP) | Base class for custom tags. |
| `org.apache.log4j.Logger` | Third‑party (deprecated) | For logging. |
| `com.salesmanager.core` packages | Third‑party | Domain services, constants, entity definitions, and utilities specific to the SalesManager e‑commerce platform. |
| `SpringUtil` | Third‑party (Spring) | Bean lookup. |
| `ServiceFactory` | Third‑party | Static service locator. |
| `CacheModule` | Third‑party | Caching abstraction used by SalesManager. |

No native platform dependencies beyond the servlet/JSP container and the SalesManager framework.

---

## 5. Additional Notes & Recommendations  

### 5.1 Robustness  
* **Locale handling** – The tag fetches the locale via `request.getAttribute("LOCALE")`. If the attribute is missing, a `NullPointerException` will be thrown. It is safer to use `request.getLocale()` or to guard against `null`.  
* **`maxCategories` default** – If the tag is used without setting `maxCategories`, the default value `0` causes the loop to break immediately. A better default would be `Integer.MAX_VALUE` or the size of the collection.  
* **Exception handling** – Swallowing cache lookup failures silently can hide problems. At minimum, log a warning.  

### 5.2 Maintainability  
* **Hard‑coded break index** – `BREAK_INDEX` is a magic number. Expose it as a tag attribute (`lineBreakQuantity`) or configure it via a properties file.  
* **Coupling to `SessionUtil`** – The tag assumes the presence of a `MerchantStore` in the session. Consider passing the store directly as a tag attribute to make the tag more reusable.  
* **Logging** – `org.apache.log4j` is legacy. Upgrade to SLF4J + Logback (or Java Util Logging) for better performance and flexibility.  

### 5.3 Performance  
* **Cache key construction** – The key is built as `Constants.CACHE_CATEGORIES_TOP + locale.getLanguage()`. If multiple merchants share a cache, consider including the merchant identifier to avoid key collisions.  
* **Body invocation** – Calling `getJspBody().invoke(null)` for each category can be expensive if the body contains complex JSP code. Evaluate if a single JSP fragment with a `<c:forEach>` loop could replace the custom tag.  

### 5.4 Testability  
* The tag currently depends on static service locators and the servlet request. Refactor to accept services via dependency injection or via a helper object to allow unit‑testing the logic in isolation.  

### 5.5 Edge Cases  
* **Empty category list** – If `categories` is `null` or empty, the tag renders nothing. Consider rendering a “no categories available” message.  
* **Very large category sets** – The loop can become slow if the number of categories is large. Pagination or limiting the number displayed (via `maxCategories`) is advisable.

---

### Suggested Refactor Skeleton  

```java
public class TopCategoriesTag extends SimpleTagSupport {

    private static final String BREAK_TAG = "<br>";
    private static final int DEFAULT_BREAK_INDEX = 4;
    private static final int DEFAULT_MAX_CATEGORIES = Integer.MAX_VALUE;

    private int merchantId;
    private int maxCategories = DEFAULT_MAX_CATEGORIES;
    private int lineBreakQuantity = DEFAULT_BREAK_INDEX;

    // Injected dependencies (for testability)
    private CatalogService catalogService;
    private CacheModule cacheModule;

    @Override
    public void doTag() throws JspException, IOException {
        try {
            // Acquire request, locale, store
            HttpServletRequest req = (HttpServletRequest) getJspContext().getAttribute(PageContext.REQUEST);
            Locale locale = req.getLocale();
            MerchantStore store = (MerchantStore) req.getAttribute("STORE");
            if (store == null) throw new JspException("Missing store");

            Collection<Category> categories = loadCategories(locale, store);
            renderCategories(categories, locale, store);
        } catch (Exception e) {
            log.error("TopCategoriesTag error", e);
            throw new JspException("Failed to render top categories", e);
        }
    }

    private Collection<Category> loadCategories(Locale locale, MerchantStore store) {
        String key = Constants.CACHE_CATEGORIES_TOP + locale.getLanguage();
        Collection<Category> categories = cacheModule.getFromCache(key, store);
        if (categories == null) {
            categories = catalogService.getSubCategoriesByParentCategoryAndLang(
                    merchantId, CatalogConstants.ROOT_CATEGORY_ID, locale.getLanguage());
            if (categories != null) {
                cacheModule.putInCache(key, categories, Constants.CACHE_CATEGORIES, store);
            }
        }
        return categories != null ? categories : Collections.emptyList();
    }

    private void renderCategories(Collection<Category> categories, Locale locale, MerchantStore store) throws IOException {
        int index = 0;
        for (Category cat : categories) {
            if (index >= maxCategories || cat.getCategoryId() == 0 || !cat.isVisible()) {
                continue;
            }
            PageContext pc = (PageContext) getJspContext();
            pc.setAttribute("category", cat);
            pc.setAttribute("contextPath", ((HttpServletRequest) pc.getRequest()).getContextPath());
            pc.setAttribute("securedDomain", ReferenceUtil.getSecureDomain(store));
            pc.setAttribute("unSecuredDomain", ReferenceUtil.getUnSecureDomain(store));

            if (index + 1 == lineBreakQuantity) {
                pc.setAttribute("break", BREAK_TAG);
            }

            getJspBody().invoke(null);
            index++;
        }
    }
}
```

*The refactor keeps the same functionality but isolates business logic, uses safer defaults, and removes hard‑coded constants.*

---

### Final Verdict  
The tag is functional and fits into the existing SalesManager framework. However, it suffers from tight coupling, hard‑coded values, and brittle error handling. By refactoring to inject services, using standard locale handling, exposing configurable attributes, and upgrading the logging framework, the tag can become more robust, maintainable, and testable.

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
package com.salesmanager.core.util.www.tags;

import java.io.IOException;
import java.util.Collection;
import java.util.Locale;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import javax.servlet.jsp.JspException;
import javax.servlet.jsp.PageContext;
import javax.servlet.jsp.tagext.SimpleTagSupport;

import org.apache.log4j.Logger;

import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.catalog.Category;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.module.model.application.CacheModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class TopCategoriesTag extends SimpleTagSupport {

	private Logger log = Logger.getLogger(TopCategoriesTag.class);
	private static final int BREAK_INDEX = 4;
	private static final long serialVersionUID = 1L;
	private int merchantId;
	private int maxCategories;
	private int lineBreakQuantity = BREAK_INDEX;

	@Override
	public void doTag() throws JspException, IOException {
		try {
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			HttpServletRequest request = ((HttpServletRequest) ((PageContext) getJspContext())
					.getRequest());
			HttpSession session = request.getSession();
			Locale locale = (Locale) request.getAttribute("LOCALE");

			MerchantStore store = SessionUtil.getMerchantStore(request);

			// get root categories from cache
			Collection<Category> categories = null;
			CacheModule cache = (CacheModule) SpringUtil.getBean("cache");
			try {
				categories = (Collection) cache.getFromCache(
						Constants.CACHE_CATEGORIES_TOP + locale.getLanguage(),
						store);
			} catch (Exception ignore) {

			}

			if (categories == null) {

				categories = cservice.getSubCategoriesByParentCategoryAndLang(
						merchantId, CatalogConstants.ROOT_CATEGORY_ID, locale
								.getLanguage());

				if (categories != null) {

					try {
						cache.putInCache(Constants.CACHE_CATEGORIES_TOP
								+ locale.getLanguage(), categories,
								Constants.CACHE_CATEGORIES, store);
					} catch (Exception e) {
						log.error(e);
					}
				}

			}
			int index = 0;
			// int currentCount = 1;
			for (Category category : categories) {
				if (maxCategories == index) {
					break;
				} else if (category.getCategoryId() == 0) {
					continue;
				}
				if (!category.isVisible()) {
					continue;
				}
				// getJspContext().setAttribute("lastIndex", "");
				// if(currentCount==index) {
				// getJspContext().setAttribute("lastIndex", index);
				// }

				getJspContext().setAttribute("category", category);
				getJspContext().setAttribute("contextPath",
						request.getContextPath());
				getJspContext().setAttribute(
						"securedDomain",
						ReferenceUtil.getSecureDomain((MerchantStore) request
								.getAttribute("STORE")));
				getJspContext().setAttribute(
						"unSecuredDomain",
						ReferenceUtil.getUnSecureDomain((MerchantStore) request
								.getAttribute("STORE")));
				index++;
				if (index == lineBreakQuantity) {
					getJspContext().setAttribute("break", "<br>");
				}
				getJspBody().invoke(null);

			}
		} catch (Exception e) {
			log.error(e);
		}
	}

	public int getMerchantId() {
		return merchantId;
	}

	public void setMerchantId(int merchantId) {
		this.merchantId = merchantId;
	}

	public int getMaxCategories() {
		return maxCategories;
	}

	public void setMaxCategories(int maxCategories) {
		this.maxCategories = maxCategories;
	}
}



```
