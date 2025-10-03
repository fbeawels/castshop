# RefCache.java

## Review

## 1. Summary

`RefCache` is a **lazy‑initialised, application‑wide cache** that holds various reference data (countries, zones, currencies, units, languages, order status, etc.) retrieved once from a `ReferenceService`.  
It follows a classic *singleton* pattern: a private static instance is created on the first call to `getInstance()`, and a `loaded` flag guarantees that the cache is populated only once.  

Key components  
| Component | Role |
|-----------|------|
| `ReferenceService rservice` | DAO that loads all reference tables |
| `Map` collections (`countriesmap`, `zonesmapByLang`, …) | In‑memory lookup tables keyed by country id, language id, zone id, etc. |
| `ZonesCollectionFilter` | Helper to filter zones by country for a given language |

The class uses only **standard Java SE collections** and Apache Log4j for logging. No frameworks (Spring, Guice, etc.) are involved.

---

## 2. Detailed Description

### 2.1 Initialization Flow

1. **Singleton creation** – `getInstance()` lazily creates a single `RefCache` instance.  
2. **Cache population** – The constructor invokes `createCache()`.  
3. **Data retrieval** – `ReferenceService` is queried for each reference table: countries, zones, country status, order status, currencies, units, languages.  
4. **Map construction** – For each retrieved record, several maps are filled:
   * Countries are stored by id, by language, and by ISO code per language.  
   * Zones are stored per language.  
   * Units are split into weight and size maps.  
   * Other lookup tables (order status, currencies, languages) are similarly populated.  
5. **Flagging** – After successful population, `loaded` is set to `true`.  

### 2.2 Runtime Behavior

* **Read‑only access** – All public API methods simply expose the pre‑built maps or collections.  
* **No mutation** – The cache is immutable after creation (unless `createCache()` is called again, which is not supported).  
* **Thread‑unsafe** – There is no synchronization around the singleton or the maps, making the class unsafe in multi‑threaded environments that might call `getInstance()` or `createCache()` concurrently.

### 2.3 Dependencies & Assumptions

* **External API** – Relies on `ReferenceService` from `ServiceFactory` to provide the underlying data.  
* **Logging** – Uses Log4j (`org.apache.log4j.Logger`).  
* **Java version** – The code predates generics in several places; raw collections are used heavily.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `getInstance()` | Returns the singleton cache instance. | – | `RefCache` | Lazy creation of the instance |
| `isLoaded()` | Indicates whether the cache has been populated. | – | `boolean` | – |
| `createCache()` | Populates all internal maps. | – | – | Sets `loaded = true` |
| `getCountries()` | Returns the raw collection of all countries. | – | `Collection` | – |
| `getZones()` | Returns the raw collection of all zones. | – | `Collection` | – |
| `getFilterdByCountryZones(int countryid, int languageid)` | Returns zones belonging to a country for a specific language. | `int countryid, int languageid` | `Collection` | Uses `ZonesCollectionFilter` |
| `getAllcountriesmap(int language)` | Countries by id for a language. | `int language` | `Map<Integer, Country>` | – |
| `getAllcountriesmapbycode(int language)` | Countries by ISO code for a language. | `int language` | `Map<String, Country>` | – |
| `getCountriesMap()` | All countries by id. | – | `Map<Integer, Country>` | – |
| `getOrderstatuswithlang(int lang)` | Order status entries for a language. | `int lang` | `Map<Integer, OrderStatus>` | – |
| `getAllZonesmap(int lang)` | Zones by id for a language. | `int lang` | `Map<Integer, Zone>` | – |
| `getCurrenciesListWithCodes()` | Currency code → Currency map. | – | `Map<String, Currency>` | – |
| `getLanguageswithindex()` | Language id → Language map. | – | `Map<Integer, Language>` | – |
| `getProductTypes()` | Delegates to `ReferenceService`. | – | `Collection` | – |
| `getSupportedCountriesMap(int languageId)` | Supported countries for a language. | `int languageId` | `Map<Integer, Country>` | – |
| `getSupportedCreditCards()` | Delegates to `ReferenceService`. | – | `Map` | – |
| `getCountriesStatus()` | Country status map. | – | `Map<Integer, CentralCountryStatus>` | – |
| `getLanguageswithcode()` | Code → Language map. | – | `Map<String, Language>` | – |
| `getSizeunits()` | Size unit map. | – | `Map<String, CentralMeasureUnits>` | – |
| `getWeightunits()` | Weight unit map. | – | `Map<String, CentralMeasureUnits>` | – |

`ZonesCollectionFilter.filterCollection` is a small helper that filters a `Map<Integer, Zone>` by a country id.

---

## 4. Dependencies

