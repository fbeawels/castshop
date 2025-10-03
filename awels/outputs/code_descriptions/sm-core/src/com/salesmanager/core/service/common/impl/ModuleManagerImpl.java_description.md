# ModuleManagerImpl.java

## Review

## 1. Summary  

**Purpose** – `ModuleManagerImpl` is a static façade that centralises two sets of responsibilities:

1. **SSL / keystore configuration** – At class‑loading time the static block checks a property (`core.usekeystore`) and, if set, populates the JVM SSL system properties (`javax.net.ssl.*`).
2. **Service lookup & configuration parsing** – It exposes a handful of static lookup helpers that retrieve `CoreModuleService` objects for a given country, service code, sub‑service or module name. It also parses raw configuration strings into two DTOs (`IntegrationKeys` and `IntegrationProperties`).

**Key components**

| Component | Role |
|-----------|------|
| `implementations` | A raw `Map` intended to hold class name → implementation mappings (unused in the current excerpt). |
| `conf` | A `Configuration` instance obtained via `PropertiesUtil.getConfiguration()`. |
| `log` | A Log4J logger (never used). |
| `ServicesUtil` | External helper that provides a list of `CoreModuleService` objects for a country. |
| `CoreModuleService` | Entity representing a module/service. |
| `IntegrationKeys` / `IntegrationProperties` | DTOs used to hold parsed credential / property data. |

The code shows a mix of legacy Java APIs (`StringTokenizer`, raw collections) and modern patterns (static factory). No design pattern is explicitly used beyond the “service locator” idea of the lookup methods.

---

## 2. Detailed Description  

### Static initialisation
```java
static {
    boolean useKeyStore = conf.getBoolean("core.usekeystore", false);
    if (useKeyStore) {
        System.setProperty("javax.net.ssl.keyStore", conf.getString("core.keyStore"));
        System.setProperty("javax.net.ssl.keyStorePassword", conf.getString("core.keyStorePassword"));
        System.setProperty("javax.net.ssl.trustStore", conf.getString("core.trustStore"));
        System.setProperty("javax.net.ssl.trustStorePassword", conf.getString("core.trustStorePassword"));
    }
}
```
- Executed once when the class is loaded.
- Reads keystore information from the global `sm-core-config.properties`.
- Side‑effects: global JVM SSL configuration. This can have far‑reaching consequences for all code that uses SSL after this class is loaded.

### Service lookup methods

1. **`getModuleService(String countryIsoCode, int serviceCode, int subservice)`**  
   - Retrieves all services for the country via `ServicesUtil.getServices`.
   - Iterates and collects services that match both `serviceCode` and `subservice`.  
   - **Bug**: the method returns `null` instead of the constructed collection, so callers always get `null`.

2. **`getModuleService(String countryIsoCode, int serviceCode)`**  
   - Same as above but matches only the `serviceCode`.

3. **`getModuleServiceByCode(String countryIsoCode, String moduleName, int subservice)`**  
   - Returns the **first** service whose name matches and subtype matches.

4. **`getModuleServiceByCode(String countryIsoCode, String moduleName)`**  
   - Returns the first service whose name matches.

All four methods share the same pattern:  
- Call `ServicesUtil.getServices(countryIsoCode)` → returns a raw `List`.  
- Iterate with an `Iterator`, cast to `CoreModuleService`, compare fields, and collect or return.

### Credential / property parsing

Both parsing helpers (`stripCredentials`, `stripProperties`) use `StringTokenizer` with `;` as delimiter and fill DTOs. They assume a fixed ordering:

| Position | Credential | Property |
|----------|------------|----------|
| 1 | userId | properties1 |
| 2 | password | properties2 |
| 3 | transactionKey | properties3 |
| 4+ | key1/key2/key3 | properties4 (only 4th accepted) |

No validation is performed: empty strings, fewer tokens, or unexpected values are silently accepted.

### Dependencies & Assumptions

