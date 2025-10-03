# ServicesUtil.java

## Review

## 1. Summary  

**Purpose** – `ServicesUtil` is a helper that loads all *core module* services (payment methods, shipping quotes, and other miscellaneous modules) at application start‑up and exposes them through a set of static lookup APIs.  

**Key components**  
| Component | Role |
|-----------|------|
| `paymentMethodList`, `shippingMethodList`, `otherModulesList` | In‑memory collections holding all services, grouped by type. |
| `CacheUtil` | A simple key/value cache used to expose country‑specific service look‑ups without repeated list scans. |
| Static block | Reads the list of `CoreModuleService` objects from the `ReferenceService`, populates the lists, and builds country‑specific caches (`pXX` for payment methods, a global `"modules"` map for all services). |
| Public static methods | Retrieve payment methods, shipping quotes, or any module by country ISO code or by name. |

**Design patterns / libraries**  
* **Singleton** – `CacheUtil` appears to be a thread‑safe, lazily‑instantiated cache.  
* **Factory** – `ServiceFactory` is used to obtain the `ReferenceService`.  
* **Simple caching** – All lookups are cached in memory; no external caching framework is used.  
* **Java collections** – Raw types are used extensively (e.g., `List`, `Map`) instead of generics.

---

## 2. Detailed Description  

1. **Static initialization**  
   * A `ReferenceService` is fetched from the `ServiceFactory`.  
   * `getCoreModuleServices()` returns a collection of `CoreModuleService` objects.  
   * For each service, the code decides whether it is a *payment*, *shipping*, or *miscellaneous* module and stores it in the appropriate list.  
   * Two caches are populated:
     * A country‑specific cache keyed `"pXX"` for payment methods, where `XX` is the ISO country code (upper‑cased).
     * A global `"modules"` cache that maps `"XX‑ModuleName"` to the `CoreModuleService`.

2. **Runtime behaviour**  
   * The static lists are immutable after construction; no public mutators exist.  
   * Public static methods perform lookups by:
     * Country‑specific cache (e.g., `getPaymentMetod`, `getShippingRealTimeQuotesMethods`).
     * Global cache (e.g., `getModule`).
     * Direct list iteration (e.g., `getMiscModulesList`, `getShippingRealTimeQuotesMethods` without a country filter).  
   * The returned objects are the same instances that were cached, meaning callers can safely modify them if the business logic allows.

3. **Assumptions & constraints**  
   * The reference data is static; any change to `CoreModuleService` objects must trigger a restart or a manual cache refresh.  
   * All ISO codes are compared case‑insensitively; the cache keys are always upper‑cased.  
   * The application is single‑threaded or the static initialization is performed only once (the code is not synchronized for reads, but the collections are only written once).

4. **Architectural decisions**  
   * **Eager loading** – All services are loaded on application start, avoiding lazy look‑ups at runtime.  
   * **In‑memory caching** – The cache is a simple HashMap; no external caching library or persistence layer.  
   * **Raw types** – The code predates generics or chooses not to use them for backward compatibility.

---

