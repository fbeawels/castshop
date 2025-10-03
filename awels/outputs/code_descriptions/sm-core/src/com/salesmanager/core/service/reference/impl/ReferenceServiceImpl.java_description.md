# ReferenceServiceImpl.java

## Review

## 1. Summary  

**Purpose**  
`ReferenceServiceImpl` is a façade that exposes a handful of static helper methods for retrieving geographical reference data (countries, zones, descriptions). It acts as a thin wrapper around an underlying `ReferenceService` (obtained through a `ServiceFactory`) and a `RefCache` that keeps a pre‑populated list of countries. The secondary class, `ZonesCollectionFilter`, filters a collection of `Zone` objects by country id.

**Key Components**  

| Class | Responsibility |
|-------|----------------|
| `ReferenceServiceImpl` | Static API for country/zone lookups, delegating to `ReferenceService` or `RefCache`. |
| `ZonesCollectionFilter` | Utility that filters a `Collection<Zone>` by the country id of each zone. |

**Design Patterns / Frameworks**  

- **Service Locator / Factory** – `ServiceFactory.getService(ServiceFactory.ReferenceService)` is used to obtain the real service implementation.  
- **Cache** – `RefCache` holds a map of country objects keyed by id.  
- **Static façade** – All API methods are `static`, making the class a global helper.  
- **Generic collections** – The code uses raw types (e.g. `List` without type parameters) which predates modern Java generics.  

The implementation is very lightweight and does not leverage any heavyweight frameworks such as Spring or Hibernate directly.

---

## 2. Detailed Description  

### Execution Flow  

1. **Lookup by Name / ISO / ID**  
   - The static methods (`getCountryByName`, `getCountryByIsoCode`, `getCountryById`) call the singleton `ReferenceService` obtained via `ServiceFactory`.  
   - `getCountryById` uses the `RefCache` map instead of going through the service.  

2. **Lookup of Zone Objects**  
   - `getZoneCodeByCode` and `getZoneCodeByName` similarly delegate to the underlying `ReferenceService`.  

3. **Filtering Zones by Country**  
   - `getFilterdByCountryZones` creates a `ZonesCollectionFilter`, pulls the full zone list from `RefCache`, and applies the filter.  

### Interaction between Components  

- **ReferenceServiceImpl ↔ ReferenceService** – Direct delegation; no business logic.  
- **ReferenceServiceImpl ↔ RefCache** – Shared read‑only cache for countries (and zones, though not directly used).  
- **ZonesCollectionFilter ↔ Zone** – Performs a simple filter; does not modify the original collection.  

### Assumptions & Constraints  

- The `ServiceFactory` must have a registered implementation for `ReferenceService`.  
- `RefCache` contains all countries and zones when the application starts.  
- The code assumes that all lookups succeed; if the service throws an exception it propagates up.  
- The `HttpServletRequest` parameter in `getFilterdByCountryZones` is currently unused; its presence hints at possible future request‑specific filtering (e.g., language, user locale).  

### Architecture  

The architecture is intentionally flat: a single façade class exposing static methods and a tiny helper. This reduces boilerplate but sacrifices testability and flexibility. The cache is a simple in‑memory map; no persistence or invalidation logic is visible.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects / Notes |
|--------|---------|------------|---------|----------------------|
| `getCountryByName(String countryname, int languageId)` | Find a `Country` by its localized name. | `countryname`, `languageId` | `Country` | Delegates to `ReferenceService`; throws `Exception`. |
| `getCountryByIsoCode(String isocode, Locale locale)` | Get a `CountryDescription` using ISO code and locale. | `isocode`, `locale` | `CountryDescription` | Uses `LanguageUtil` to convert locale to language id. |
| `getCountryByIsoCode(String isocode)` | Find a `Country` by ISO code. | `isocode` | `Country` | Delegates to `ReferenceService`. |
| `getCountryById(int id)` | Retrieve a cached `Country` by its id. | `id` | `Country` | Reads from `RefCache`. |
| `getZoneCodeByCode(String code, Locale locale)` | Find a `Zone` by ISO code and locale. | `code`, `locale` | `Zone` | Delegates to `ReferenceService`. |
| `getZoneCodeByName(String name, int language)` | Find a `Zone` by name and language. | `name`, `language` | `Zone` | Delegates to `ReferenceService`. |
| `getFilterdByCountryZones(int countryid, HttpServletRequest req)` | Return zones belonging to a country. | `countryid`, `req` | `Collection<Zone>` | Calls `ZonesCollectionFilter.filterCollection`. |
| `ZonesCollectionFilter.filterCollection(int value, Collection original, HttpServletRequest req)` | Internal helper that filters zones by country id. | `value` (country id), `original` (zone collection), `req` | `List<Zone>` | Iterates over `original`; returns new list. |

**Reusable / Utility Methods**  

