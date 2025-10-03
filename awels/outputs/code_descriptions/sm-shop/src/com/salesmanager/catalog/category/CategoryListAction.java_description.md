# CategoryListAction.java

## Review

## 1. Summary

**Purpose & Functionality**  
`CategoryListAction` is a Struts‑based action that renders a product listing page for a given category in a multi‑merchant e‑commerce platform.  
It:

1. Resolves the target category either from the HTTP session or by looking up the SEO URL.  
2. Retrieves the category description, sets SEO meta‑data and page titles.  
3. Builds the category hierarchy for the side‑bar navigation.  
4. Loads a paginated list of products belonging to the selected category (and its sub‑categories).  
5. Caches category lists, category paths, and category‑specific configuration to reduce database load.  

**Key Components**

| Component | Role |
|-----------|------|
| `PageBaseAction` | Base class providing paging, locale handling and template‑related helpers. |
| `CatalogService` | DAO/service layer that fetches categories & products. |
| `CacheModule` | In‑memory or distributed cache for category lists & paths. |
| `SessionUtil` / `SpringUtil` | Helpers to obtain the `MerchantStore` and Spring beans. |
| `SearchProductCriteria / Response` | DTOs for paginated product queries. |

**Design Patterns / Libraries**

* **Service Locator** – `ServiceFactory.getService()` to obtain the catalog service.  
* **Cache‑Aside** – The action checks the cache first and falls back to DB if not present.  
* **MVC (Struts)** – The action populates a view model (`categories`, `products`, `categoryPath`, etc.) for the JSP layer.  
* **Utility/Helper classes** – `LanguageUtil`, `LocaleUtil`, `CategoryUtil`, `PropertiesHelper`.  
* **Apache Commons** – `StringUtils` from XWork (deprecated) and `Configuration`.  
* **Log4j** – Logging.  

---

## 2. Detailed Description

### Execution Flow

1. **`page()`** – Entry point when the request hits `/categorylist`.  
   * Attempts to read a `currentCategory` from the session.  
   * If found, uses it to populate page metadata and calls `setCategories()` to load products.  
   * If not found, falls back to `displayCategory()`, which resolves the category from the SEO URL.

2. **`displayCategory()`** – Handles the “canonical” request path where the category is identified by its SEO URL.  
   * Sets the current category in the session (`mainUrl` for top‑level categories, `subCategory` for children).  
   * Calls `setCategories()` to fetch the paginated product list.

3. **`setCategories(Category c, int startIndex)`** – Core method that:
   * Builds the lineage string and stores it in `categoryLineage`.  
   * Looks up a cached `CategoryList` (containing sub‑categories and their IDs).  
   * If missing, queries the database via `CatalogService.findCategoriesByMerchantIdAndLineageAndLanguageId`.  
   * Stores the result (or a “missed” flag) back into the cache.  
   * Builds a `SearchProductCriteria` with the list of category IDs and the requested page size & offset.  
   * Executes `findProductsByCategoryList()` to obtain `SearchProductResponse`.  
   * Populates `products` and `categoryPath`, the latter also cached similarly.  
   * Sets paging information on the `PageBaseAction` (total count, real count, etc.).

4. **`getProductCount()`** – Reads the configured product‑per‑page limit from the store configuration (session attribute `STORECONFIGURATION`) or falls back to the static `size`.  
   * Updates the static `size` field to the resolved value.  

5. **Utility methods** – Getters/setters for fields exposed to the JSP layer.

### Dependencies & Interaction

