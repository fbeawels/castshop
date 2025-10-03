# UpdateZones.java

## Review

## 1. Summary
`UpdateZones` is a tiny helper class that exposes two Ajax‑style endpoints for retrieving zone (state/province) information by country and language.  
- **`getZones(String, int)`** – returns an array of `Zone` objects for the supplied country, falling back to a default “N/A” zone when none are found.  
- **`updateZones(String, int)`** – behaves almost identically to `getZones` but also stores the selected country in the HTTP session.  

The class relies on a **configuration file** (`core.system.defaultcountryid`) for a fallback country, a **reference cache** (`RefCache`) to pull the zone list, and DWR’s `WebContextFactory` to access the servlet request.  The code is written in Java 1.5 style, using raw collections and manual array conversion.

---

## 2. Detailed Description
### Initialization
```java
private static Configuration conf = PropertiesUtil.getConfiguration();
private static int defaultcountryid = 223;
static {
    try {
        defaultcountryid = conf.getInt("core.system.defaultcountryid");
    } catch (Exception e) {
        log.error("Problem parsing default countryid");
    }
}
```
- The default country ID is read once on class load. If the configuration key is missing or malformed, the hard‑coded value **223** is retained.

### Runtime Flow
1. **Parsing the country id**  
   Both public methods attempt `Integer.parseInt` on the supplied string; failure triggers an error log but leaves `country` at its default value.

2. **Fetching zones**  
   `RefCache.getFilterdByCountryZones(country, languageId)` is called to obtain a `Collection<Zone>` (currently a raw `Collection`).  
   - If the collection is non‑null, it is copied into a `Zone[]` via `toArray`.
   - If null, a single dummy zone with name “N/A” and ID `0` is returned.

3. **Session side‑effect** (only in `updateZones`)  
   The country id is stored in the HTTP session under the key `"COUNTRY"` using DWR’s request wrapper.

4. **Return** – an array of `Zone` objects is always returned; callers must handle the empty‑array case.

### Cleanup
No explicit resource cleanup is required. All objects are short‑lived; the only persistent effect is the session attribute.

### Dependencies & Constraints
- **Configuration** – `org.apache.commons.configuration.Configuration`.  
- **Logging** – `org.apache.log4j.Logger`.  
- **Servlet context** – `uk.ltd.getahead.dwr.WebContextFactory`.  
- **Caching / service layer** – `RefCache` and `ReferenceService` (the latter is unused but instantiated).  
- **Assumptions** –  
  - The configuration key exists or the fallback `223` is acceptable.  
  - `RefCache.getFilterdByCountryZones` returns a non‑null collection or `null`; it never throws.  
  - The caller is tolerant of a dummy “N/A” zone.

---

## 3. Functions/Methods
| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects |
|--------|-----------|---------|--------|---------|--------------|
| `getZones` | `public Zone[] getZones(String countryId, int languageId)` | Retrieve zones for a country and language. | `countryId` (String, may be non‑numeric), `languageId` (int) | `Zone[]` – zone list or single “N/A” zone | Logs parse errors. |
| `updateZones` | `public Zone[] updateZones(String countryid, int languageid)` | Same as `getZones`, but also stores country in the HTTP session. | `countryid` (String), `languageid` (int) | `Zone[]` – zone list or single “N/A” zone | Sets session attribute `"COUNTRY"`, logs parse errors. |

*Utility/Helper methods* – none; duplicated logic could be extracted.

---

## 4. Dependencies
| Library | Scope | Remarks |
|---------|-------|---------|
| `org.apache.commons.configuration` | Third‑party | For loading the `core.system.defaultcountryid` property. |
| `org.apache.log4j` | Third‑party | Classic logging; no SLF4J abstraction. |
| `uk.ltd.getahead.dwr` | Third‑party | DWR WebContextFactory for servlet access. |
| `com.salesmanager.core` | Internal | Entities (`Zone`), caching (`RefCache`), and services (`ReferenceService`). |

No native Java EE APIs beyond `HttpServletRequest`.  The code is not platform‑specific beyond the DWR context.

---