| External | Type | Comments |
|----------|------|----------|
| `com.salesmanager.core.service.reference.ReferenceService` | DAO / Service | Provides all reference data |
| `com.salesmanager.core.constants.Constants` | Constant holder | Defines unit type codes |
| `org.apache.log4j.Logger` | Logging | Simple log4j logger |
| Java SE collections (`Map`, `TreeMap`, `HashMap`, `Collection`, `Iterator`, `Set`) | Standard | No third‑party collections |
| `ServiceFactory` | Service locator | Returns the `ReferenceService` instance |

There are **no platform‑specific** dependencies; the code runs on any Java SE environment that includes Log4j.

---

## 5. Additional Notes & Recommendations

### 5.1 Thread Safety

* **Singleton construction is not thread‑safe.** Multiple threads could create multiple instances before the static `cache` is set.  
* **`loaded` flag and map population** are also unsynchronised. Concurrent calls to `createCache()` could corrupt the internal maps.  

**Fixes**  
- Make `getInstance()` synchronized or use a static initializer (`private static final RefCache cache = new RefCache();`).  
- Consider `volatile` for `loaded` or use an `AtomicBoolean`.  
- Use a `ReadWriteLock` if the cache will ever need to be refreshed.

### 5.2 Type Safety

The code heavily uses **raw types** (`Map`, `Collection`, `Iterator`). This leads to unchecked casts and potential `ClassCastException`.  
**Recommendation:** Use generics throughout:

```java
private static Map<Integer, Country> countriesmap = new TreeMap<>();
private static Map<Integer, Map<Integer, Zone>> zonesmapByLang = new TreeMap<>();
// etc.
```

### 5.3 Readability & Maintainability

* Method names such as `getFilterdByCountryZones` contain typos and are not idiomatic.  
* The `createCache()` method is large (≈ 450 lines) and handles many concerns. Breaking it into smaller, well‑named private methods (e.g., `loadCountries()`, `loadZones()`, `loadUnits()`) would improve readability.  
* Logging only the exception stack trace (`log.error(ex)`) hides the cause when exceptions are swallowed; consider logging more context or rethrowing a runtime exception.

### 5.4 Performance

* The code uses `TreeMap` for all lookups. While order is maintained, a `HashMap` would typically be faster for lookup‑heavy scenarios.  
* `zonesmapByLang` and `allcountriesmapbylang` create new `TreeMap` instances each time `getAllZonesmap` or `getAllcountriesmap` is called when the language is missing. This is negligible but could be simplified by returning an immutable empty map.

### 5.5 Cache Expiry / Refresh

Currently the cache is *static and never refreshed*. If reference data changes at runtime, the system has no way to invalidate or update the cache.  
Possible enhancements:
- Expose a `refresh()` method that clears and repopulates all maps.  
- Use a `ScheduledExecutorService` to refresh at a fixed interval.  
- Integrate with an event system that notifies the cache when underlying data changes.

### 5.6 API Design

* The public API exposes raw `Map` and `Collection` objects, allowing callers to modify them. Consider returning unmodifiable views (`Collections.unmodifiableMap(...)`) to preserve immutability.  
* Return more specific types, e.g., `Map<Integer, Country>` instead of raw `Map`, to aid IDE autocomplete.

### 5.7 Documentation

* Javadoc is minimal. Adding method descriptions, parameter explanations, and notes about thread‑safety would help future developers.  
* The class header contains a legal notice but no functional description of the cache strategy.

---

### Bottom‑Line

`RefCache` is a functional, but **raw‑style, single‑threaded** cache implementation. It fulfills its purpose of providing quick in‑memory lookups for reference data, but it lacks thread safety, type safety, and modern Java practices. Refactoring to use generics, synchronized singleton construction, and clearer API contracts would make the code more robust, maintainable, and easier to extend.

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
package com.salesmanager.core.service.cache;

import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.TreeMap;

import org.apache.log4j.Logger;

import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.orders.OrderStatus;
import com.salesmanager.core.entity.reference.CentralCountryStatus;
import com.salesmanager.core.entity.reference.CentralMeasureUnits;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.entity.reference.ZoneDescription;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.reference.ReferenceService;

/**
 * Contains all reference tables
 * 
 * @author Carl Samson
 * 
 */
public class RefCache {

	private static boolean loaded = false;
	private static RefCache cache = null;
	private static Logger log = Logger.getLogger(RefCache.class);

	private static List countries = null;
	private static Collection zones = null;

	private static Map zonesmap = new TreeMap();// contains zoneid Zone object
	private static Map zonesmapByLang = new TreeMap();// contains <int
														// lang,<zonemap>>