| Layer | External Service | Notes |
|-------|------------------|-------|
| Web | Struts action (`PageBaseAction`) | Handles request/response plumbing. |
| Business | `CatalogService` | CRUD for categories/products. |
| Persistence | JPA/Hibernate (inside `CatalogService`) | Not visible here. |
| Cache | `CacheModule` | Provides get/put with store key. |
| Session | `SessionUtil` | Retrieves current `MerchantStore`. |
| Locale | `LocaleUtil` | Converts entities to the current locale. |
| Configuration | `PropertiesHelper` | Reads `catalog.categorylist.maxsize`. |

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side Effects |
|--------|---------|------------|---------|--------------|
| `String page()` | Default Struts action entry point. | None | `"SUCCESS"` | Populates page metadata, sets categories & products in the action. |
| `String displayCategory()` | Handles SEO‑URL based category lookup. | None | `"SUCCESS"` | Loads category, sets session attributes, sets categories & products. |
| `void setCategories(Category c, int startIndex)` | Core logic to build sub‑category list, cache handling, and product pagination. | `Category c`, `int startIndex` | None | Populates `categories`, `products`, `categoryPath`; updates paging state. |
| `int getProductCount()` | Reads per‑page product count from store config. | None | `int` | Updates static `size` field. |
| Getters/Setters | Standard JavaBean accessors for fields used by the JSP. | N/A | N/A | None. |

---

## 4. Dependencies

