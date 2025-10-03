# CountryUtil.java

## Review

## 1. Summary  

**Purpose**  
`CountryUtil` is a static helper class that centralises look‑ups for country and zone information in the Sales Manager system.  It exposes a small set of convenience methods that convert between common identifiers (country id, name, ISO code, locale) and the rich domain objects (`Country`, `CountryDescription`, `Zone`).

**Key Components**

| Component | Role |
|-----------|------|
| `RefCache` | Provides a read‑only cache of `Country` objects indexed by ID. |
| `ReferenceService` (via `ServiceFactory`) | Enterprise‑level service that can fetch country/zone data from the persistence layer. |
| `ReferenceServiceImpl` | Concrete implementation of `ReferenceService` – some of its static methods are used directly. |
| `LanguageUtil` | Utility that maps ISO language strings to numeric language codes. |

**Design patterns / libraries**

* **Service Locator** – `ServiceFactory.getService(...)` hides the instantiation details of `ReferenceService`.  
* **Cache** – `RefCache` implements an in‑memory cache of reference data.  
* **Log4j** – A `Logger` is created (though unused).  
* **Java 5 Generics** – The code predates or ignores generics, using raw `Map` types.  

The overall architecture is a thin façade over a more complex persistence/cache layer.  

---

## 2. Detailed Description  

### Flow of Execution

1. **Lookup by ID (`getCountryIsoCodeById`)**  
   * Retrieves the global country map from `RefCache`.  
   * Fetches a `Country` instance by key (the ID).  
   * Returns the ISO‑2 code, or `null` if not found.

2. **Lookup by Name (`getCountryByName`)**  
   * Delegates to the static method `ReferenceServiceImpl.getCountryByName`.  
   * Passes the supplied country name and language ID.  
   * Any underlying exception propagates outwards.

3. **Lookup by ISO code (`getCountryByIsoCode`)**  
   * Obtains an instance of `ReferenceService` from `ServiceFactory`.  
   * Calls `getCountryDescriptionByIsoCode`, passing the ISO code and a language number resolved by `LanguageUtil`.  
   * Returns the corresponding `CountryDescription`.

4. **Lookup Zone by code (`getZoneCodeByCode`) / by name (`getZoneCodeByName`)**  
   * Both call static methods on `ReferenceServiceImpl`.  
   * Return a `Zone` instance that matches the supplied name/code and locale/language.

### Assumptions & Constraints

* The cache (`RefCache`) is assumed to be populated elsewhere and never mutated by this utility.  
* The service layer (`ReferenceService`) is expected to be available in the runtime environment.  
* Locale handling assumes a two‑letter ISO language code (`locale.getLanguage()`).  
* The class is **not thread‑safe** for mutations (there are none) but relies on the thread‑safety of `RefCache` and `ReferenceService`.  
* Errors are largely propagated as generic `Exception` rather than more specific checked/unchecked types.

### Architecture & Design Choices

* **Static façade** – All methods are static, making the class convenient to call without instantiation.  
* **Mixed service usage** – Some methods call static `ReferenceServiceImpl` methods, others obtain an instance via `ServiceFactory`.  This inconsistency could confuse developers and complicate testing.  
* **Raw types** – The cache map uses a raw `Map` type, foregoing compile‑time type safety.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Exceptions | Notes |
|--------|---------|------------|--------|------------|-------|
| `getCountryIsoCodeById(int countryid)` | Fetches the ISO‑2 code for a country by its internal numeric ID. | `countryid` | `String` (ISO code) | None | Returns `null` if ID not present. |
| `getCountryByName(String countryname, int languageId)` | Retrieves a `Country` object by name and language. | `countryname`, `languageId` | `Country` | `Exception` (from underlying service) | Delegates to `ReferenceServiceImpl.getCountryByName`. |
| `getCountryByIsoCode(String countryIsoCode, Locale locale)` | Obtains a `CountryDescription` for a given ISO code and locale. | `countryIsoCode`, `locale` | `CountryDescription` | `Exception` | Uses `ServiceFactory` to obtain a `ReferenceService`. |
| `getZoneCodeByCode(String name, Locale locale)` | Finds a `Zone` by its code (likely a short string) and locale. | `name`, `locale` | `Zone` | `Exception` | Delegates to static `ReferenceServiceImpl.getZoneCodeByCode`. |
| `getZoneCodeByName(String name, int languageId)` | Finds a `Zone` by its full name and language. | `name`, `languageId` | `Zone` | `Exception` | Delegates to static `ReferenceServiceImpl.getZoneCodeByName`. |

