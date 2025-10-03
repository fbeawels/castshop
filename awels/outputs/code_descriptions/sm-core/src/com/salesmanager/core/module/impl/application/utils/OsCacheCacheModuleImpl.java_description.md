# OsCacheCacheModuleImpl.java

## Review

## 1. Summary

| Aspect | Details |
|--------|---------|
| **Purpose** | Provides a thin, application‑specific wrapper around **OSCache** (`GeneralCacheAdministrator`) to store and retrieve objects per `MerchantStore`. |
| **Key components** | * `OsCacheCacheModuleImpl` – Singleton cache module implementing the `CacheModule` interface.<br>* `GeneralCacheAdministrator` – OSCache's core cache engine.<br>* Methods: `flushAll`, `flushEntry`, `flushCacheGroup`, `getFromCache`, `putInCache`. |
| **Design patterns / frameworks** | • **Singleton** – enforced via a private static instance and `getInstance()`.<br>• **Facade** – hides OSCache internals behind a simpler API.<br>• Uses OSCache (third‑party) and a custom `CacheModuleException` wrapper. |

---

## 2. Detailed Description

### Overall flow

1. **Instantiation**  
   The first call to `OsCacheCacheModuleImpl.getInstance()` creates a singleton instance and initializes a static `GeneralCacheAdministrator`.

2. **Cache operations**  
   - **Retrieval** (`getFromCache`) builds a key by appending the merchant ID, attempts to read the object with a *hard‑coded* refresh period of 2000 s, and handles `NeedsRefreshException`/other failures by cancelling the update and propagating a `CacheModuleException`.  
   - **Insertion** (`putInCache`) again appends the merchant ID, attempts to store the object under the supplied group, and ensures `cancelUpdate` is called if the operation fails.  
   - **Flushing** – `flushAll`, `flushEntry`, `flushCacheGroup` delegate directly to the OSCache API.  The overload that accepts a `MerchantStore` simply ignores the store argument.

3. **Shutdown** – No explicit cleanup; the static `admin` lives for the JVM’s lifetime.

### Assumptions & constraints

| Item | Detail |
|------|--------|
| **Thread safety** | The singleton creation is *not* thread‑safe; concurrent calls could create multiple instances or re‑initialize the cache admin. |
| **Cache key** | Keys are concatenated with the merchant ID without any separator or sanitisation.  This could lead to key collisions or accidental sharing of cached entries across stores. |
| **Refresh period** | Hard‑coded to 2000 s; no configuration hook. |
| **Store usage** | The `store.isUseCache()` guard is present, but the `flushCacheGroup(String group, MerchantStore store)` method ignores the store and flushes the group for all merchants. |
| **Error handling** | Exceptions from OSCache are wrapped in `CacheModuleException`, but the actual error messages are largely lost, and there’s no logging. |
| **Dependencies** | Relies on OSCache (`GeneralCacheAdministrator`) and a custom exception class. |

---

## 3. Functions / Methods

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `getInstance()` | Thread‑unsafe Singleton factory. | None | `OsCacheCacheModuleImpl` | Instantiates `admin` if not yet created. |
| `flushAll()` | Clears the entire cache. | None | void | Calls `admin.flushAll()`. |
| `flushEntry(String key)` | Removes a single cache entry. | `key` | void | `admin.flushEntry(key)`. |
| `flushCacheGroup(String group, MerchantStore store)` | Intended to flush a group for a particular store (currently ignored). | `group`, `store` | void | `admin.flushGroup(group)`. |
| `flushCacheGroup(String group)` | Flushes a group for all stores. | `group` | void | `admin.flushGroup(group)`. |
| `getFromCache(String key, MerchantStore store)` | Retrieves an object from cache if the store allows caching. | `key`, `store` | `Object` | Builds key, calls `admin.getFromCache(key, 2000)`. Handles `NeedsRefreshException` and other `Exception`. |
| `putInCache(String key, Object content, String group, MerchantStore store)` | Inserts an object into cache under a group if caching is enabled. | `key`, `content`, `group`, `store` | void | Builds key, calls `admin.putInCache(key, content, new String[]{group})`. |

*Reusable utilities*: The key‑concatenation logic appears in two methods; it could be extracted to a protected helper.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `com.opensymphony.oscache.general.GeneralCacheAdministrator` | Third‑party (OSCache) | Core caching engine. |
| `com.salesmanager.core.module.model.application.CacheModule` | Internal interface | Contract for cache modules. |
| `com.salesmanager.core.module.impl.application.CacheModuleException` | Internal exception | Wraps OSCache errors. |
| `com.salesmanager.core.entity.merchant.MerchantStore` | Internal entity | Provides merchant ID and caching flag. |
| `com.opensymphony.oscache.base.NeedsRefreshException` | OSCache exception | Indicates a cache entry needs refresh. |

No external build‑tool or platform‑specific dependencies beyond OSCache.

---

## 5. Additional Notes & Recommendations

### Thread‑safety

* The `getInstance()` method is not synchronized; multiple threads could create separate instances or re‑initialize the static `admin`.  
  * **Fix**: Use double‑checked locking with `volatile`, or better, the **Bill‑Pugh Singleton** (static inner helper) or an enum singleton.

### Key construction

* Concatenating the merchant ID directly (`key + store.getMerchantId()`) risks key collisions (e.g., `"abc1"` vs `"abc" + "1"`).  
  * **Fix**: Use a separator (`":"` or `"_"`) and normalise the key (`String.format("%s:%d", key, store.getMerchantId())`).  
  * Consider URL‑encoding or hashing if keys may contain special characters.

