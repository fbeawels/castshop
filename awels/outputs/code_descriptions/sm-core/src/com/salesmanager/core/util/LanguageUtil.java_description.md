# LanguageUtil.java

## Review

## 1. Summary
`LanguageUtil` is a small helper class that performs basic language‑related lookups and parsing.  
It is used by the SalesManager core layer to:

| Method | Purpose |
|--------|---------|
| `parseLanguages(String)` | Splits a semi‑colon delimited string into a list of language codes. |
| `getLanguageNumberCode(String)` | Returns the numeric ID for a language code (falls back to **1**). |
| `getLanguageByCode(String)` | Returns a `Language` entity for a code. |
| `getLanguageStringCode(int)` | Returns the two‑letter code for a numeric language ID (falls back to English). |
| `getDefaultLanguage()` | Reads the default language code from the configuration (defaults to `"en"`). |

The class relies on a shared `RefCache` that stores two maps:

* **languageswithcode** – `Map<String, Language>` keyed by lower‑case code.  
* **languageswithindex** – `Map<Integer, Language>` keyed by numeric ID.

No design patterns beyond simple static utility helpers are used, and the code is tightly coupled to the `RefCache` singleton.

---

## 2. Detailed Description
### Core Flow
1. **Initialization** – Nothing is performed at class load time; all methods are pure static lookups.
2. **Runtime** – Each public method queries the `RefCache` maps and performs a simple lookup or transformation.  
   * `parseLanguages` tokenises the input string and returns the tokens as a `List`.
   * The *get* methods first sanity‑check the input (e.g., null checks), then retrieve from the cache, and finally apply a default if the key is missing.
3. **Cleanup** – No resources are opened or closed; the class is essentially stateless.

### Assumptions & Constraints
| Assumption | Impact |
|------------|--------|
| `RefCache.getLanguageswithcode()` / `getLanguageswithindex()` never return `null`. | Otherwise a `NullPointerException` would be thrown. |
| The cache maps are correctly populated before any lookup occurs. | If the cache is empty or stale the methods will silently fall back to defaults, possibly hiding configuration errors. |
| Language IDs are 1‑based and “1” is always English. | Hardcoded defaults reduce flexibility and make the code brittle if the default language changes. |
| The configuration key `core.system.defaultlanguage` exists and is a valid language code. | If missing, the default `"en"` is used – acceptable but silent. |

---

## 3. Functions/Methods

| Method | Signature | Purpose | Inputs | Outputs | Side‑Effects | Notes |
|--------|-----------|---------|--------|---------|--------------|-------|
| `parseLanguages` | `public static List parseLanguages(String langs)` | Tokenises a string into language codes. | `langs` – semi‑colon delimited string or `null`. | `List` of `String` (raw type). | None. | Uses legacy `StringTokenizer`; could be replaced with `String.split`. |
| `getLanguageNumberCode` | `public static int getLanguageNumberCode(String lang)` | Maps a code to its numeric ID, defaulting to 1. | `lang` – language code (`null` allowed). | `int` language ID. | None. | Magic number 1 is hard‑coded; better to use a constant. |
| `getLanguageByCode` | `public static Language getLanguageByCode(String lang)` | Retrieves the `Language` entity for a code. | `lang` – language code (`null` allowed). | `Language` or `null`. | None. | No caching beyond `RefCache`. |
| `getLanguageStringCode` | `public static String getLanguageStringCode(int lang)` | Reverse lookup: ID → two‑letter code. | `lang` – numeric language ID. | `String` code (defaults to English). | None. | Fallback to `Constants.ENGLISH_CODE`. |
| `getDefaultLanguage` | `public static String getDefaultLanguage()` | Reads default language from configuration. | None. | `String` code. | None. | Uses `PropertiesUtil.getConfiguration()` (not shown). |

**Reusable/Utility methods** – None beyond the static helpers above.

---

## 4. Dependencies
| Dependency | Type | Role |
|------------|------|------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads configuration for default language. |
| `com.salesmanager.core.constants.Constants` | Internal | Provides the constant `ENGLISH_CODE`. |
| `com.salesmanager.core.entity.reference.Language` | Internal | Domain object representing a language. |
| `com.salesmanager.core.service.cache.RefCache` | Internal | Singleton cache holding language maps. |
| `com.salesmanager.core.util.PropertiesUtil` | Internal | Supplies a `Configuration` instance. |

