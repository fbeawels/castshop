# CategoryUtil.java

## Review

## 1. Summary

**Purpose**  
`CategoryUtil` is a static helper library that supports JSP pages (particularly `categories.jsp`) in a retail/ERP web application. It provides a handful of convenience methods for:

* Counting the number of products that belong to a category (including sub‑categories).
* Checking whether a category actually contains products.
* Building a breadcrumb style “path” for a category.
* Retrieving the root category for a given sub‑category.
* Generating a hierarchical, indented list of all categories for a drop‑down UI component.
* Internally, recursively walking the category tree.

**Key Components**

| Class | Role |
|-------|------|
| `CategoryUtil` | Static façade that delegates to `CatalogService` and uses a simple cache (`CacheUtil`). |
| `CatalogService` | External service (via `ServiceFactory`) that fetches categories and product counts from the data store. |
| `CacheUtil` | Custom in‑memory cache used to avoid repeated DB calls for the drop‑down list. |
| `CategoryPadding` | DTO that stores an indented display string and a category ID – used to present the drop‑down hierarchy. |

**Design Patterns / Libraries**

* **Service Locator** – `ServiceFactory.getService()` is used instead of dependency injection.
* **Cache** – a simple map‑based cache is implemented by `CacheUtil`.
* **Singleton** – `CategoryUtil` has a private constructor, making it effectively a singleton of static methods.
* **Apache Commons Configuration** – used to read configuration properties.
* **Apache Log4j** – for logging.
* **Java Collections** – raw collections (`ArrayList`, `LinkedHashMap`, `TreeMap`) are heavily employed; generics are largely absent.

The code predates modern Java (prior to Java 5 generics) and thus relies on unchecked casts and raw types.

---

## 2. Detailed Description

### Initialization
The class contains a private static block that is empty – no actual initialization logic. The static fields `log` and `conf` are instantiated at class load time.

### Execution Flow
1. **Request‑based operations** (`getItemPerCategoryCount`, `categoryHasItems`) rely on attributes stored in the `HttpServletRequest` or `HttpSession`.  
2. **Category path retrieval** (`getCategoryPath`) fetches a map of all categories for a merchant and language from `CatalogService`, then walks up the tree from the target category to the root, reversing the list at the end.  
3. **Drop‑down list creation** (`getCategoriesForDropDownBox`) first checks a cache. If a cache miss occurs, it loads all categories, organizes them by parent ID, then recursively flattens the hierarchy while generating indentation.  
4. **Recursive traversal** is performed by `walkCategories`, which populates a flat `Map` of `CategoryPadding` objects.

### Assumptions / Constraints
* The request/session attributes must be correctly populated (`PROODUCTS` typo is a real bug; should be `PRODUCTS` or another agreed key).
* Category IDs are `long` and use `0` to signal the root parent.
* The `CatalogService` is expected to be thread‑safe since the util is stateless.
* Cache entries are identified by the language string only; no merchant ID is considered (potential bug in multi‑merchant environments).
* The code expects a property `core.cachecategoriiesstructure` to be `"true"` or `"false"`.

### Architecture Choices
The util adopts a *pull‑style* architecture: it explicitly pulls data from the service layer for each request. It also mixes data access (`CatalogService`) with presentation logic (indenting strings). This violates the separation‑of‑concerns principle; a cleaner approach would separate data retrieval from UI formatting.

---

## 3. Functions / Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `private CategoryUtil()` | Prevents instantiation. | – | – | – |
| `static String getItemPerCategoryCount(HttpServletRequest, String, Category)` | Counts how many products belong to a category and its sub‑categories, returning a formatted string for display. | `HttpServletRequest req`, `String lang`, `Category category` | `<b><font color='red'>[count]</font></b>` or empty string | Logs on exception |
| `static boolean categoryHasItems(HttpServletRequest, long)` | Checks if a session cache contains a product count for the given category ID. | `HttpServletRequest req`, `long categoryid` | `true` if count exists, `false` otherwise | Reads from session |
| `static List getCategoryPath(String, int, long)` | Builds a breadcrumb list from the target category up to the root. | `String lang`, `int merchantId`, `long categoryid` | `List<Category>` | Logs on exception |
| `static Category getRootCategoryforCategory(String, int, long)` | Convenience wrapper that returns the top‑level category in the path. | `String lang`, `int merchantId`, `long categoryid` | `Category` | Uses `getCategoryPath` |
| `static List getCategoriesForDropDownBox(int, String)` | Produces an indented list of categories for a drop‑down UI component. Uses caching if enabled. | `int merchantId`, `String lang` | `List<CategoryPadding>` | May hit cache or query DB; uses `log` |
| `private static Map walkCategories(Map, Map, long)` | Recursive helper that flattens the category hierarchy into an ordered map of `CategoryPadding`. | `Map returnMap`, `Map classification`, `long categoryId` | `Map` (the updated `returnMap`) | Mutates `returnMap` |