	private static Map allcountriesmapbylang = new TreeMap();// contains <int
																// langid,<allcountriesmap>>
	private static Map allcountriesmapbylangbycode = new TreeMap();// contains
																	// <int
																	// langid,<allcountriesmapbycode>>
																	// allcountriesmapbycode
																	// ->
																	// countryIsoCode,
																	// Country
	private static Map countriesmap = new TreeMap();// contains countries
													// <countryId, Country>

	private static Map supportedCountriesMapByLang = new TreeMap();// supported
																	// countries
																	// by the
																	// system
																	// countryid,
																	// Country

	private static Map countriesStatus = new TreeMap();

	private static Map weightunits = new TreeMap();// contains
													// CentralMeasureUnits
													// objects with key as code
													// (CM, KG...)
	private static Map sizeunits = new TreeMap();// contains CentralMeasureUnits
													// objects with key code

	private static Map orderstatuswithlang = new HashMap();// orderstatus by
															// integer language

	private static Map currencieswithcode = new TreeMap();

	private static Map languageswithindex = new TreeMap();// contains
															// languageId,
															// Language
	private static Map languageswithcode = new TreeMap();// contains code,
															// Language

	private static ReferenceService rservice = (ReferenceService) ServiceFactory
			.getService(ServiceFactory.ReferenceService);

	private RefCache() {
		createCache();
	}

	public static RefCache getInstance() {
		if (cache == null) {
			cache = new RefCache();
		}
		return cache;
	}
	
	public static boolean isLoaded() {
		return loaded;
	}

	public static Collection getCountries() {
		return countries;
	}

	public static Collection getZones() {
		return zones;
	}

	public static void createCache() {

		if (loaded)
			return;


		try {


			/**
			 * Country
			 */

			Collection allcts = rservice.getCountries();
			
			if (allcts != null && allcts.size()>0) {
				Iterator allctsit = allcts.iterator();
				while (allctsit.hasNext()) {
					Country c = (Country) allctsit.next();
					Set descriptions = c.getDescriptions();
					countriesmap.put(c.getCountryId(), c);// all countries map
					if (descriptions != null) {
						Iterator i = descriptions.iterator();
						while (i.hasNext()) {
							CountryDescription desc = (CountryDescription) i
									.next();
							String name = desc.getCountryName();
							int langid = desc.getId().getLanguageId();
							c.setCountryName(name);
							Map allcountriesmap = (Map) allcountriesmapbylang
									.get(langid);
							if (allcountriesmap == null) {
								allcountriesmap = new TreeMap();
								allcountriesmapbylang.put(langid,
										allcountriesmap);
							}
							if (c.isSupported()) {

								Map supportedCountriesMap = (Map) supportedCountriesMapByLang
										.get(langid);
								if (supportedCountriesMap == null) {
									supportedCountriesMap = new TreeMap();
									supportedCountriesMapByLang.put(langid,
											supportedCountriesMap);
								}
								supportedCountriesMap.put(c.getCountryId(), c);
							}
							allcountriesmap.put(c.getCountryId(), c);
							Map allcountriesmapbycode = (Map) allcountriesmapbylangbycode
									.get(langid);
							if (allcountriesmapbycode == null) {
								allcountriesmapbycode = new TreeMap();
								allcountriesmapbylangbycode.put(langid,
										allcountriesmapbycode);
							}
							allcountriesmapbycode
									.put(c.getCountryIsoCode2(), c);
						}
					}

				}
			}

			/**
			 * Zone
			 */

			Collection zns = rservice.getZones();

			zones = zns;
			if (zns != null) {
				Iterator zonesit = zns.iterator();
				while (zonesit.hasNext()) {
					Zone z = (Zone) zonesit.next();
					Set descriptions = z.getDescriptions();

					if (descriptions != null) {

						Iterator i = descriptions.iterator();
						while (i.hasNext()) {
							ZoneDescription zd = (ZoneDescription) i.next();
							int lang = zd.getId().getLanguageId();
							Map zonemaplang = (Map) zonesmapByLang.get(lang);
							String name = zd.getZoneName();
							z.setZoneName(name);
							if (zonemaplang == null) {
								zonemaplang = new TreeMap();
								zonesmapByLang.put(lang, zonemaplang);
							}
							zonemaplang.put(z.getZoneId(), z);
						}
					}
					zonesmap.put(z.getZoneId(), z);
				}
			}

			/**
			 * Countries status
			 */

			Collection ctsstatus = rservice.getCountryStatus();
			if (ctsstatus != null) {
				Iterator ctsstatusit = ctsstatus.iterator();
				while (ctsstatusit.hasNext()) {
					CentralCountryStatus co = (CentralCountryStatus) ctsstatusit
							.next();
					countriesStatus.put(co.getCountryId(), co);
				}
			}

			/** Order Status **/
			Collection os = rservice.getOrderStatus();
			if (os != null) {
				Iterator osit = os.iterator();
				while (osit.hasNext()) {
					OrderStatus o = (OrderStatus) osit.next();

					Map vals = (Map) orderstatuswithlang.get(o.getId()
							.getLanguageId());
					if (vals == null) {
						vals = new TreeMap();
					}

					vals.put(o.getId().getOrderStatusId(), o);
					orderstatuswithlang.put(o.getId().getLanguageId(), vals);

				}
			}

			// Currencies

			Collection cur = rservice.getCurrencies();
			if (cur != null) {
				Iterator curit = cur.iterator();
				while (curit.hasNext()) {
					Object o = curit.next();
					com.salesmanager.core.entity.reference.Currency c = (com.salesmanager.core.entity.reference.Currency) o;
					currencieswithcode.put(c.getCode(), c);
				}
			}

			/** Units **/

			Collection units = rservice.getMeasureUnits();
			if (units != null) {
				Iterator allctsit = units.iterator();
				while (allctsit.hasNext()) {
					CentralMeasureUnits u = (CentralMeasureUnits) allctsit
							.next();
					if (u.getCentralMeasureUnitsType() == Constants.WEIGHT_UNITS_TYPE) {
						weightunits.put(u.getCentralMeasureUnitsCode(), u);
					} else if (u.getCentralMeasureUnitsType() == Constants.SIZE_UNITS_TYPE) {
						sizeunits.put(u.getCentralMeasureUnitsCode(), u);
					}
				}
			}

			Collection lang = rservice.getLanguages();
			if (lang != null) {
				Iterator langit = lang.iterator();
				while (langit.hasNext()) {
					Language l = (Language) langit.next();
					languageswithindex.put(l.getLanguageId(), l);
					languageswithcode.put(l.getCode(), l);
				}

			}

		} catch (Exception ex) {
			log.error(ex);
		}

		loaded = true;
	}