### Refresh period

* Hard‑coded 2000 seconds is opaque and cannot be tuned per application.  
  * **Fix**: Expose a configuration property (e.g., `cache.refreshPeriod`) or accept it as a constructor parameter.

### Store‑specific flushing

* The overload `flushCacheGroup(String group, MerchantStore store)` currently ignores `store`.  
  * **Fix**: Either remove the parameter or implement per‑store flushing by prefixing group names with the merchant ID.

### Error handling & logging

* Exceptions are caught and wrapped, but no message or stack trace is logged.  
  * **Fix**: Add a logger (`org.slf4j.Logger`) and log at WARN/ERROR level before rethrowing.

### Exception propagation

* In `getFromCache`, if a `NeedsRefreshException` occurs, the method silently returns `null`.  
  * **Fix**: Decide whether to rethrow a custom exception or return a sentinel value; at least document this behaviour.

### Code cleanliness

* The `try`/`catch` blocks in `putInCache` swallow the caught exception (`nre`) and do not log it.  
  * **Fix**: Log the exception and, if appropriate, rethrow as `CacheModuleException`.

### API consistency

* All cache‑modifying methods (`flushAll`, `flushEntry`, `flushCacheGroup`, `putInCache`) should be `synchronized` if the underlying OSCache is not thread‑safe.  
  * **Fix**: Add `synchronized` or rely on OSCache’s internal thread safety guarantees.

### Potential enhancements

| Idea | Rationale |
|------|-----------|
| **Cache metrics** | Provide methods to query cache hit/miss statistics. |
| **Cache eviction policy** | Allow configuration of LRU, LFU, etc. via OSCache settings. |
| **Cache grouping by merchant** | Store group names prefixed with merchant ID to avoid accidental cross‑store contamination. |
| **Cache warm‑up** | Provide a callback to pre‑populate the cache on startup. |
| **Unit tests** | Add tests for key construction, concurrency, and exception paths. |

---

### Bottom line

The wrapper gives a simple API for OSCache, but it currently suffers from several design and robustness issues:

1. **Thread‑unsafe singleton** – may lead to race conditions in multi‑threaded environments.  
2. **Fragile key handling** – risks collisions and accidental cache sharing.  
3. **Hard‑coded configuration** – limits flexibility.  
4. **Incomplete store awareness** – group flushing ignores store context.  
5. **Sparse logging** – hard to diagnose cache failures.  

Addressing these points will make the module more reliable, maintainable, and adaptable to future requirements.

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
package com.salesmanager.core.module.impl.application.utils;

import com.opensymphony.oscache.base.NeedsRefreshException;
import com.opensymphony.oscache.general.GeneralCacheAdministrator;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.module.impl.application.CacheModuleException;
import com.salesmanager.core.module.model.application.CacheModule;

/**
 * OSCache implementation wrapper
 * 
 * @author Carl Samson
 * 
 */
public class OsCacheCacheModuleImpl implements CacheModule {

	private static GeneralCacheAdministrator admin = null;
	private static OsCacheCacheModuleImpl instance = null;

	private OsCacheCacheModuleImpl() {

	}

	public static OsCacheCacheModuleImpl getInstance() {
		if (instance == null) {
			instance = new OsCacheCacheModuleImpl();
			admin = new GeneralCacheAdministrator();
		}

		return instance;
	}

	public void flushAll() throws CacheModuleException {
		admin.flushAll();
	}

	public void flushEntry(String key) throws CacheModuleException {
		admin.flushEntry(key);
	}

	public void flushCacheGroup(String group, MerchantStore store)
			throws CacheModuleException {
		// store is ignored for now
		admin.flushGroup(group);
	}

	public void flushCacheGroup(String group) throws CacheModuleException {
		admin.flushGroup(group);
	}

	public Object getFromCache(String key, MerchantStore store)
			throws CacheModuleException {

		// @todo, refreshPeriod from configuration file
		// property to cache or not //2000 seconds

		Object o = null;

		key = key + store.getMerchantId();

		if (store != null && store.isUseCache()) {

			try {
				o = admin.getFromCache(key, 2000);
			} catch (NeedsRefreshException nre) {

				admin.cancelUpdate(key);

				/*
				 * try { // Get the value myValue =
				 * "This is the content retrieved."; // Store in the cache
				 * admin.putInCache(myKey, myValue); updated = true; } finally {
				 * if (!updated) { // It is essential that cancelUpdate is
				 * called if the // cached content could not be rebuilt
				 * admin.cancelUpdate(myKey); } }
				 */

				// throw new CacheModuleException(nre);
			} catch (Exception e) {
				admin.cancelUpdate(key);
				throw new CacheModuleException(e);
			}
		}

		return o;

	}

	public void putInCache(String key, Object content, String group,
			MerchantStore store) throws CacheModuleException {

		if (store.isUseCache()) {

			key = key + store.getMerchantId();

			boolean updated = false;

			try {
				admin.putInCache(key, content, new String[] { group });
				updated = true;
			} catch (Exception nre) {
				admin.cancelUpdate(key);
			} finally {
				if (!updated) {
					// It is essential that cancelUpdate is called if the
					// cached content could not be rebuilt
					admin.cancelUpdate(key);
				}
			}

		}

	}

}



```