## 3. Functions / Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects | Notes |
|--------|-----------|---------|--------|---------|--------------|-------|
| `private ServicesUtil()` | N/A | Private constructor to prevent instantiation. | N/A | N/A | N/A | Standard utility pattern. |
| `public static List getPaymentMethodList()` | `List` | Returns the raw list of all payment method services. | N/A | `List` (raw) | None | Should return a typed `List<CoreModuleService>` and be immutable. |
| `public static CoreModuleService getPaymentMetod(String name, String countryIsoCode)` | `CoreModuleService` | Looks up a payment module by name and country. Falls back to the `ALLCOUNTRY` cache if country‑specific entry is missing. | `name`, `countryIsoCode` | Matching `CoreModuleService` or `null` | None | Misspelled *Metod*; uses raw cache. |
| `public static List<com.salesmanager.core.entity.reference.CoreModuleService> getServices(String countryIsoCode)` | `List<CoreModuleService>` | Aggregates payment, shipping, and misc modules for the given country. | `countryIsoCode` | Combined list | None | Calls three other helper methods. |
| `public static CoreModuleService getModule(String countryIsoCode, String name)` | `CoreModuleService` | Retrieves any module by country and name from the global `"modules"` cache. | `countryIsoCode`, `name` | Module or `null` | None | Returns country‑specific or fallback (`ALLCOUNTRY`). |
| `public static List<CoreModuleService> getServices()` | `List<CoreModuleService>` | Returns all services (payment + shipping + misc). | N/A | Combined list | None | Reads from the three static lists. |
| `public static Map getPaymentMetodsMap(String countryIsoCode)` | `Map` | Returns the country‑specific payment map (or fallback). | `countryIsoCode` | `Map` (raw) | None | Typo *Metods*; raw `Map`. |
| `public static List getMiscModulesList(String countryIsoCode)` | `List` | Returns misc modules applicable to a country (country‑specific or global). | `countryIsoCode` | List of `CoreModuleService` | None | Manual iteration, could be replaced by filtering. |
| `public static List getPaymentMethods()` | `List` | Returns all payment method services (no filtering). | N/A | Raw list | None | Unused `HashSet` in code; returns the original `paymentMethodList`. |
| `public static List getPaymentMethodsList(String countryIsoCode)` | `List` | Returns payment methods for a specific country (combining global and country‑specific). | `countryIsoCode` | List of `CoreModuleService` | None | Builds a temporary `Map` to deduplicate; could be simplified. |
| `public static List<CoreModuleService> getShippingRealTimeQuotesMethods(String countryIsoCode)` | `List<CoreModuleService>` | Returns RT shipping quotes for a country. | `countryIsoCode` | List of `CoreModuleService` | None | Manual iteration, same pattern as misc modules. |
| `public static List getShippingRealTimeQuotesMethods()` | `List` | Returns all RT shipping modules. | N/A | Raw list | None | Simply returns `shippingMethodList`. |

*All public methods are static; no instance state is modified.*

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party logging | Standard log4j 1.x. |
| `com.salesmanager.core.constants.*` | Internal | Constants for service codes and ISO codes. |
| `com.salesmanager.core.entity.reference.CoreModuleService` | Internal | Domain entity representing a service module. |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Factory for retrieving other services (e.g., `ReferenceService`). |
| `com.salesmanager.core.service.reference.ReferenceService` | Internal | Provides `getCoreModuleServices()`. |
| `com.salesmanager.core.util.CacheUtil` | Internal | Simple in‑memory cache; used for both country‑specific and global maps. |

*No external persistence or remote APIs are called directly; all data originates from the `ReferenceService`.*

---

## 5. Additional Notes  

### Strengths  
* **Simple, eager loading** – All service data is available immediately after startup, avoiding costly lazy look‑ups.  
* **Clear separation by module type** – Three distinct lists make it obvious which services belong where.  
* **Fallback strategy** – Country‑specific look‑ups gracefully fall back to global (`ALLCOUNTRY`) entries.  

### Issues & Improvement Opportunities  

| Category | Observation | Suggested Fix |
|----------|-------------|---------------|
| **Generics** | Raw types (`List`, `Map`) throughout. | Use generics everywhere (`List<CoreModuleService>`, `Map<String, CoreModuleService>`) to avoid unchecked casts and compiler warnings. |
| **Naming** | Misspelled method names (`getPaymentMetod`, `getPaymentMetodsMap`). | Rename to `getPaymentMethod` and `getPaymentMethodsMap`. |
| **Thread safety** | Static lists are read‑only after initialization, but the cache population in the static block is not synchronized. | Wrap the static block in a `synchronized` block or use a `static final` initialization with `Collections.unmodifiableList`. |
| **Null safety** | `cf.getCacheMap` can return `null`; subsequent casts are unchecked. | Guard against `null` and throw informative exceptions or return empty collections. |
| **Code duplication** | Several methods iterate over the same list to filter by country or ISO. | Extract a helper `filterByCountry` that accepts a list and a predicate. |
| **Unused variables** | `HashSet returnlist` in `getPaymentMethods()` is never used. | Remove the unused variable. |
| **Documentation** | Methods lack Javadoc; code comments are minimal. | Add Javadoc for each public method, describing parameters, return values, and possible exceptions. |
| **Cache strategy** | `CacheUtil` appears to be a simple `Map`‑based cache; no expiration or eviction logic. | Consider a TTL or LRU cache if the dataset can grow large or change at runtime. |
| **Performance** | For small data sets this is fine; however, iterating through entire lists on every request could become a bottleneck if the number of services grows. | Keep country‑specific maps for quick lookup; remove iteration in public APIs. |
| **Testing** | No unit tests shown; the static initialization makes mocking difficult. | Refactor to allow dependency injection of `ReferenceService` and `CacheUtil` for easier testing. |