*Reusable / utility methods:*  
The class itself acts as a reusable utility; all methods are public and static.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Declared but never used. |
| `com.salesmanager.core.entity.reference.*` | Domain entities | `Country`, `CountryDescription`, `Zone`. |
| `com.salesmanager.core.service.*` | Service layer | `ServiceFactory`, `ReferenceService`. |
| `com.salesmanager.core.service.cache.RefCache` | Cache | Provides static country map. |
| `com.salesmanager.core.util.LanguageUtil` | Utility | Converts locale language string to numeric code. |
| `java.util.Locale`, `java.util.Map` | Standard JDK | Raw types used. |

No external APIs beyond the internal Sales Manager framework and Log4j.

---

## 5. Additional Notes  

### Strengths  

* Centralises reference look‑ups in one place.  
* Uses caching to avoid repeated database hits for country data.  
* Keeps the public API minimal and focused.  

### Weaknesses & Edge Cases  

1. **Inconsistent Service Access** – Mixing static `ReferenceServiceImpl` methods with an instance obtained via `ServiceFactory` can lead to confusion and harder unit‑testing.  
2. **Raw Types** – The map returned by `RefCache` and the use of raw `Map` compromise type safety and can cause `ClassCastException` at runtime.  
3. **Null Handling** – `getCountryIsoCodeById` silently returns `null` if the ID is missing.  A caller might expect an exception or an empty string.  
4. **Logging Not Used** – The logger is instantiated but never called; any errors are simply propagated as generic `Exception`.  
5. **Locale Handling** – `locale.getLanguage()` can return an empty string if the locale is not fully specified, leading to an invalid language code lookup.  
6. **Exception Granularity** – All methods throw generic `Exception`, making it difficult for callers to react to specific failure modes.  

### Potential Enhancements  

* **Use Generics** – Replace raw `Map` with `Map<Integer, Country>` to gain compile‑time safety.  
* **Consistent Service Use** – Prefer dependency injection or a consistent factory pattern for all service look‑ups.  
* **Improve Error Handling** – Define custom checked exceptions (`CountryNotFoundException`, `ZoneNotFoundException`) or return `Optional<T>` where appropriate.  
* **Leverage Logging** – Add meaningful log statements for cache hits/misses, exceptions, and boundary conditions.  
* **Unit‑Testing** – Provide interfaces or adapters to allow mocking the cache and service layers.  
* **Documentation** – Add Javadoc comments to each method, clarifying return semantics and exception conditions.  

Overall, the class serves its purpose as a convenience façade but would benefit from modern Java practices (generics, proper exception handling, dependency injection) and clearer design consistency.

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

import java.util.Locale;
import java.util.Map;

import org.apache.log4j.Logger;

import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.service.reference.impl.ReferenceServiceImpl;

public class CountryUtil {

	private static Logger log = Logger.getLogger(CountryUtil.class);

	public static String getCountryIsoCodeById(int countryid) {

		Map countries = RefCache.getCountriesMap();
		Country c = (Country) countries.get(countryid);
		if (c != null) {
			return c.getCountryIsoCode2();
		} else {
			return null;
		}

	}

	public static Country getCountryByName(String countryname, int languageId)
			throws Exception {

		return ReferenceServiceImpl.getCountryByName(countryname, languageId);

	}

	public static CountryDescription getCountryByIsoCode(String countryIsoCode,
			Locale locale) throws Exception {


		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		return rservice.getCountryDescriptionByIsoCode(countryIsoCode,
				LanguageUtil.getLanguageNumberCode(locale.getLanguage()));

	}

	public static Zone getZoneCodeByCode(String name, Locale locale)
			throws Exception {

		return ReferenceServiceImpl.getZoneCodeByCode(name, locale);

	}

	public static Zone getZoneCodeByName(String name, int languageId)
			throws Exception {

		return ReferenceServiceImpl.getZoneCodeByName(name, languageId);

	}

}



```