No platform‑specific APIs; all are pure Java SE or third‑party open‑source libraries.

---

## 5. Additional Notes

### Strengths
* **Simplicity** – Small, self‑contained utilities that are easy to understand.
* **Single responsibility** – Each method performs a single, well‑defined lookup.
* **Configuration‑driven defaults** – The default language can be overridden in a properties file.

### Weaknesses & Edge Cases
1. **Raw types** – The `List` returned by `parseLanguages` and the `Map` parameters are not generified, leading to unchecked casts and potential `ClassCastException` at runtime.
2. **Magic numbers** – Default language ID (`1`) and the fallback `"en"` are hard‑coded; changing the default language in the database would silently break the logic.
3. **Null safety** – The code assumes that `RefCache` will always provide non‑null maps. If the cache is not initialized, a `NullPointerException` will propagate.
4. **Legacy tokeniser** – `StringTokenizer` is outdated; `String.split` or a `Scanner` would be clearer.
5. **No logging** – When a lookup fails, the methods silently return a default; this may hide misconfigurations or data integrity problems.
6. **Thread‑safety** – While the class itself is stateless, the underlying `RefCache` must be thread‑safe. The code does not guard against concurrent modifications.
7. **Performance** – Each call performs a map lookup; if used heavily in tight loops, the overhead could be non‑trivial. Caching results locally might help.

### Possible Enhancements
* **Generics** – Refactor to use `List<String>` and `Map<String, Language>` / `Map<Integer, Language>` to eliminate unchecked casts.
* **Constants** – Extract default language ID and default code into a dedicated constants class (`LanguageUtil.DEFAULT_LANGUAGE_ID`, `LanguageUtil.DEFAULT_LANGUAGE_CODE`).
* **Error handling** – Throw a custom unchecked exception (`LanguageNotFoundException`) or return an `Optional<Language>` to make failure explicit.
* **Logging** – Log a warning when a lookup fails or when `null` is passed, aiding debugging.
* **Modern string parsing** – Replace `StringTokenizer` with `String.split("\\s*;\\s*")` to trim whitespace and avoid legacy APIs.
* **Immutable caching** – Ensure `RefCache` returns immutable maps or use `Collections.unmodifiableMap` to prevent accidental modification.
* **Unit tests** – Add comprehensive tests covering normal lookups, missing keys, null inputs, and malformed configuration.

By addressing the raw‑type usage, magic numbers, and adding defensive coding practices, `LanguageUtil` can become more robust, maintainable, and future‑proof.

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
import java.util.List;
import java.util.Map;
import java.util.StringTokenizer;

import org.apache.commons.configuration.Configuration;

import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.service.cache.RefCache;

public class LanguageUtil {

	public static List parseLanguages(String langs) {
		List lst = new ArrayList();
		if (langs != null) {
			StringTokenizer st = new StringTokenizer(langs, ";");
			while (st != null && st.hasMoreTokens()) {
				String lang = st.nextToken();
				lst.add(lang);
			}
		}
		return lst;
	}

	public static int getLanguageNumberCode(String lang) {
		if (lang == null) {
			return 1;
		}
		Map langmap = RefCache.getLanguageswithcode();
		Language l = (Language) langmap.get(lang.toLowerCase());
		if (l != null) {
			return l.getLanguageId();
		} else {
			return 1;
		}
	}

	public static Language getLanguageByCode(String lang) {
		if (lang == null) {
			return null;
		}
		Map langmap = RefCache.getLanguageswithcode();
		Language l = (Language) langmap.get(lang.toLowerCase());
		return l;
	}

	public static String getLanguageStringCode(int lang) {

		Map langmap = RefCache.getLanguageswithindex();
		Language l = (Language) langmap.get(lang);
		if (l != null) {
			return l.getCode();
		} else {
			return Constants.ENGLISH_CODE;
		}
	}

	public static String getDefaultLanguage() {

		Configuration conf = PropertiesUtil.getConfiguration();
		String defaultLang = conf
				.getString("core.system.defaultlanguage", "en");
		return defaultLang;

	}
}



```