	/**
	 * Special methods
	 * 
	 * @return
	 */

	public static Collection getFilterdByCountryZones(int countryid,
			int languageid) {
		ZonesCollectionFilter filter = new ZonesCollectionFilter();
		Map newzones = filter.filterCollection(countryid,
				getAllZonesmap(languageid));
		return newzones.values();
	}

	public static Map getAllcountriesmap(int language) {

		Map returnmap = (Map) allcountriesmapbylang.get(language);
		if (returnmap == null) {
			returnmap = new TreeMap();
		}
		return returnmap;
	}

	public static Map getAllcountriesmapbycode(int language) {

		Map returnmap = (Map) allcountriesmapbylangbycode.get(language);
		if (returnmap == null) {
			returnmap = new TreeMap();
		}
		return returnmap;

	}

	public static Map getCountriesMap() {

		return countriesmap;

	}

	public static Map getOrderstatuswithlang(int lang) {
		if (orderstatuswithlang.containsKey(lang)) {
			return (Map) orderstatuswithlang.get(lang);
		} else {
			return new HashMap();
		}
	}

	public static Map getAllZonesmap(int lang) {
		Map returnmap = (Map) zonesmapByLang.get(lang);
		if (returnmap == null) {
			returnmap = new TreeMap();
		}
		return returnmap;

	}

	public static Map getCurrenciesListWithCodes() {
		return currencieswithcode;
	}

	public static Map getLanguageswithindex() {
		return languageswithindex;
	}

	public static Collection getProductTypes() {

		return rservice.getProductTypes();
	}

	public static Map getSupportedCountriesMap(int languageId) {

		Map countriesmap = (Map) supportedCountriesMapByLang.get(languageId);
		if (countriesmap != null) {
			return countriesmap;
		} else {
			return new TreeMap();
		}
	}

	public static Map getSupportedCreditCards() {
		return rservice.getSupportedCreditCards();
	}

	public static Map getCountriesStatus() {
		return countriesStatus;
	}

	public static Map getLanguageswithcode() {
		return languageswithcode;
	}

	public static Map getSizeunits() {
		return sizeunits;
	}

	public static Map getWeightunits() {
		return weightunits;
	}

}

class ZonesCollectionFilter {
	protected Map filterCollection(int value, Map original) {

		Map returnzones = new TreeMap();
		Iterator i = original.keySet().iterator();
		while (i.hasNext()) {
			int zoneId = (Integer) i.next();
			Zone z = (Zone) original.get(zoneId);
			if (z.getZoneCountryId() == value) {
				returnzones.put(zoneId, z);
			}
		}
		return returnzones;
	}
}



```