### Future Enhancements  

1. **Immutable collections** – Return unmodifiable lists/sets to prevent accidental modification by callers.  
2. **Dynamic reloading** – Expose a method to refresh the cached services (e.g., after an admin adds a new module).  
3. **Better error handling** – Replace `log.error("FATAL STATIC INIT " + e)` with a more informative message and rethrow if initialization fails.  
4. **Internationalization** – Add support for locales beyond ISO codes (e.g., region or currency).  
5. **Metrics** – Expose simple counters for cache hits/misses to aid monitoring.  

Overall, `ServicesUtil` serves a clear purpose and is straightforward, but it would benefit from modern Java practices (generics, immutability, proper documentation) and a few small refactors to improve safety and maintainability.

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
package com.salesmanager.core.service.common.impl;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;

import org.apache.log4j.Logger;

import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.CacheUtil;

/**
 * Build List of CenralIntegrationServices
 * 
 * @author Carl Samson
 * 
 */
public class ServicesUtil {

	private ServicesUtil() {
	}

	private static List paymentMethodList = new ArrayList();
	private static List shippingMethodList = new ArrayList();
	private static List otherModulesList = new ArrayList();

	private static Logger log = Logger.getLogger(ServicesUtil.class);

	static {

		try {

			ReferenceService rservice = (ReferenceService) ServiceFactory
					.getService(ServiceFactory.ReferenceService);
			Collection services = rservice.getCoreModuleServices();

			CacheUtil cf = CacheUtil.getInstance();
			CacheUtil global = CacheUtil.getInstance();
			Map globalmap = global.createCacheMap("modules");
			Iterator i = services.iterator();
			while (i.hasNext()) {
				CoreModuleService cs = (CoreModuleService) i.next();
				if (cs.getCoreModuleServiceCode() == PaymentConstants.INTEGRATION_SERVICE_PAYMENT_METHODS) {
					paymentMethodList.add(cs);
					Map p = null;

					if (cf.containsCache("p"
							+ cs.getCountryIsoCode2().toUpperCase())) {
						p = cf.getCacheMap("p"
								+ cs.getCountryIsoCode2().toUpperCase());
					} else {
						p = cf.createCacheMap("p"
								+ cs.getCountryIsoCode2().toUpperCase());
					}
					p.put(cs.getCoreModuleName(), cs);
				} else if (cs.getCoreModuleServiceCode() == ShippingConstants.INTEGRATION_SERVICE_SHIPPING_RT_QUOTE) {
					shippingMethodList.add(cs);
				} else {
					otherModulesList.add(cs);
				}
				globalmap.put(cs.getCountryIsoCode2().toUpperCase() + "-"
						+ cs.getCoreModuleName(), cs);

			}

		} catch (Exception e) {
			log.error("FATAL STATIC INIT " + e);
		}
	}

	public static List getPaymentMethodList() {
		return paymentMethodList;
	}

	public static CoreModuleService getPaymentMetod(String name,
			String countryIsoCode) {
		CacheUtil cf = CacheUtil.getInstance();

		Map map = cf.getCacheMap("p" + countryIsoCode.toUpperCase());
		if (map != null) {
			CoreModuleService cs = (CoreModuleService) map.get(name);
			return cs;
		} else if (cf.getCacheMap("p" + Constants.ALLCOUNTRY_ISOCODE) != null) {
			map = cf.getCacheMap("p" + Constants.ALLCOUNTRY_ISOCODE);
			CoreModuleService cs = (CoreModuleService) map.get(name);
			return cs;
		}
		return null;
	}