- **External utilities**: `PropertiesUtil`, `ServicesUtil` – expected to be available on the classpath.
- **Configuration**: Properties are fetched once and never refreshed.
- **Thread‑safety**: All methods are static and read‑only except the static block. The map and configuration are shared across threads, but no mutating operations are performed in the excerpt.
- **Generics**: The code uses raw types everywhere; generics are omitted, which can lead to `ClassCastException` at runtime.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑effects | Notes |
|--------|---------|------------|---------|--------------|-------|
| `getModuleService(String, int, int)` | Retrieve services matching `serviceCode` and `subservice`. | `countryIsoCode`, `serviceCode`, `subservice` | `Collection<CoreModuleService>` (should be collection but returns `null` due to bug) | None | **Bug** – always returns `null`. |
| `getModuleService(String, int)` | Retrieve services matching `serviceCode`. | `countryIsoCode`, `serviceCode` | `Collection<CoreModuleService>` | None | Returns empty list if none found. |
| `getModuleServiceByCode(String, String, int)` | Retrieve single service by name and subtype. | `countryIsoCode`, `moduleName`, `subservice` | `CoreModuleService` | None | Returns `null` if not found. |
| `getModuleServiceByCode(String, String)` | Retrieve single service by name. | `countryIsoCode`, `moduleName` | `CoreModuleService` | None | Returns `null` if not found. |
| `stripCredentials(String)` | Parse a semicolon‑delimited credential string into `IntegrationKeys`. | `configvalue` | `IntegrationKeys` | None | No decryption performed; expects at least 3 tokens. |
| `stripProperties(String)` | Parse a semicolon‑delimited properties string into `IntegrationProperties`. | `configvalue` | `IntegrationProperties` | None | Accepts up to 4 tokens. |
| `static` block | Configure SSL keystore at class load time. | None | None | Sets system properties if `core.usekeystore` is true. |

No helper or utility methods are defined outside of the public API. All operations are performed inline.

---

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| `org.apache.commons.configuration.Configuration` | Third‑party (Apache Commons Configuration) | For reading properties. |
| `org.apache.log4j.Logger` | Third‑party (Log4J) | Logger instance declared but never used. |
| `com.salesmanager.core.util.PropertiesUtil` | Internal | Provides `getConfiguration()`. |
| `com.salesmanager.core.entity.reference.CoreModuleService` | Internal | Domain entity. |
| `com.salesmanager.core.service.common.model.*` | Internal | DTOs. |
| `com.salesmanager.core.service.common.impl.ServicesUtil` | Internal | Supplies list of services per country. |

All dependencies are either open‑source or part of the same application. No platform‑specific APIs are used.

---

## 5. Additional Notes  

### Edge Cases & Missing Validation  

- **Empty or malformed config strings** – `stripCredentials`/`stripProperties` will silently set fields to `null` or empty; callers may not expect this.  
- **Missing service list** – If `ServicesUtil.getServices` returns `null`, the methods gracefully return `null`/empty list, but callers may not distinguish between “none found” and “error retrieving services”.  
- **Concurrent modifications** – The `implementations` map is never used, but if it were mutated elsewhere, lack of synchronization could cause race conditions.  
- **Return type mismatch** – The first `getModuleService` method’s contract (returning a collection) is violated; this is a critical bug that will break all callers expecting a collection.  

### Potential Enhancements  

1. **Fix the bug**: Replace `return null;` with `return returnList;`.  
2. **Use generics** everywhere (`Map<String, Class<?>>`, `List<CoreModuleService>`, etc.) to avoid unchecked casts.  
3. **Improve parsing**: Replace `StringTokenizer` with `String.split(";")`, perform length checks, and possibly support URL‑encoded values.  
4. **Optional decryption**: Add a pluggable encryption/decryption strategy rather than hard‑coded comments.  
5. **Caching**: Cache the result of `ServicesUtil.getServices` per country to avoid repeated lookups.  
6. **Logging**: Make use of the `log` instance to record missing services, malformed configs, or errors.  
7. **Unit tests**: Provide comprehensive tests covering all lookup scenarios and parsing edge cases.  
8. **Documentation**: Add Javadoc to each method, describing parameter expectations, possible null returns, and side‑effects.  
9. **Thread safety**: If the static map or configuration may be updated, guard with synchronization or use concurrent collections.  
10. **Decouple SSL configuration**: Expose a dedicated SSL configuration utility rather than performing it in a static block; this would give callers more control over when the properties are applied.

### Summary  

The class provides a straightforward service locator and configuration parser, but suffers from several code‑quality issues: raw types, missing validation, a critical bug in one method, and unused logging. Addressing these points will make the API more robust, easier to maintain, and safer in a multi‑threaded or multi‑module environment.

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
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.StringTokenizer;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.util.PropertiesUtil;

/**
 * Maintains a Map of implementation name - Class defined into
 * sm-core-config.properties
 * 
 * @author Carl Samson
 * 
 */
public class ModuleManagerImpl {

	private static Map implementations = new HashMap();
	private static Configuration conf = PropertiesUtil.getConfiguration();
	private static Logger log = Logger.getLogger(ModuleManagerImpl.class);

