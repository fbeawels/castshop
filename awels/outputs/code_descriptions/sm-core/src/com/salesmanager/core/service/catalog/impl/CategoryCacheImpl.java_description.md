# CategoryCacheImpl.java

## Review

## 1. Summary  

`CategoryCacheImpl` is a **singleton cache holder** for product category data in an e‑commerce system.  
For each supported language it builds four nested maps:

| Map name | Purpose |
|----------|---------|
| `masterCategoriesMapByLang` | top‑level (“master”) categories (parentId = 0) |
| `categoriesMapByLang` | all categories |
| `subCategoriesMapByLang` | a map of parent→child maps (sub‑categories) |
| `genericCategoriesMapByLang` | categories that belong to the global merchant (`merchantId = 0`) |

The cache is populated once (`loadCategoriesInCache`) by retrieving all `CategoryDescription` objects via `CatalogService`, then converting them to `Category` objects and organizing them into the four structures.  
After loading, the data is stored in a generic `CacheModule` bean using pre‑defined keys.

Key design patterns / libraries  

* **Singleton** – lazy initialization, but not thread‑safe.  
* **BeanUtils** – used to copy properties between two `Category` instances.  
* **Apache Commons** – `BeanUtils`, `Collections.synchronizedMap`.  
* **Spring** – `SpringUtil` to look up the `CacheModule` bean.  
* **Custom Cache** – `CacheModule` with `putInCache` method.  

## 2. Detailed Description  

### Initialization & Loading  

* The constructor is private. It immediately calls `loadCategoriesInCache()` inside a try‑catch, ignoring any exception.  
* `loadCategoriesInCache()` first checks a static `loaded` flag; if `true`, it returns immediately.  
* It obtains a `CatalogService` instance via `ServiceFactory`.  
* It iterates over all languages retrieved from `RefCache.getLanguageswithindex()`.  
* For each language, it creates fresh `LinkedHashMap`s for the four category collections.  
* It calls `cservice.getAllCategoriesByLang(l.getLanguageId())` to obtain a list of `CategoryDescription`s.  
* For each description it:  
  1. Loads the base `Category` via `cservice.getCategory(...)`.  
  2. Sets the name from the description.  
  3. Creates a shallow copy of the category (`categ`) using `BeanUtils`.  
  4. Inserts the copy into `categoriesMap`.  
  5. Adds it to `masterCategoriesMap` if `parentId == 0`.  
  6. Adds it to `genericCategoriesMap` if `merchantId == 0`.  
  7. Adds it into `subCategoriesMap` keyed by the parent id.  
* After all categories for a language are processed, the four maps are cloned, wrapped with `synchronizedMap`, and stored in the language‑specific fields (`masterCategoriesMapByLang` etc.).  
* Finally, the four language‑specific maps are inserted into the `CacheModule` under a set of hard‑coded string keys. The global flag `loaded` is set to `true`.  

### Runtime & Access  

The public API only exposes getters for the master and sub‑category maps (the other two are commented out). Each getter lazily reloads the cache if the internal map is `null`.  

The rest of the system presumably retrieves category data from these maps by language and category id.  

### Assumptions & Constraints  

* Language data is static; `loadCategoriesInCache` runs only once unless the `loaded` flag is manually reset.  
* All category information is kept in memory for fast lookup – this may become memory‑heavy for large catalogs.  
* The code relies on third‑party `CatalogService` for database access; no error handling beyond logging is performed.  

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `private CategoryCacheImpl()` | Singleton constructor – loads cache | none | none | Calls `loadCategoriesInCache()` |
| `public static CategoryCacheImpl getInstance()` | Lazy singleton accessor | none | singleton instance | Instantiates the class on first call |
| `public synchronized void loadCategoriesInCache() throws Exception` | Builds all four maps from the database and puts them in the `CacheModule` | none | none | Sets static `loaded` flag; populates four `Map` fields; inserts into cache |
| `public Map getMasterCategoriesMapByLang()` | Returns master category map, reloading if necessary | none | `Map` | May trigger `loadCategoriesInCache()` |
| `public Map getSubCategoriesMapByLang()` | Returns sub‑category map, reloading if necessary | none | `Map` | May trigger `loadCategoriesInCache()` |

