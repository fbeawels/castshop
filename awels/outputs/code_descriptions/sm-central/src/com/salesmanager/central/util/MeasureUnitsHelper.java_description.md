# MeasureUnitsHelper.java

## Review

## 1. Summary  

`MeasureUnitsHelper` is a tiny utility class that exposes two static helper methods to translate a *size* or *weight* unit code (e.g., `"CM"` or `"LB"`) stored in the user's session/profile into a human‑readable description (e.g., `"centimeter"` or `"pound"`).  

Key components  
- **Context** – A custom profile object retrieved from the request or session.  
- **RefCache** – A cache that stores lookup tables for size and weight units.  
- **CentralMeasureUnits** – The entity that holds unit metadata (description, locale, etc.).  

The class does not depend on any external frameworks beyond the servlet API and the application‑specific packages (`com.salesmanager.*`). It simply performs a map lookup and returns a string.  

## 2. Detailed Description  

### Execution flow  
1. **`displaySizeUnitSymbol`**  
   * Reads the current `Context` from the request attributes.  
   * Obtains the unit code (`ctx.getSizeunit()`), defaulting to `"CM"` if missing.  
   * Pulls the map of size units from `RefCache`.  
   * Retrieves the `CentralMeasureUnits` instance for the code and (incorrectly) calls `setLocale` *before* checking for `null`.  
   * Returns the unit’s description or an empty string if not found.

2. **`displayWeightUnitSymbol`**  
   * Reads the `Context` from the HTTP session.  
   * Obtains the weight unit code (`ctx.getWeightunit()`), defaulting to `"LB"` if missing.  
   * Pulls the map of weight units from `RefCache`.  
   * Retrieves the `CentralMeasureUnits` instance and returns its description or an empty string.

### Design choices & constraints  
* **Static helpers** – The class is purely static; there is no state or dependency injection.  
* **Hard‑coded defaults** – `"CM"` and `"LB"` are hard‑coded; if business rules change these must be updated in code.  
* **Locale handling** – Only the size unit method attempts to set the locale on the unit entity; weight unit ignores the locale altogether.  
* **Null‑safety** – The code performs minimal null checks; a `NullPointerException` is possible if the map entry is missing or if the `Context` is null.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `public static String displaySizeUnitSymbol(HttpServletRequest req)` | Convert size unit code to description. | `HttpServletRequest` containing a `Context` attribute. | `String` description (empty if unknown). | Reads from session cache; **sets locale** on the retrieved unit (potentially NPE). |
| `public static String displayWeightUnitSymbol(HttpServletRequest req)` | Convert weight unit code to description. | `HttpServletRequest` containing a `Context` attribute in the session. | `String` description (empty if unknown). | Reads from session cache; no other side‑effects. |

**Reusable/utility aspects**  
The methods are straightforward lookups; they could be refactored into a single generic helper that accepts the map and default code as parameters.

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `javax.servlet.http.HttpServletRequest` | Standard | Servlet API. |
| `com.salesmanager.central.profile.Context` | Application | Holds user profile data. |
| `com.salesmanager.central.profile.ProfileConstants` | Application | Provides context attribute keys. |
| `com.salesmanager.core.entity.reference.CentralMeasureUnits` | Application | Entity representing a unit. |
| `com.salesmanager.core.service.cache.RefCache` | Application | Cache of unit lookup tables. |
| `java.util.Map`, `java.util.Locale` | Standard | Raw collections (no generics). |

No external third‑party libraries are used beyond the application's own packages.

## 5. Additional Notes  

### Edge cases & bugs  
* **NullPointerException** – In `displaySizeUnitSymbol`, `cmu.setLocale(req.getLocale())` is called before checking if `cmu` is null.  
* **Inconsistent context retrieval** – Size units read the context from request attributes; weight units read it from the session. This inconsistency can lead to hard‑to‑debug bugs.  
* **Locale misuse** – The weight unit method never uses the request locale, while the size unit method ignores the `locale` variable it creates.  
* **Raw maps** – The `Map` objects are not typed (`Map<String, CentralMeasureUnits>`). This leads to unchecked casts and can hide bugs.  
* **Hard‑coded defaults** – If the default unit changes (e.g., from `"CM"` to `"IN"`), the code must be updated.  
* **Thread‑safety** – The methods are static but only read from shared caches; no modification occurs, so thread safety is acceptable.  

### Suggested improvements  
1. **Null safety** – Move the `null` check before any method invocation on `cmu`.  
2. **Generic collections** – Declare `Map<String, CentralMeasureUnits>` to avoid unchecked casts.  
3. **Unified context access** – Decide whether the context lives in request attributes or the session and use it consistently.  
4. **Locale handling** – Either always set the locale on the unit or drop that logic entirely; make it consistent.  
5. **Parameterize defaults** – Pass the default unit code (or load it from configuration) instead of hard‑coding.  
6. **Extract common logic** – Create a generic helper method `lookupUnit(String unitCode, Map<String, CentralMeasureUnits> map, String defaultCode)` to avoid duplication.  
7. **Unit tests** – Add tests covering missing context, missing unit, and locale variations.  

Implementing these changes would make the helper more robust, easier to maintain, and less error‑prone.

## Code Critique



## Code Preview

```java
/*
 * Provided by CSTI Consulting 
 * Following GNU LESSER GENERAL PUBLIC LICENSE
 * You may obtain more details at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.central.util;

import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.reference.CentralMeasureUnits;
import com.salesmanager.core.service.cache.RefCache;

public class MeasureUnitsHelper {

	public static String displaySizeUnitSymbol(HttpServletRequest req) {

		Context ctx = (Context) req.getAttribute(ProfileConstants.context);

		String unit = ctx.getSizeunit();
		if (unit == null) {
			unit = "CM";
		}
		Map mapunits = (Map) RefCache.getSizeunits();

		// Locale locale = req.getLocale();
		String description;
		com.salesmanager.core.entity.reference.CentralMeasureUnits cmu = (com.salesmanager.core.entity.reference.CentralMeasureUnits) mapunits
				.get(unit);
		cmu.setLocale(req.getLocale());

		if (cmu == null) {
			description = "";
		} else {

			description = cmu.getDescription();
		}

		return description;

	}

	public static String displayWeightUnitSymbol(HttpServletRequest req) {

		Context ctx = (Context) req.getSession().getAttribute(
				ProfileConstants.context);

		String unit = ctx.getWeightunit();
		if (unit == null) {
			unit = "LB";
		}
		Map mapunits = (Map) RefCache.getWeightunits();

		Locale locale = req.getLocale();
		String description;
		CentralMeasureUnits cmu = (CentralMeasureUnits) mapunits.get(unit);

		if (cmu == null) {
			description = "";
		} else {

			description = cmu.getDescription();
		}

		return description;

	}

}



```