	private ModuleManagerImpl() {
	}

	static {

		// initialize keystore

		boolean useKeyStore = conf.getBoolean("core.usekeystore", false);

		if (useKeyStore) {

			System.setProperty("javax.net.ssl.keyStore", conf
					.getString("core.keyStore"));
			System.setProperty("javax.net.ssl.keyStorePassword", conf
					.getString("core.keyStorePassword"));
			System.setProperty("javax.net.ssl.trustStore", conf
					.getString("core.trustStore"));
			System.setProperty("javax.net.ssl.trustStorePassword", conf
					.getString("core.trustStorePassword"));

		}

	}

	public static Collection<CoreModuleService> getModuleService(
			String countryIsoCode, int serviceCode, int subservice) {

		List services = ServicesUtil.getServices(countryIsoCode);

		Collection returnList = new ArrayList();

		if (services != null) {

			Iterator i = services.iterator();
			while (i.hasNext()) {
				CoreModuleService srv = (CoreModuleService) i.next();
				if (srv.getCoreModuleServiceCode() == serviceCode
						&& srv.getCoreModuleServiceSubtype() == subservice) {
					returnList.add(srv);
					// return srv;
				}
			}
		}

		return null;
	}

	public static Collection<CoreModuleService> getModuleService(
			String countryIsoCode, int serviceCode) {

		List services = ServicesUtil.getServices(countryIsoCode);

		Collection returnList = new ArrayList();

		if (services != null) {

			Iterator i = services.iterator();
			while (i.hasNext()) {
				CoreModuleService srv = (CoreModuleService) i.next();
				if (srv.getCoreModuleServiceCode() == serviceCode) {
					// return srv;
					returnList.add(srv);
				}
			}
		}

		return returnList;

	}

	public static CoreModuleService getModuleServiceByCode(
			String countryIsoCode, String moduleName, int subservice) {

		List services = ServicesUtil.getServices(countryIsoCode);

		if (services != null) {

			Iterator i = services.iterator();
			while (i.hasNext()) {
				CoreModuleService srv = (CoreModuleService) i.next();
				if (srv.getCoreModuleName().equals(moduleName)
						&& srv.getCoreModuleServiceSubtype() == subservice) {
					return srv;
				}
			}
		}

		return null;

	}
	
	public static CoreModuleService getModuleServiceByCode(
			String countryIsoCode, String moduleName) {

		List services = ServicesUtil.getServices(countryIsoCode);

		if (services != null) {

			Iterator i = services.iterator();
			while (i.hasNext()) {
				CoreModuleService srv = (CoreModuleService) i.next();
				if (srv.getCoreModuleName().equals(moduleName)) {
					return srv;
				}
			}
		}

		return null;

	}

	public static IntegrationKeys stripCredentials(String configvalue)
			throws Exception {
		if (configvalue == null)
			return new IntegrationKeys();
		StringTokenizer st = new StringTokenizer(configvalue, ";");
		int i = 1;
		int j = 1;
		IntegrationKeys keys = new IntegrationKeys();
		while (st.hasMoreTokens()) {
			String value = st.nextToken();

			if (i == 1) {
				// decrypt
				keys.setUserid(value);
			} else if (i == 2) {
				// decrypt
				keys.setPassword(value);
			} else if (i == 3) {
				// decrypt
				keys.setTransactionKey(value);
			} else {
				if (j == 1) {
					keys.setKey1(value);
				} else if (j == 2) {
					keys.setKey2(value);
				} else if (j == 3) {
					keys.setKey3(value);
				}
				j++;
			}
			i++;
		}
		return keys;
	}

	/**
	 * Properties are 1) Production(1) - Test(2) 2) Pre-Auth(1) - Capture (2) -
	 * Sale (0) 3) No CCV (1) - With CCV (2)
	 * 
	 * @param configvalue
	 * @return
	 */
	public static IntegrationProperties stripProperties(String configvalue) {
		if (configvalue == null)
			return new IntegrationProperties();
		StringTokenizer st = new StringTokenizer(configvalue, ";");
		int i = 1;
		IntegrationProperties keys = new IntegrationProperties();
		while (st.hasMoreTokens()) {
			String value = st.nextToken();
			if (i == 1) {
				keys.setProperties1(value);
			} else if (i == 2) {
				keys.setProperties2(value);
			} else if (i == 3) {
				keys.setProperties3(value);
			} else {
				keys.setProperties4(value);
			}
			i++;
		}
		return keys;
	}

}



```