*Utility methods* – none beyond the getters.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.beanutils.BeanUtils` | Third‑party | Used to copy properties between two `Category` objects. |
| `org.apache.log4j.Logger` | Third‑party | Logging. |
| `com.salesmanager.core.constants.Constants` | Internal | Holds constants (`GLOBAL_MERCHANT_ID`, `CACHE_CATEGORIES`). |
| `com.salesmanager.core.entity.*` | Internal | JPA/POJO entities (`Category`, `CategoryDescription`, `MerchantStore`, `Language`). |
| `com.salesmanager.core.module.model.application.CacheModule` | Internal | Custom cache interface. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Factory for service objects. |
| `com.salesmanager.core.service.cache.RefCache` | Internal | Provides language list. |
| `com.salesmanager.core.service.catalog.CatalogService` | Internal | Data access for categories. |
| `com.salesmanager.core.util.SpringUtil` | Internal | Spring bean lookup for the cache module. |

No platform‑specific APIs; everything runs in a standard Java EE / Spring environment.

## 5. Additional Notes & Recommendations  

### 5.1. Code Quality / Readability  

* **Raw types** – All `Map`, `List`, `Iterator` usages are raw; generics should be applied (e.g., `Map<Integer, Map<Long, Category>>`).  
* **Hard‑coded cache keys** – The string keys contain typos (`masterCaategoriesMapByLang`, `caategoriesMapByLang`, `subCaategoriesMapByLang`, `genericCaategoriesMapByLang`). The last one even uses `subCategoriesMapByLang` instead of `genericCategoriesMapByLang`. These typos can silently corrupt the cache.  
* **Unnecessary BeanUtils** – `BeanUtils` is a utility class with static methods; creating a new instance is pointless. Additionally, the copy logic is fragile; a simple copy constructor or a dedicated mapper would be clearer.  
* **Exception handling** – The constructor swallows all exceptions. If cache loading fails, the instance remains unusable but the application may continue without knowledge of the failure.  
* **Synchronization** – `synchronizedMap` is used, but the outer maps (`masterCategoriesMapByLang`, etc.) are still accessed without external synchronization. The `synchronized` keyword on `loadCategoriesInCache` protects the loading logic, but not the getters.  

### 5.2. Thread‑Safety & Concurrency  

* The singleton is **not thread‑safe**. Two threads may concurrently create separate instances if they both hit `getInstance()` before the first one finishes construction.  
* The `loaded` flag is not volatile or protected by a lock; race conditions may allow multiple loads.  
* The lazy reload logic in getters also lacks proper synchronization, potentially causing duplicate loads.  

**Fix** – Use the **Initialization‑On‑Demand Holder** idiom, or make `instance` `volatile` and double‑check locking, or switch to an enum singleton.

### 5.3. Performance & Memory  

* All categories for **every language** are loaded into memory regardless of whether the application actually needs them. For large catalogs or many languages, this can exhaust heap space.  
* The nested maps duplicate category objects (`Category`) four times (master, all, sub, generic). This redundancy should be avoided.  
* The cache module is used only to store the already‑loaded maps; but the class also holds them in static fields, rendering the external cache redundant.  

**Fix** – Store only one canonical `Category` object per ID, then maintain lightweight indexes (e.g., parent‑id to list of child IDs) or use lazy lookup from a single map.

### 5.4. Extensibility  

* If a new cache entry (e.g., “featured categories”) is required, the current implementation forces adding another field and re‑implementing the loading logic.  
* The current API only exposes two of the four maps; other components cannot access master or generic categories directly.  

### 5.5. Suggested Refactor Outline  

1. **Define generics** – Replace raw types with parameterized collections.  
2. **Singleton improvement** – Use `private static class Holder { static final CategoryCacheImpl INSTANCE = new CategoryCacheImpl(); }`.  
3. **Centralized cache** – Remove static maps; let the `CacheModule` hold everything. Provide typed accessors that fetch data by language and id.  
4. **Single category source** – Load a single `Map<Long, Category>` per language; build other indexes lazily or on demand.  
5. **Error handling** – Propagate exceptions to the caller or throw a custom unchecked exception; avoid silent failure.  
6. **Bean mapping** – Replace `BeanUtils` copy with a dedicated constructor or `Category.copyFrom(...)`.  
7. **Cache key constants** – Define them in a separate constants class to avoid typos.  
8. **Unit tests** – Add tests for cache population, concurrency, and cache key correctness.  

### 5.6. Edge Cases Not Handled  

* **Circular parent references** – The code does not detect cycles; an infinite loop could occur if data is corrupted.  
* **Missing parent categories** – If a sub‑category refers to a non‑existent parent, it is simply ignored.  
* **Duplicate IDs across languages** – The map uses `categoryId` as key; if the same ID exists in different languages but refers to different entities, the language‑specific maps will override each other.  

## 6. Conclusion  

`CategoryCacheImpl` provides a basic in‑memory caching mechanism for category data, but its current implementation suffers from thread‑safety, redundancy, and maintainability issues. A refactor that embraces generics, proper singleton patterns, a single source of truth for categories, and clearer error handling would significantly improve robustness and performance.

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
package com.salesmanager.core.service.catalog.impl;

import java.util.Collections;
import java.util.HashMap;
import java.util.Iterator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;

import org.apache.commons.beanutils.BeanUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.catalog.Category;
import com.salesmanager.core.entity.catalog.CategoryDescription;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.module.model.application.CacheModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.SpringUtil;

public class CategoryCacheImpl {

	/*
	 * private static Map masterCategoriesMapByLang =
	 * Collections.synchronizedMap(new HashMap());//contains a map of <langid,
	 * map[categoryid, Category]> private static Map categoriesMapByLang =
	 * Collections.synchronizedMap(new HashMap());//contains a map of <langid,
	 * map[categoryid, Category]> private static Map subCategoriesMapByLang =
	 * Collections.synchronizedMap(new HashMap());//contains a map of <langid,
	 * map[categoryid, map-subcategories[categoryid, Category]]>
	 * 
	 * //categories of merchantId 0 private static Map
	 * genericCategoriesMapByLang = Collections.synchronizedMap(new
	 * HashMap());//contains a map of <langid, map[categoryid,
	 * map-subcategories[categoryid, Category]]>
	 */

	private Map masterCategoriesMapByLang = Collections
			.synchronizedMap(new HashMap());// contains a map of <langid,
											// map[categoryid, Category]>
	private Map categoriesMapByLang = Collections
			.synchronizedMap(new HashMap());// contains a map of <langid,
											// map[categoryid, Category]>
	private Map subCategoriesMapByLang = Collections
			.synchronizedMap(new HashMap());// contains a map of <langid,
											// map[categoryid,
											// map-subcategories[categoryid,
											// Category]]>

	// categories of merchantId 0
	private Map genericCategoriesMapByLang = Collections
			.synchronizedMap(new HashMap());// contains a map of <langid,
											// map[categoryid,
											// map-subcategories[categoryid,
											// Category]]>

	private static Logger log = Logger.getLogger(CategoryCacheImpl.class);
	private static boolean loaded = false;

	private static CategoryCacheImpl instance = null;

	private CategoryCacheImpl() {
		try {
			loadCategoriesInCache();
		} catch (Exception e) {
			log.error(e);
		}

	}

	public static CategoryCacheImpl getInstance() {
		if (instance == null) {
			instance = new CategoryCacheImpl();
		}
		return instance;
	}

	public synchronized void loadCategoriesInCache() throws Exception {

		if (loaded) {
			return;
		}

		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);

		Iterator langs = RefCache.getLanguageswithindex().entrySet().iterator();

		Map categoriesMap = null;
		Map masterCategoriesMap = null;
		Map subCategoriesMap = null;

		Map genericCategoriesMap = null;

		while (langs.hasNext()) {

			categoriesMap = new LinkedHashMap();
			masterCategoriesMap = new LinkedHashMap();
			subCategoriesMap = new LinkedHashMap();

			genericCategoriesMap = new LinkedHashMap();

			Entry e = (Entry) langs.next();

			Language l = (Language) e.getValue();

			// Get all description
			List catdesc =

			cservice.getAllCategoriesByLang(l.getLanguageId());

			Iterator allcategsit = catdesc.iterator();
			while (allcategsit.hasNext()) {

				CategoryDescription desc = (CategoryDescription) allcategsit
						.next();

				Category cat = cservice.getCategory(desc.getId()
						.getCategoryId());

				if (cat != null) {

					cat.setName(desc.getCategoryName());

					Category categ = new Category();

					try {
						BeanUtils bu = new BeanUtils();
						bu.copyProperties(categ, cat);
					} catch (Exception ie) {
						log.error(ie);
					}

					if (!categoriesMap.containsKey(categ.getCategoryId())) {
						categoriesMap.put(categ.getCategoryId(), categ);
					}

					if (categ.getParentId() == 0) {// this is a master category
						masterCategoriesMap.put(categ.getCategoryId(), categ);
					}

					if (categ.getMerchantId() == Constants.GLOBAL_MERCHANT_ID) {
						if (!genericCategoriesMap.containsKey(categ
								.getCategoryId())) {
							genericCategoriesMap.put(categ.getCategoryId(),
									categ);
						}
					}

					// populate sub categor

					long supint = categ.getParentId();
					if (supint == 0) {
						continue;
					}
					if (!subCategoriesMap.containsKey(supint)) {
						Map submap = Collections
								.synchronizedMap(new LinkedHashMap());
						submap.put(categ.getCategoryId(), categ);
						subCategoriesMap.put(supint, submap);
					} else {
						Map submap = (Map) subCategoriesMap.get(supint);
						submap.put(categ.getCategoryId(), categ);
					}

				}

			}

			Map masters = ((Map) ((LinkedHashMap) masterCategoriesMap).clone());
			Map categs = ((Map) ((LinkedHashMap) categoriesMap).clone());
			Map subs = ((Map) ((LinkedHashMap) subCategoriesMap).clone());
			Map gen = ((Map) ((LinkedHashMap) genericCategoriesMap).clone());

			masterCategoriesMapByLang.put(l.getLanguageId(), Collections
					.synchronizedMap(masters));

			categoriesMapByLang.put(l.getLanguageId(), Collections
					.synchronizedMap(categs));

			subCategoriesMapByLang.put(l.getLanguageId(), Collections
					.synchronizedMap(subs));

			genericCategoriesMapByLang.put(l.getLanguageId(), Collections
					.synchronizedMap(gen));

		}

		CacheModule module = (CacheModule) SpringUtil.getBean("cache");

		// simulate a store
		MerchantStore store = new MerchantStore();
		store.setUseCache(true);

		module.putInCache("masterCaategoriesMapByLang",
				masterCategoriesMapByLang, Constants.CACHE_CATEGORIES, store);
		module.putInCache("caategoriesMapByLang", categoriesMapByLang,
				Constants.CACHE_CATEGORIES, store);
		module.putInCache("subCaategoriesMapByLang", subCategoriesMapByLang,
				Constants.CACHE_CATEGORIES, store);
		module.putInCache("genericCaategoriesMapByLang", subCategoriesMapByLang,
				Constants.CACHE_CATEGORIES, store);

		loaded = true;

	}

	/*
	 * private static void setCategoriesMapByLang(Map categoriesMapByLang) {
	 * CategoryCacheImpl.categoriesMapByLang = categoriesMapByLang; } private
	 * static void setMasterCategoriesMapByLang(Map masterCategoriesMapByLang) {
	 * CategoryCacheImpl.masterCategoriesMapByLang = masterCategoriesMapByLang;
	 * }
	 * 
	 * private static void setSubCategoriesMapByLang(Map subCategoriesMapByLang)
	 * { CategoryCacheImpl.subCategoriesMapByLang = subCategoriesMapByLang; }
	 * 
	 * private static void setGenericCategoriesMapByLang(Map
	 * genericCategoriesMapByLang) {
	 * CategoryCacheImpl.genericCategoriesMapByLang =
	 * genericCategoriesMapByLang; }
	 */

	/**
	 * Exposed getters
	 * 
	 * @return
	 */

	/*
	 * public Map getCategoriesMapByLang() {
	 * 
	 * 
	 * 
	 * if(categoriesMapByLang==null) { try { loaded = false;
	 * instance.getInstance().loadCategoriesInCache(); } catch (Exception e) {
	 * log.error(e); }
	 * 
	 * } return categoriesMapByLang; }
	 */

	public Map getMasterCategoriesMapByLang() {
		if (masterCategoriesMapByLang == null) {
			try {
				loaded = false;
				instance.getInstance().loadCategoriesInCache();
			} catch (Exception e) {
				log.error(e);
			}

		}
		return masterCategoriesMapByLang;
	}

	public Map getSubCategoriesMapByLang() {
		if (subCategoriesMapByLang == null) {
			try {
				loaded = false;
				instance.getInstance().loadCategoriesInCache();
			} catch (Exception e) {
				log.error(e);
			}

		}
		return subCategoriesMapByLang;
	}

	/*
	 * public static Map getGenericCategoriesMapByLang() {
	 * if(genericCategoriesMapByLang==null) { try { loaded = false;
	 * instance.getInstance().loadCategoriesInCache(); } catch (Exception e) {
	 * log.error(e); }
	 * 
	 * } return genericCategoriesMapByLang; }
	 */

}



```