### Reusable / Utility Methods
`walkCategories` is the only private helper. The class relies heavily on the external `CatalogService` for data, so there are no generic utilities beyond this recursion.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Java EE | Standard servlet API |
| `org.apache.commons.configuration.Configuration` | Third‑party | For reading application properties |
| `org.apache.log4j.Logger` | Third‑party | Logging framework |
| `com.salesmanager.core.constants.CatalogConstants` | Project | Holds constant values (e.g., ROOT_CATEGORY_ID) |
| `com.salesmanager.core.entity.catalog.Category` | Project | Domain model |
| `com.salesmanager.core.entity.catalog.CategoryPadding` | Project | Simple DTO for display |
| `com.salesmanager.core.service.ServiceFactory` | Project | Service locator |
| `com.salesmanager.core.service.catalog.CatalogService` | Project | Business service |
| `com.salesmanager.core.util.PropertiesUtil` | Project | Loads configuration |
| `com.salesmanager.core.util.CacheUtil` | Project | Simple in‑memory cache |

All external dependencies are **third‑party** (Apache Commons, Log4j). No platform‑specific assumptions beyond the servlet container.

---

## 5. Additional Notes

### Strengths
* Keeps presentation logic localized to a single helper, reducing duplication in JSPs.
* Uses caching to avoid repetitive DB calls for the drop‑down list.
* Provides useful debugging output via Log4j.

### Issues / Edge Cases
1. **Raw Types / Unchecked Casts** – The code compiles with many unchecked warnings. This can lead to `ClassCastException` at runtime if the expected types differ.
2. **Typos** – The request attribute `"PROODUCTS"` is almost certainly a typo; it will return `null` leading to a count of zero or a `NullPointerException`.
3. **Null / Empty Checks** – Several methods silently swallow `null` values or return empty collections; callers may misinterpret the absence of data as legitimate.
4. **Cache Invalidation** – No mechanism to clear or refresh the cached drop‑down list when categories change. The cache key only contains language, not merchant ID, causing cross‑merchant leakage.
5. **Concurrency** – The static `CacheUtil` instance is shared; if `walkCategories` modifies the shared map in an unsynchronized way, a race condition could arise.
6. **Separation of Concerns** – Formatting logic (indentation via `&nbsp;`) is mixed with data retrieval. A cleaner approach would separate a “view model” builder from the raw service layer.
7. **Performance** – `getCategoriesForDropDownBox` re‑builds the hierarchy every time it is not cached, performing multiple passes over the category list. This could be optimized with a single traversal.
8. **Internationalization** – The method `getItemPerCategoryCount` embeds HTML and a hard‑coded color; such formatting should be delegated to the view layer or a templating engine.

### Potential Enhancements
* **Generics** – Refactor all collections to use typed generics, removing raw types and cast warnings.
* **Dependency Injection** – Replace `ServiceFactory` with a DI framework (e.g., Spring) to make the code more testable.
* **Cache Improvements** – Include merchant ID in cache keys, and expose cache eviction methods.
* **Unit Tests** – Introduce JUnit tests covering normal, edge, and error scenarios. Mock `CatalogService` and `HttpServletRequest`.
* **Configuration Validation** – Validate `core.cachecategoriiesstructure` at startup, defaulting to `false` if missing.
* **Refactor Formatting** – Move HTML generation to JSP or a dedicated helper, returning plain data (e.g., counts) from the util.
* **Logging Enhancements** – Log method entry/exit and parameter values to aid debugging, and use parameterized logging to avoid string concatenation overhead.
* **Exception Handling** – Rather than swallowing all exceptions, throw custom checked exceptions that callers can react to.

### Final Recommendation
While `CategoryUtil` currently serves its purpose in the legacy codebase, it would benefit significantly from a modernization effort. Removing raw types, improving caching logic, decoupling data and presentation layers, and adding tests would increase maintainability, reduce runtime errors, and prepare the code for future features. If the project is being migrated to a modern framework, consider rewriting these utilities as Spring components or using a dedicated view‑model layer.

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

import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.LinkedList;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.catalog.Category;
import com.salesmanager.core.entity.catalog.CategoryPadding;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;

/**
 * Helper for categories.jsp
 * 
 * @author Carl Samson
 * 
 */
public class CategoryUtil {

	private CategoryUtil() {
	}

	private static Logger log = Logger.getLogger(CategoryUtil.class);

	private static Configuration conf = PropertiesUtil.getConfiguration();

	static {

	}

	/**
	 * Get the number of products per category
	 * 
	 * @param req
	 * @param categoryid
	 * @return
	 */
	public static String getItemPerCategoryCount(HttpServletRequest req,
			String lang, Category category) {

		try {

			CatalogService service = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			int count = service.countProductsPerCategoryAndSubCategories(
					(List) req.getAttribute("PROODUCTS"), lang, category);

			if (count == 0) {
				return "";
			} else {
				return "<b><font color='red'>[" + count + "]</font></b>";
			}

		} catch (Exception e) {
			log.error(e);
			return "";
		}

	}