## 5. Additional Notes & Recommendations
### Code Quality Issues
1. **Raw collections** – `Collection c` should be `Collection<Zone>`.  This eliminates unchecked casts and compiler warnings.  
2. **Redundant code** – The array conversion logic is duplicated.  Create a private helper method `toZoneArray(Collection<Zone>)`.  
3. **Unnecessary object creation** – `ReferenceService` is instantiated but never used; remove it or implement the intended logic.  
4. **Static mutable state** – `defaultcountryid` is static and mutable (though only set once).  Consider making it `final` or reading it lazily in a thread‑safe manner.  
5. **Logging** – Errors during parsing are logged but the message is generic; include the offending value for easier debugging.  
6. **Magic strings** – Session key `"COUNTRY"` is hard‑coded; use a constant.  
7. **Exception handling** – Swallowing `Exception` may mask serious problems.  Catch `NumberFormatException` only, or propagate a custom exception.  
8. **Thread safety** – The class itself is stateless except for the static config, so no concurrency issues.  However, the use of `RefCache` should be confirmed to be thread‑safe.

### Edge Cases
- **Null input** – Passing `null` for `countryId` will trigger a `NumberFormatException`; the method will log and fall back to the default.  Document this behaviour.  
- **Empty zone list** – If `RefCache` returns an empty collection instead of `null`, the current logic will return an empty array (size 0) – callers should be prepared for this.  
- **Session fixation** – Storing the country in the session is fine but ensure that session fixation protection is handled elsewhere.

### Future Enhancements
- **Internationalization** – The “N/A” zone name could be localized instead of hard‑coded.  
- **Caching strategy** – If zone data changes frequently, consider invalidating the cache or exposing a refresh endpoint.  
- **API Modernization** – Move away from raw arrays to `List<Zone>` or a JSON‑serialisable DTO; this would integrate better with modern REST or Ajax frameworks.  
- **Unit tests** – Write tests for parsing, cache interaction, and session handling.  
- **Logging framework** – Migrate to SLF4J with Logback for better abstraction.

Overall, the class performs its intended job but could be cleaned up for maintainability, type safety, and adherence to modern Java coding practices.

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
package com.salesmanager.core.util.www.ajax;

import java.util.Collection;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;

import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.PropertiesUtil;

public class UpdateZones {

	private static Logger log = Logger.getLogger(UpdateZones.class);
	private static Configuration conf = PropertiesUtil.getConfiguration();

	private static int defaultcountryid = 223;

	static {

		try {
			defaultcountryid = conf.getInt("core.system.defaultcountryid");
		} catch (Exception e) {
			log.error("Problem parsing default countryid");
		}

	}

	@SuppressWarnings("unchecked")
	public Zone[] getZones(String countryId, int languageId) {
		int country = defaultcountryid;
		try {
			country = Integer.parseInt(countryId);
		} catch (Exception e) {
			log.error(e);
		}
		Collection<Zone> c = RefCache.getFilterdByCountryZones(country,
				languageId);

		if (c != null) {
			Zone[] z = new Zone[c.size()];
			Zone[] znarray = (Zone[]) c.toArray(z);
			return znarray;
		} else {
			Zone[] z = new Zone[1];
			Zone zone = new Zone();
			zone.setZoneCountryId(country);
			zone.setZoneName("N/A");
			zone.setZoneId(0);
			z[0] = zone;
			return z;
		}
	}

	public Zone[] updateZones(String countryid, int languageid) {

		RefCache cache = RefCache.getInstance();

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();

		int country = defaultcountryid;

		try {
			country = Integer.parseInt(countryid);
		} catch (Exception e) {
			log.error(e);
		}

		req.getSession().setAttribute("COUNTRY", country);

		ReferenceService service = new ReferenceService();
		// Collection c = service.getZonesByCountry(country, req);
		Collection c = RefCache.getFilterdByCountryZones(country, languageid);

		if (c != null) {
			Zone[] z = new Zone[c.size()];
			Zone[] znarray = (Zone[]) c.toArray(z);
			return znarray;
		} else {
			Zone[] z = new Zone[1];
			Zone zone = new Zone();
			zone.setZoneCountryId(country);
			zone.setZoneName("N/A");
			zone.setZoneId(0);
			z[0] = zone;
			return z;
		}

	}

}



```