	public static List<com.salesmanager.core.entity.reference.CoreModuleService> getServices(
			String countryIsoCode) {

		List retlist = new ArrayList();
		retlist.addAll(getPaymentMethodsList(countryIsoCode));
		retlist.addAll(getShippingRealTimeQuotesMethods(countryIsoCode));
		retlist.addAll(getMiscModulesList(countryIsoCode));
		return retlist;

	}

	public static CoreModuleService getModule(String countryIsoCode, String name) {
		CacheUtil global = CacheUtil.getInstance();
		if (global.containsCache("modules")) {
			Map cache = global.getCacheMap("modules");
			CoreModuleService module = (CoreModuleService) cache
					.get(countryIsoCode.toUpperCase() + "-" + name);
			if (module == null) {
				module = (CoreModuleService) cache
						.get(Constants.ALLCOUNTRY_ISOCODE + "-" + name);
			}
			return module;
		} else {
			return null;
		}
	}

	public static List<CoreModuleService> getServices() {

		List retlist = new ArrayList();
		retlist.addAll(paymentMethodList);
		retlist.addAll(shippingMethodList);
		retlist.addAll(otherModulesList);
		return retlist;

	}

	public static Map getPaymentMetodsMap(String countryIsoCode) {
		CacheUtil cf = CacheUtil.getInstance();
		if (cf.containsCache("p" + countryIsoCode.toUpperCase())) {
			return cf.getCacheMap("p" + countryIsoCode.toUpperCase());
		} else if (cf.containsCache("p" + Constants.ALLCOUNTRY_ISOCODE)) {
			return cf.getCacheMap("p" + Constants.ALLCOUNTRY_ISOCODE);
		} else {
			return new HashMap();
		}
	}

	public static List getMiscModulesList(String countryIsoCode) {

		List returnlist = new ArrayList();

		List services = otherModulesList;

		Iterator i = services.iterator();
		while (i.hasNext()) {
			CoreModuleService cs = (CoreModuleService) i.next();

			if (cs.getCountryIsoCode2().equalsIgnoreCase(countryIsoCode)) {
				returnlist.add(cs);
				continue;
			}
			if (cs.getCountryIsoCode2().equals(Constants.ALLCOUNTRY_ISOCODE)) {
				returnlist.add(cs);
			}

		}

		return returnlist;

	}

	public static List getPaymentMethods() {

		Set returnlist = new HashSet();

		List services = paymentMethodList;

		return services;
	}

	public static List getPaymentMethodsList(String countryIsoCode) {

		//List returnlist = new ArrayList();
		Map paymentMap = new HashMap();

		// List services = (List)RefCache.getServiceintegrationlist();
		List services = paymentMethodList;

		Iterator i = services.iterator();
		while (i.hasNext()) {
			CoreModuleService cs = (CoreModuleService) i.next();


			if (cs.getCountryIsoCode2().equals(Constants.ALLCOUNTRY_ISOCODE)) {
				paymentMap.put(cs.getCoreModuleName(), cs);
			} else if(cs.getCountryIsoCode2().equalsIgnoreCase(countryIsoCode)) {
				paymentMap.put(cs.getCoreModuleName(), cs);
			}

		}

		return new ArrayList(paymentMap.values());
	}

	/**
	 * Builds a list of RT quotes shipping module based on the countryid
	 * 
	 * @param countryid
	 * @return
	 */

	public static List<CoreModuleService> getShippingRealTimeQuotesMethods(
			String countryIsoCode) {

		List returnlist = new ArrayList();
		// List services = (List)RefCache.getServiceintegrationlist();
		List services = shippingMethodList;

		Iterator i = services.iterator();
		while (i.hasNext()) {
			CoreModuleService cs = (CoreModuleService) i.next();
			if (cs.getCountryIsoCode2().equalsIgnoreCase(countryIsoCode)) {
				returnlist.add(cs);
				continue;
			}
			if (cs.getCountryIsoCode2().equals(Constants.ALLCOUNTRY_ISOCODE)) {
				returnlist.add(cs);
			}

		}

		return returnlist;
	}

	public static List getShippingRealTimeQuotesMethods() {

		return shippingMethodList;
	}
}



```