	/**
	 * Determine if a catagory has products
	 * 
	 * @param req
	 * @param categoryid
	 * @return
	 */
	public static boolean categoryHasItems(HttpServletRequest req,
			long categoryid) {

		Map reccount = (Map) req.getSession().getAttribute("PRODUCTCOUNT");
		if (reccount != null) {
			Integer count = (Integer) reccount.get(categoryid);
			if (count == null)
				return false;
			return true;
		} else {
			return false;
		}

	}



	/**
	 * returns a tree path for a given category from the lowest level
	 * 
	 * @param req
	 * @param categoriesid
	 * @return
	 */
	public static List getCategoryPath(String lang, int merchantId,
			long categoryid) {

		List returnlist = new ArrayList();

		try {

			CatalogService service = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			Map cat = service.getCategoriesByLang(merchantId, lang);

			if (cat == null || categoryid == 0) {
				return returnlist;
			}

			boolean atroot = false;
			long curcateg = categoryid;
			// while root category not reached
			while (!atroot) {
				Category categ = (Category) cat.get(curcateg);
				long parentcategid = categ.getParentId();
				returnlist.add(categ);
				curcateg = parentcategid;
				if (parentcategid == 0)
					atroot = true;
			}

			Collections.reverse(returnlist);

		} catch (Exception e) {
			log.error(e);
		}

		return returnlist;

	}

	public static Category getRootCategoryforCategory(String lang,
			int merchantId, long categoryid) {
		List path = getCategoryPath(lang, merchantId, categoryid);
		Category c = (Category) path.get(path.size() - 1);
		return c;
	}

	public static List getCategoriesForDropDownBox(int merchantId, String lang) {

		String usecache = conf.getString("core.cachecategoriiesstructure");
		if (usecache != null && usecache.equals("true")) {
			// use os cache
			CacheUtil cache = CacheUtil.getInstance();
			if (cache.containsCache("categoriiesdropdown")) {
				Map cachemap = cache.getCacheMap("categoriiesdropdown");
				List elements = (List) cachemap.get("categoriiesdropdown-"
						+ lang);
				if (elements != null) {
					return elements;
				}
			}
		}

		List returnlist = new ArrayList();

		try {

			CatalogService service = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			Collection categories = service
					.findCategoriesByMerchantIdAndLineageAndLanguageId(
							merchantId, CatalogConstants.LINEAGE_DELIMITER,
							lang);

			Map newClassification = new LinkedHashMap();

			List returnList = new LinkedList();

			if (categories != null && categories.size() > 0) {

				Iterator i = categories.iterator();

				Map classification = new LinkedHashMap();

				String lineage = "";
				int padding = 1;
				while (i.hasNext()) {

					Category c = (Category) i.next();

					// need parent id
					long parentId = c.getParentId();
					if (c.getCategoryId() == CatalogConstants.ROOT_CATEGORY_ID)
						continue;
					Map subCategoriesMap = (Map) classification.get(parentId);
					if (subCategoriesMap == null) {
						subCategoriesMap = new TreeMap();
						classification.put(parentId, subCategoriesMap);
					}

					CategoryPadding cpadding = new CategoryPadding();
					StringBuffer spaddingname = new StringBuffer();

					for (int pad = 0; pad < c.getDepth() - 1; pad++) {
						spaddingname.append("&nbsp;&nbsp;");
					}

					spaddingname.append(c.getName());
					cpadding.setName(spaddingname.toString());
					cpadding.setCategoryId(c.getCategoryId());
					subCategoriesMap.put(c.getCategoryId(), cpadding);

				}

				// re-order

				Iterator classificationIterator = classification.keySet()
						.iterator();
				while (classificationIterator.hasNext()) {
					Long parentId = (Long) classificationIterator.next();
					Map subCategoriesMap = (Map) classification.get(parentId);
					Iterator subCategoriesMapIterator = subCategoriesMap
							.keySet().iterator();
					while (subCategoriesMapIterator.hasNext()) {
						Long categoryId = (Long) subCategoriesMapIterator
								.next();

						CategoryPadding cpadding = (CategoryPadding) subCategoriesMap
								.get(categoryId);
						if (!newClassification.containsKey(cpadding
								.getCategoryId())) {
							newClassification.put(cpadding.getCategoryId(),
									cpadding);

							// get children
							newClassification = walkCategories(
									newClassification, classification, cpadding
											.getCategoryId());
						}

					}
				}

			}

			return new LinkedList(newClassification.values());

		} catch (Exception e) {
			log.error(e);
		}

		return returnlist;

	}

	private static Map walkCategories(Map returnMap, Map classification,
			long categoryId) {

		Map csMap = (Map) classification.get(categoryId);
		if (csMap != null) {
			Iterator csMapIterator = csMap.keySet().iterator();
			while (csMapIterator.hasNext()) {
				// get a subcategory
				Long csKey = (Long) csMapIterator.next();
				CategoryPadding cp = (CategoryPadding) csMap.get(csKey);
				returnMap.put(cp.getCategoryId(), cp);
				returnMap = walkCategories(returnMap, classification, cp
						.getCategoryId());
			}
		}

		return returnMap;

	}

}



```