- The `filterCollection` method is the only reusable utility; it can be used elsewhere if zone filtering by country is required.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `ServiceFactory` | Third‑party (application‑specific) | Provides the `ReferenceService` instance. |
| `ReferenceService` | Interface (application‑specific) | Exposes lookup methods for countries and zones. |
| `RefCache` | Application‑specific cache | Holds maps of countries and zones. |
| `LanguageUtil` | Utility (application‑specific) | Converts Java `Locale` to a numeric language id. |
| `HttpServletRequest` | Java EE | Passed to `getFilterdByCountryZones` and `filterCollection` but not actually used. |
| `Country`, `CountryDescription`, `Zone` | Entities | Domain objects representing reference data. |
| `java.util.*`, `javax.servlet.*` | Standard library | Collection types, HTTP request handling. |

No external frameworks (Spring, Hibernate, etc.) are used directly. The code relies on the application’s own service and cache layers.

---

## 5. Additional Notes & Recommendations  

### Strengths  

- **Simplicity** – The façade presents a minimal API for common lookup operations.  
- **Caching** – `RefCache` improves performance for country lookups by id.  
- **Separation of Concerns** – Lookup logic is delegated to `ReferenceService`; the façade does not contain business rules.

### Weaknesses & Risks  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Static façade** | Hard to unit‑test; global state; violates dependency injection principles. | Convert to an instance bean, inject `ReferenceService` and `RefCache`. |
| **Raw types** | Compile‑time type safety lost; may lead to `ClassCastException`. | Use generics: `List<Zone>`, `Collection<Zone>`. |
| **Exception handling** | Methods throw generic `Exception`; callers need to handle or propagate. | Define custom checked/unchecked exceptions or use runtime exceptions. |
| **Unused parameters** | `HttpServletRequest` in filtering is unused; may mislead developers. | Remove the parameter or implement request‑based filtering (e.g., locale). |
| **No validation** | Null inputs can cause `NullPointerException` at runtime. | Validate arguments, throw informative exceptions. |
| **Cache coupling** | `getCountryById` bypasses the service layer entirely; future changes to the service may not reflect in the cache. | Consistently use the service or provide a single source of truth. |
| **Thread‑safety** | `RefCache` is assumed immutable, but if it can change, concurrent access may be problematic. | Ensure immutability or synchronize access. |
| **Documentation** | Javadoc comments are minimal and sometimes inaccurate (e.g., method names). | Expand Javadoc, include parameter descriptions, examples. |

### Future Enhancements  

1. **Injectable Service Layer** – Replace static methods with an instance that can be injected (e.g., Spring’s `@Service`).  
2. **Generic Collections** – Modernize code with Java 8+ generics and streams for filtering.  
3. **Locale‑Aware Caching** – Extend `RefCache` to cache descriptions per locale, avoiding repeated lookups.  
4. **Request‑Scoped Filtering** – Use `HttpServletRequest` to filter by user language or permissions.  
5. **Logging** – Add structured logging for lookup failures or cache misses.  
6. **Error Handling** – Introduce domain‑specific exceptions (e.g., `CountryNotFoundException`).  
7. **Unit Tests** – With dependency injection, write unit tests for each lookup method, mocking the service and cache.

By addressing these points the code will become more robust, testable, and maintainable while preserving the convenience of the current façade.

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
package com.salesmanager.core.service.reference.impl;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.LanguageUtil;

/**
 * Manage Country, Zone, Currencies
 * 
 * @author Carl Samson
 * 
 */
public class ReferenceServiceImpl {

	/**
	 * Will return a Country Object based on the textual country name e.g.
	 * Canada
	 * 
	 * @param countryname
	 * @return com.salesmanager.core.entity.reference.Country
	 * @throws Exception
	 */
	public static Country getCountryByName(String countryname, int languageId)
			throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		return rservice.getCountryByName(countryname, languageId);

	}

	public static CountryDescription getCountryByIsoCode(String isocode,
			Locale locale) throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		return rservice.getCountryDescriptionByIsoCode(isocode, LanguageUtil
				.getLanguageNumberCode(locale.getLanguage()));

	}

	public static Country getCountryByIsoCode(String isocode) throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		return rservice.getCountryByIsoCode(isocode);

	}

	public static Country getCountryById(int id) throws Exception {

		Map m = RefCache.getCountriesMap();
		return (Country) m.get(id);

	}

	public static Zone getZoneCodeByCode(String code, Locale locale)
			throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		return rservice.getZoneByIsoCode(code, LanguageUtil
				.getLanguageNumberCode(locale.getLanguage()));

	}

	public static Zone getZoneCodeByName(String name, int language)
			throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		return rservice.getZoneByName(name, language);

	}

	public static Collection<com.salesmanager.core.entity.reference.Zone> getFilterdByCountryZones(
			int countryid, HttpServletRequest req) {
		ZonesCollectionFilter filter = new ZonesCollectionFilter();
		RefCache cache = RefCache.getInstance();
		List newzones = filter.filterCollection(countryid, cache.getZones(),
				req);
		return newzones;
	}

}

class ZonesCollectionFilter {
	protected List<com.salesmanager.core.entity.reference.Zone> filterCollection(
			int value, Collection original, HttpServletRequest req) {

		List returnzones = new ArrayList();
		Iterator i = original.iterator();
		while (i.hasNext()) {
			Zone z = (Zone) i.next();
			if (z.getZoneCountryId() == value) {

				returnzones.add(z);
			}
		}
		return returnzones;
	}
}



```