| Library | Type | Notes |
|---------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads properties file. |
| `org.apache.commons.lang.xwork.StringUtils` | Third‑party (deprecated) | Used for `isBlank()`. Replace with `org.apache.commons.lang3.StringUtils`. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.*` | In‑house | Domain entities (`Category`, `Product`, …), services, constants, util classes. |
| `javax.servlet.http.HttpSession` | Standard | Session storage. |
| `java.util.*` | Standard | Collections, Locale, Map. |

---

## 5. Additional Notes & Recommendations

### 5.1 Code‑Quality Issues

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (e.g., `Collection categoryPath`, `ArrayList categories`) | Compile‑time warnings, potential `ClassCastException`. | Use generics: `Collection<Category> categoryPath;` `List<Category> categories = new ArrayList<>();`. |
| **Deprecated `StringUtils`** | Future compatibility, missing newer features. | Replace with `org.apache.commons.lang3.StringUtils`. |
| **Static mutable `size`** | Thread‑safety risk in a multi‑threaded servlet container. | Make `size` thread‑local or remove static; store per‑request value. |
| **Unnecessary `super` calls** (`super.setSize(...)`) | Confusing readability; could be replaced with `this.setSize(...)`. | Use instance methods directly. |
| **Null‑pointer risks** | e.g., `c.getCategoryDescription()` if `c` is null; `cache.getFromCache` returns null but used without check. | Add null checks, fallback logic, or guard clauses. |
| **Hard‑coded cache keys** | Potential collision or mis‑caching across languages or merchants. | Build keys with merchant ID and language to avoid cross‑store leaks. |
| **Duplicate `categoryList` creation** | Logic can be simplified; `categoryList` can be reused. | Refactor to avoid duplicated code blocks. |
| **Missing exception handling** (`catch (Exception e) { logger.error(e); }`) | Swallows stack trace details, may hide bugs. | Log with `logger.error("...", e)` or rethrow as runtime. |
| **No pagination bounds check** | Start index could be negative or beyond total count. | Validate `startIndex >= 0` and clamp to valid range. |
| **No transaction handling** | Possible data inconsistency if the database state changes mid‑request. | Ensure `CatalogService` methods are transactional. |
| **Potential memory leak** | Caching large lists of categories/products in session/bean. | Use weak references or evict old entries; limit cache size. |

### 5.2 Performance & Scalability

* **Cache Usage** – The cache is consulted for both category lists and paths. However, the “missed” flag logic is duplicated and may lead to stale data if underlying DB changes. Consider using a proper cache‑aside library (e.g., Ehcache, Hazelcast) with TTL.  
* **Lazy Loading** – The product list is fetched per request; if the same category is viewed frequently, caching the product list could reduce DB load.  
* **Pagination** – The logic sets `size` based on configuration but still pulls all matching products into memory before paginating. If the result set is large, consider delegating pagination to the DAO (`findProductsByCategoryList` already accepts quantity & start index). Ensure it uses `LIMIT/OFFSET` in the query.  

### 5.3 Security

* **SQL Injection** – The code uses service layer methods; ensure those methods use parameterized queries.  
* **Session fixation** – Category objects are stored in the session (`mainUrl`, `subCategory`). Ensure these are cleaned up when the user navigates away.  

### 5.4 Future Enhancements

1. **Refactor to MVC Framework** – Replace Struts with Spring MVC or JSF to benefit from dependency injection and cleaner controllers.  
2. **DTOs for View** – Instead of exposing entity objects directly to the JSP, create lightweight DTOs containing only the needed fields.  
3. **Unit Tests** – Add JUnit tests for `CategoryListAction` using mocks for `CatalogService` and `CacheModule`.  
4. **Internationalization** – Store `categoryLineage` and cache keys in a language‑agnostic format; avoid concatenating language codes directly.  
5. **Asynchronous Loading** – Load sub‑category navigation asynchronously (AJAX) to improve initial page load.  

---

**Overall Assessment**  
The action fulfills its core responsibilities and demonstrates a solid integration with the underlying service and caching layers. However, several code‑quality and maintainability concerns (raw types, deprecated libraries, thread‑safety of static fields, duplicated logic) should be addressed to improve robustness and future‑proofing of the module. With the suggested refactors and modernization, the component would be more testable, maintainable, and performant.

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

package com.salesmanager.catalog.category;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.xwork.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.common.PageBaseAction;
import com.salesmanager.common.util.PropertiesHelper;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.catalog.Category;
import com.salesmanager.core.entity.catalog.CategoryDescription;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.SearchProductCriteria;
import com.salesmanager.core.entity.catalog.SearchProductResponse;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.module.model.application.CacheModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.CategoryUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

/**
 * Display product listing pages
 * 
 * @author Carl Samson
 * 
 */
public class CategoryListAction extends PageBaseAction {

	private static final long serialVersionUID = 3928621995996114942L;
	private static Logger logger = Logger.getLogger(CategoryListAction.class);
	private static Configuration config = PropertiesHelper.getConfiguration();

	private int merchantId;
	private long categoryId;

	private String currentEntity;
	private String categoryLineage;

	private static int size = 9;

	static {
		size = config.getInt("catalog.categorylist.maxsize", 9);
	}

	public String getCategoryLineage() {
		return categoryLineage;
	}

	public void setCategoryLineage(String categoryLineage) {
		this.categoryLineage = categoryLineage;
	}

	public String getCurrentEntity() {
		return currentEntity;
	}

	public void setCurrentEntity(String currentEntity) {
		this.currentEntity = currentEntity;
	}

	private Category category;

	private Collection categoryPath;

	private Collection<Product> products;
	private Collection<Category> categories = new ArrayList();

	public Collection<Category> getCategories() {
		return categories;
	}

	public void setCategories(Collection<Category> categories) {
		this.categories = categories;
	}

	public Collection<Product> getProducts() {
		return products;
	}

	public void setProducts(Collection<Product> products) {
		this.products = products;
	}

	public String page() {

		try {

			Category c = (Category) super.getServletRequest().getSession()
					.getAttribute("currentCategory");

			super.setSize(getProductCount());// defined in configuration
			// according to template
			super.setPageStartNumber();

			if (c != null) {

				CategoryDescription description = c.getCategoryDescription();

				this.setCategory(c);

				this.setMetaDescription(description.getMetatagDescription());
				this.setMetaKeywords(description.getMetatagKeywords());
				
				if(!StringUtils.isBlank(description.getCategoryTitle())) {
					this.setPageTitle(description.getCategoryTitle());
				} else {
					this.setPageTitle(description.getCategoryName());
				}
				this.setPageText(description.getCategoryDescription());

				this.setCategories(c, this.getPageStartIndex());

			} else {
				super.setRequestedEntityId(this.getCurrentEntity());
				this.displayCategory();
			}

		} catch (Exception e) {
			logger.error(e);
		}

		return SUCCESS;

	}

	public String displayCategory() {

		try {

			super.setSize(getProductCount());// defined in configuration
			// according to template
			super.setPageStartNumber();

			// 1) Get Category

			String url = super.getRequestedEntityId();
			this.setCurrentEntity(url);
			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			Locale locale = (Locale) super.getLocale();

			// make a query to retrieve a category by id or seurl
			Category c = cservice.getCategoryByMerchantIdAndSeoURLAndByLang(
					store.getMerchantId(), url, locale.getLanguage());

			if (c != null) {

				CategoryDescription description = c.getCategoryDescription();

				this.setCategory(c);

				this.setMetaDescription(description.getMetatagDescription());
				this.setMetaKeywords(description.getMetatagKeywords());
				if(!StringUtils.isBlank(description.getCategoryTitle())) {
					this.setPageTitle(description.getCategoryTitle());
				} else {
					this.setPageTitle(description.getCategoryName());
				}
				this.setPageText(description.getCategoryDescription());

				// SET CURRENT MAIN CATEGORY AND SUB CATEGORY IN HTTP SESSION
				if (c.getParentId() == 0) {
					// will be used for top category display
					super.getServletRequest().getSession().setAttribute(
							"mainUrl", c);
				} else {
					// will be used for side bar navigation categories
					super.getServletRequest().getSession().setAttribute(
							"subCategory", c);
				}

				super.getServletRequest().getSession().setAttribute(
						"currentCategory", c);

			}

			this.setCategories(c, this.getPageStartIndex());

		} catch (Exception e) {
			logger.error(e);
		}

		return SUCCESS;

	}

	private void setCategories(Category c, int startIndex) throws Exception {

		MerchantStore store = SessionUtil.getMerchantStore(super
				.getServletRequest());
		CacheModule cache = (CacheModule) SpringUtil.getBean("cache");
		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);

		// get store template maximum item quantity per page
		List idList = new ArrayList();

		// Get category list for left menu
		String lineageQuery = new StringBuffer().append(c.getLineage()).append(
				c.getCategoryId()).append(CatalogConstants.LINEAGE_DELIMITER)
				.toString();
		this.setCategoryLineage(lineageQuery);

		// Category List from cache
		CategoryList categoryList = null;
		try {

			categoryList = (CategoryList) cache.getFromCache(
					Constants.CACHE_CATEGORIES + lineageQuery + "_"
							+ super.getLocale().getLanguage(), store);
		} catch (Exception ignore) {

		}

		if (categoryList == null) {

			// get from missed
			boolean missed = false;
			try {
				missed = (Boolean) cache.getFromCache(
						Constants.CACHE_CATEGORIES + lineageQuery + "_MISSED_"
								+ super.getLocale().getLanguage(), store);
			} catch (Exception ignore) {

			}

			if (!missed) {

				Collection subcategs = cservice
						.findCategoriesByMerchantIdAndLineageAndLanguageId(c
								.getMerchantId(), lineageQuery, super
								.getLocale().getLanguage());

				Collection ids = new ArrayList();
				if (subcategs != null && subcategs.size() > 0) {
					categoryList = new CategoryList();
					Iterator cIterator = subcategs.iterator();
					while (cIterator.hasNext()) {
						Category sc = (Category) cIterator.next();
						categories.add(sc);
						idList.add(sc.getCategoryId());
					}

					Collection categs = new ArrayList();
					categs.addAll(categories);
					categoryList.setCategories(categs);

				}

				// add master category
				idList.add(c.getCategoryId());

				if (subcategs != null && subcategs.size() > 0) {
					ids.addAll(idList);
					categoryList.setCategoryIds(ids);
				}

				if (categoryList != null) {

					try {
						cache
								.putInCache(Constants.CACHE_CATEGORIES
										+ lineageQuery + "_"
										+ super.getLocale().getLanguage(),
										categoryList,
										Constants.CACHE_CATEGORIES, store);
					} catch (Exception e) {
						logger.error(e);
					}

				} else {

					try {
						cache
								.putInCache(Constants.CACHE_CATEGORIES
										+ lineageQuery + "_MISSED_"
										+ super.getLocale().getLanguage(),
										categoryList,
										Constants.CACHE_CATEGORIES, store);
					} catch (Exception e) {
						logger.error(e);
					}

				}

			}

		} else {
			
			
			idList.add(c.getCategoryId());
			idList.addAll(categoryList.getCategoryIds());
			
		}

		int productCount = getProductCount();

		// get product list
		SearchProductCriteria criteria = new SearchProductCriteria();
		criteria.setMerchantId(store.getMerchantId());
		criteria.setCategoryList(idList);
		criteria.setLanguageId(LanguageUtil.getLanguageNumberCode(super
				.getLocale().getLanguage()));
		criteria.setQuantity(productCount);// qty based on template config
		criteria.setStartindex(startIndex);

		SearchProductResponse response = cservice
				.findProductsByCategoryList(criteria);

		this.setListingCount(response.getCount());
		Collection prds = response.getProducts();

		LocaleUtil.setLocaleToEntityCollection(prds, super.getLocale(), store
				.getCurrency());

		// get category path
		try {
			categoryPath = (Collection) cache.getFromCache(
					Constants.CACHE_CATEGORIES_PATH + "_" + c.getCategoryId()
							+ "_" + super.getLocale(), store);
		} catch (Exception ignore) {

		}

		if (categoryPath == null || categoryPath.size() == 0) {

			// get from missed
			boolean missed = false;
			try {
				missed = (Boolean) cache.getFromCache(
						Constants.CACHE_CATEGORIES_PATH + "_MISSED_"
								+ c.getCategoryId() + "_" + super.getLocale(),
						store);
			} catch (Exception ignore) {

			}

			if (!missed) {

				categoryPath = CategoryUtil.getCategoryPath(super.getLocale()
						.getLanguage(), store.getMerchantId(), c
						.getCategoryId());

				if (categoryPath != null && categoryPath.size() > 0) {

					try {
						cache
								.putInCache(Constants.CACHE_CATEGORIES_PATH
										+ "_" + c.getCategoryId() + "_"
										+ super.getLocale(), categoryPath,
										Constants.CACHE_CATEGORIES, store);
					} catch (Exception e) {
						logger.error(e);
					}

				} else {

					try {
						cache.putInCache(Constants.CACHE_CATEGORIES_PATH
								+ "_MISSED_" + c.getCategoryId() + "_"
								+ super.getLocale(), true,
								Constants.CACHE_CATEGORIES, store);
					} catch (Exception e) {
						logger.error(e);
					}
				}

			}

		}

		categoryPath = CategoryUtil.getCategoryPath(super.getLocale()
				.getLanguage(), store.getMerchantId(), c.getCategoryId());

		products = prds;

		super.setListingCount(response.getCount());
		super.setRealCount(products.size());
		super.setPageElements();

		/*
		 * if(products==null || products.size()==0) { this.setFirstItem(0);
		 * this.setLastItem(response.getCount()); } else {
		 * 
		 * this.setFirstItem(startIndex+1); if(productCount<response.getCount())
		 * { this.setLastItem(startIndex + products.size()); } else {
		 * this.setLastItem(response.getCount()); } }
		 */

	}

	private int getProductCount() {

		int maxQuantity = size;
		MerchantStore store = (MerchantStore) super.getServletRequest()
				.getSession().getAttribute("STORE");
		Map storeConfiguration = (Map) super.getServletRequest().getSession()
				.getAttribute("STORECONFIGURATION");
		if (storeConfiguration != null) {
			String sMaxQuantity = null;
			try {
				sMaxQuantity = (String) storeConfiguration
						.get("listingitemsquantity");
				if (sMaxQuantity != null) {
					maxQuantity = Integer.parseInt(sMaxQuantity);

				}
			} catch (Exception e) {
				logger
						.warn("Invalid value for listing quantity (table modulee_configuration.configurationKey listingitemsquantity has value "
								+ sMaxQuantity
								+ " for modulee_configuration.configuration_module "
								+ store.getTemplateModule());
			}
		}
		size = maxQuantity;
		return maxQuantity;

	}

	public int getMerchantId() {
		return merchantId;
	}

	public void setMerchantId(int merchantId) {
		this.merchantId = merchantId;
	}

	public long getCategoryId() {
		return categoryId;
	}

	public void setCategoryId(long categoryId) {
		this.categoryId = categoryId;
	}

	public Collection getCategoryPath() {
		return categoryPath;
	}

	public void setCategoryPath(Collection categoryPath) {
		this.categoryPath = categoryPath;
	}

	public Category getCategory() {
		return category;
	}

	public void setCategory(Category category) {
		this.category = category;
	}

}



```
