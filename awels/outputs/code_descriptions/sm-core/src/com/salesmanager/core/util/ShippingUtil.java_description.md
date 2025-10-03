# ShippingUtil.java

## Review

## 1. Summary  
`ShippingUtil` is a pure‑utility class that contains a wide variety of static helper methods for handling shipping‑related configuration data in a Sales‑Manager application. The class is responsible for:

| Category | Responsibility |
|----------|----------------|
| **Key & Property construction** | `buildShippingKeyLine`, `buildShippingPropertiesLine` create semi‑colon separated strings from `IntegrationKeys`/`IntegrationProperties` objects. |
| **Configuration parsing** | `getConfigurationValuesMap`, `getConfigurationList`, `getConfigurationMap`, `getConfigurationLine` transform delimited strings into Java `Map`s or `List`s. |
| **Service / package mapping** | `buildServiceMapFromList`, `buildServiceMapByCode`, `buildServiceMapLabelByCode`, `buildServiceMap`, `buildPackageMap` build mappings of service IDs → labels or codes, pulling values from a `ResourceBundle` and the `ReferenceService`. |
| **Merchant configuration persistence** | `arrangeConfigurationsToSave` prepares a list of `MerchantConfiguration` objects ready for persistence. |
| **Zone/estimate handling** | `buildShippingPriceRegionMap` converts zone/estimate configuration strings into a `TreeMap<Integer, ShippingPriceRegion>` suitable for pricing calculations. |

The code relies on **Apache Commons Lang** for `StringUtils`, **Java ResourceBundle** for localisation, and the application’s own **ServiceFactory** and **ReferenceService** for database look‑ups. No external UI frameworks are involved.

---

## 2. Detailed Description  

### General Flow  
All methods are static, making the class effectively stateless and thread‑safe provided the underlying services (`ServiceFactory`, `ReferenceService`) are themselves thread‑safe. The typical workflow for a caller is:

1. **Read a configuration string** from a `MerchantConfiguration` or the `ReferenceService`.  
2. **Parse** it using one of the `getConfiguration*` helpers.  
3. **Transform** the result into a useful map or list (service ID → label, zone → price, etc.).  
4. **Persist** any changes by building a list of `MerchantConfiguration` objects with `arrangeConfigurationsToSave`.

### Key Design Choices  
- **Delimiters** – The code assumes a very specific syntax (e.g., `;` as a top‑level separator, `|` as an inner separator). This design keeps parsing logic straightforward but is brittle if the format changes.  
- **ResourceBundle** – Labels for services and packages are retrieved from bundle files (`moduleid.properties`) instead of hard‑coded strings, which supports localisation.  
- **Service lookup** – Service configurations are fetched via `ReferenceService` and fall back to a generic “XX” country if a country‑specific entry is missing.  

### Notable Implementation Details  

| Method | Highlights |
|--------|------------|
| `buildShippingKeyLine` | Concatenates userid, password, key1‑4 separated by `;`. |
| `buildShippingPropertiesLine` | **Bug** – uses `props.getProperties1()` twice; the second should be `getProperties2()`. |
| `getConfigurationValuesMap` | Parses a string like `<ID>|<label>;<ID>|<label>` and retrieves the *label* from a `ResourceBundle`. The logic is convoluted; the key is ignored after being used to lookup the label. |
| `getConfigurationMap` | Tokenises on two delimiters and returns a map of the *second* token per pair. |
| `getConfigurationLine` | Serialises a map back into `key|value;key|value` form. Order is non‑deterministic because a `HashMap` is used. |
| `buildServiceMap*` | All three methods share the same lookup logic but differ in whether the value is the service **code** or the **label**. |
| `buildPackageMap` | Similar to the service map, but for package options. |
| `arrangeConfigurationsToSave` | Updates or creates two `MerchantConfiguration` objects: one for credentials, one for packages and service lists. Uses `StringUtils.isBlank` for null/empty checks. |
| `buildShippingPriceRegionMap` | Very verbose parsing logic that converts zone/estimate configuration strings into `ShippingPriceRegion` objects. It manually handles tokenisation, index tracking and numeric parsing. |

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `buildShippingKeyLine(IntegrationKeys)` | Builds a semi‑colon separated key line from an `IntegrationKeys` object. | `keys` | `String` | None |
| `buildShippingPropertiesLine(IntegrationProperties)` | Builds a semi‑colon separated properties line. **Bug**: uses `properties1` twice. | `props` | `String` | None |
| `getConfigurationValuesMap(String, String, Locale)` | Parses `<ID>|<label>` style string, looks up label from bundle, returns map of key→label. | `packageline`, `moduleid`, `locale` | `Map` (raw) | None |
| `trimPostalCode(String)` | Removes all non‑alphanumeric characters from a postal code. | `postalCode` | `String` | None |
| `getConfigurationList(String)` | Splits a `;` delimited string into a `List`. | `serviceline` | `List` (raw) | None |
| `getConfigurationMap(String, String, String)` | Converts `<key><inner>value` pairs into a map. | `line`, `mainDelimiter`, `innerDelimiter` | `Map` (raw) | None |
| `getConfigurationLine(Map)` | Serialises a map into `key|value;...` string. | `map` | `String` | None |
| `buildServiceMapFromList(List, String, Locale)` | Builds a map of service ID → label from a list of IDs. | `servicelist`, `moduleid`, `locale` | `Map` (raw) | None |
| `buildServiceMapByCode(String, Locale)` | Builds a map of service ID → code from module configuration. | `moduleid`, `locale` | `Map` (raw) | Uses `ReferenceService` |
| `buildServiceMapLabelByCode(String, Locale)` | Builds a map of service ID → label from module configuration. | `moduleid`, `locale` | `Map` (raw) | Uses `ReferenceService` |
| `buildServiceMap(String, Locale)` | Builds a map of service ID → label from a module configuration string. | `moduleid`, `locale` | `Map` (raw) | Uses `ReferenceService` |
| `buildPackageMap(String, Locale)` | Builds a map of package ID → label from a module configuration string. | `moduleid`, `locale` | `Map` (raw) | Uses `ReferenceService` |
| `arrangeConfigurationsToSave(int, ConfigurationResponse, String, String, String, String, String, String)` | Creates/updates `MerchantConfiguration` objects for credentials and package/service settings. | `merchantid`, `originalconfig`, `moduleid`, `credentiallines`, `propertiesline`, `packageOption`, `servicelinedomestic`, `servicelineintl` | `List` (raw) | Mutates the returned list; no DB writes. |
| `buildShippingPriceRegionMap(String, String, String)` | Parses zone and estimate configuration strings into a sorted map of `ShippingPriceRegion` objects. | `countryIsoCode`, `zonesConfigurationLine`, `estimateConfigurationLine` | `Map` (TreeMap) | None |

### Reusable / Utility Methods  
- All parsing helpers (`getConfiguration*`, `getConfigurationLine`) are general enough to be reused by other modules if the delimiter convention is maintained.  
- `trimPostalCode` could be part of a broader `AddressUtil` class.

---

## 4. Dependencies  

| External | Version | Role |
|----------|---------|------|
| `org.apache.commons.lang.StringUtils` | Any 2.x | `isBlank` and other string helpers |
| `java.util.ResourceBundle` | JDK | Localised label look‑ups |
| `com.salesmanager.core.business.service.config.ServiceFactory` | App‑specific | Provides `ReferenceService` instances |
| `com.salesmanager.core.business.service.reference.ReferenceService` | App‑specific | Reads module‑configuration strings from the database |
| `com.salesmanager.core.business.repository.model.MerchantConfiguration` | App‑specific | Stores shipping configuration data |
| `com.salesmanager.core.business.repository.model.shipping.ShippingPriceRegion` | App‑specific | Holds zone / estimate pricing details |
| `com.salesmanager.core.business.repository.model.configuration.IntegrationKeys`, `IntegrationProperties` | App‑specific | Hold credentials / properties |
| `com.salesmanager.core.business.repository.model.configuration.ConfigurationResponse` | App‑specific | Represents an existing configuration state |

No UI frameworks, no logging libraries (apart from the commented‑out `log` references), no ORM utilities are imported.

---

## 4. Additional Notes  

### 4.1. Code Quality & Modernisation  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** – All `Map`, `List`, `Iterator` usages are raw. This produces unchecked‑cast warnings and forces callers to perform unsafe casts. | Clutters IDE warnings, can lead to `ClassCastException` at runtime. | Replace with generics (`Map<String, String>`, `List<MerchantConfiguration>`, etc.) and add appropriate `@SuppressWarnings("unchecked")` only where unavoidable. |
| **StringTokenizer** – The entire class relies on `StringTokenizer`, an old, low‑level API. | Harder to read, error‑prone (e.g., no trim, no handling of empty tokens). | Replace with `String.split()` or `Pattern`/`Matcher`. |
| **StringBuffer → StringBuilder** – All concatenation uses `StringBuffer`. | Synchronized unnecessarily, slightly slower. | Switch to `StringBuilder`. |
| **Bug in `buildShippingPropertiesLine`** – `props.getProperties1()` is appended twice. | Credential strings will miss the second property, leading to broken API calls. | Use `getProperties2()` for the second token. |
| **Missing null checks** – Methods such as `buildShippingKeyLine` and `arrangeConfigurationsToSave` assume non‑null arguments. | `NullPointerException` in production if a caller forgets to validate. | Add defensive checks or document pre‑conditions. |
| **Unordered maps** – `getConfigurationLine` uses a `HashMap` → order of key/value pairs in the output string is nondeterministic. | The string representation may vary between executions. | Use a `LinkedHashMap` when order matters or sort the keys before serialising. |
| **Hard‑coded constants** – `locale.getVariant().equals("EUR")` and `countryIsoCode != null` comparisons are brittle. | May break when new locales or country codes are added. | Replace with a more flexible country‑lookup strategy or expose a configuration map. |
| **Logging** – The class silently swallows parsing errors (e.g., `NumberFormatException`) inside `buildShippingPriceRegionMap`. | Silent failures may make debugging difficult. | Add proper logging (SLF4J) or propagate a custom checked exception. |
| **Date usage** – `new Date(new Date().getTime())` is unnecessarily verbose and uses legacy `java.util.Date`. | Minor performance overhead; future deprecation risk. | Use `java.time.Instant.now()` or `LocalDateTime.now()` in newer Java releases. |
| **Exception handling** – Several methods declare `throws Exception` but only use raw `Exception`. | Hard to pinpoint the real cause of a failure. | Declare more specific exceptions (e.g., `IOException`, `ConfigurationException`) or wrap them in a custom unchecked type. |
| **Unit‑testability** – No unit tests are provided, and the heavy reliance on static state (even though local) makes mocking the `ReferenceService` cumbersome. | Hard to maintain regression safety. | Write JUnit tests for each parsing helper; use dependency injection (e.g., `@FunctionalInterface` or passing a `Supplier<ReferenceService>`) to make the class more testable. |

### 4.2. Suggested Refactor Path  

1. **Generics first** – Convert every raw `Map`, `List`, `Iterator` into generics.  
2. **Modern string API** – Replace all `StringTokenizer` calls with `String.split()` or regex, and use `StringBuilder`.  
3. **Fix bugs** – Correct `buildShippingPropertiesLine` and review `getConfigurationValuesMap` for logical clarity.  
4. **Ordering guarantees** – Where output order matters (`getConfigurationLine`, service maps), use `LinkedHashMap` or explicitly sort keys before serialisation.  
5. **Logging & Exception propagation** – Replace commented‑out `log.error` placeholders with a proper SLF4J logger and propagate meaningful exceptions.  
6. **Documentation** – Add Javadoc to every public method, clearly stating the expected format of the input string and the meaning of the returned map.  
7. **Unit tests** – Implement tests for each helper, especially edge cases (empty strings, null inputs, malformed tokens).  
8. **Optional refactor** – If the configuration format can change in the future, consider encapsulating the delimiter logic into a small parser class or configuration‑format abstraction.  
9. **Date improvements** – Use the Java Time API (`Instant`, `ZoneId`) instead of `java.util.Date`.  

---

### 4.3. Final Take‑away  
`ShippingUtil` is functional and covers the full lifecycle of shipping configuration handling in the current codebase. However, the implementation suffers from a number of **legacy‑style** and **buggy** patterns that will make future maintenance difficult:

- Raw types → compile‑time warnings.  
- Manual tokenisation → fragile, hard to read.  
- Hard‑coded delimiters → brittle.  
- Silent error handling → hidden failures.  
- A known bug in property line generation.

Addressing these points will make the utility easier to understand, safer to use, and ready for modern Java development practices.

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
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.ResourceBundle;
import java.util.StringTokenizer;
import java.util.TreeMap;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.reference.ModuleConfiguration;
import com.salesmanager.core.entity.shipping.ShippingPriceRegion;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.reference.ReferenceService;

public class ShippingUtil {

	private ShippingUtil() {
	}

	public static String buildShippingKeyLine(IntegrationKeys keys) {
		StringBuffer keyLine = new StringBuffer();
		keyLine.append(keys.getUserid());
		if (keys.getPassword() != null) {
			keyLine.append(";");
			keyLine.append(keys.getPassword());
		}
		if (keys.getKey1() != null) {
			keyLine.append(";");
			keyLine.append(keys.getKey1());
		}
		if (keys.getKey2() != null) {
			keyLine.append(";");
			keyLine.append(keys.getKey2());
		}
		if (keys.getKey3() != null) {
			keyLine.append(";");
			keyLine.append(keys.getKey3());
		}
		if (keys.getKey4() != null) {
			keyLine.append(";");
			keyLine.append(keys.getKey4());
		}
		return keyLine.toString();
	}

	public static String buildShippingPropertiesLine(IntegrationProperties props) {
		StringBuffer keyLine = new StringBuffer();
		keyLine.append(props.getProperties1());
		if (props.getProperties1() != null) {
			keyLine.append(";");
			keyLine.append(props.getProperties1());
		}
		if (props.getProperties2() != null) {
			keyLine.append(";");
			keyLine.append(props.getProperties2());
		}
		if (props.getProperties3() != null) {
			keyLine.append(";");
			keyLine.append(props.getProperties3());
		}
		if (props.getProperties4() != null) {
			keyLine.append(";");
			keyLine.append(props.getProperties4());
		}
		if (props.getProperties5() != null) {
			keyLine.append(";");
			keyLine.append(props.getProperties5());
		}
		return keyLine.toString();
	}

	/**
	 * Strip a map of configuration if built as <ID>|<SHIPPING
	 * LABEL>;<ID>|<SHIPPING LABEL>
	 * 
	 * @param packageline
	 * @return
	 */
	public static Map getConfigurationValuesMap(String packageline,
			String moduleid, Locale locale) {

		ResourceBundle bundle = ResourceBundle.getBundle(moduleid, locale);

		Map returnmap = new HashMap();
		StringTokenizer st = new StringTokenizer(packageline, ";");
		while (st.hasMoreTokens()) {
			String token = st.nextToken();
			StringTokenizer stst = new StringTokenizer(token, "|");
			int i = 0;
			String key = null;
			while (stst.hasMoreTokens()) {
				String ptoken = stst.nextToken();
				if (i == 0) {
					key = ptoken;
				}
				// get value from bundle
				String value = bundle
						.getString("shipping.quote.services.label." + key);
				if (i == 1 && token.contains("|")) {
					returnmap.put(key, value);
				} else {
					returnmap.put(key, value);
				}
				i++;
			}
		}
		return returnmap;
	}

	public static String trimPostalCode(String postalCode) {

		String pc = postalCode.replaceAll("[^a-zA-Z0-9]", "");

		return pc;

	}

	/**
	 * Strip the list of configuration if buit as 1;2;3;4;5....
	 * 
	 * @param serviceline
	 * @return
	 */
	public static List getConfigurationList(String serviceline) {

		List returnlist = new ArrayList();
		StringTokenizer st = new StringTokenizer(serviceline, ";");
		while (st.hasMoreTokens()) {
			String token = st.nextToken();
			returnlist.add(token);
		}
		return returnlist;
	}

	/**
	 * Strip a map of configuration if built as <ID>|<VALUE>;<ID>|<VALUE>
	 * 
	 * @param packageline
	 * @return
	 */
	public static Map getConfigurationMap(String line, String mainDelimiter,
			String innerDelimiter) {

		Map returnmap = new HashMap();

		if (StringUtils.isBlank(line)) {
			return returnmap;
		}
		StringTokenizer st = new StringTokenizer(line, mainDelimiter);

		while (st.hasMoreTokens()) {
			String token = st.nextToken();
			StringTokenizer stst = new StringTokenizer(token, innerDelimiter);
			int i = 0;
			String key = null;
			while (stst.hasMoreTokens()) {
				String ptoken = stst.nextToken();
				if (i == 0) {
					key = ptoken;
				}
				if (i == 1) {
					returnmap.put(key, ptoken);
				}
				i++;
			}
		}
		return returnmap;
	}

	/**
	 * Build a line with <ID>;<VALUE>|<ID>;<VALUE>
	 * 
	 * @param map
	 * @return
	 */
	public static String getConfigurationLine(Map map) {

		StringBuffer returnLine = new StringBuffer();
		if (map != null) {
			Iterator i = map.keySet().iterator();
			int count = 1;
			while (i.hasNext()) {
				Object key = i.next();
				returnLine.append(key);
				returnLine.append("|");
				returnLine.append(map.get(key));
				if (count < map.size()) {
					returnLine.append(";");
				}
			}
		}
		return returnLine.toString();
	}

	/**
	 * Helper method for building a Map of services <id,name> from a list of
	 * services. It uses the resource bundles.
	 * 
	 * @param servicelist
	 * @param moduleid
	 * @param locale
	 * @return
	 * @throws Exception
	 */
	public static Map<String, String> buildServiceMapFromList(List servicelist,
			String moduleid, Locale locale) throws Exception {

		ResourceBundle bundle = ResourceBundle.getBundle(moduleid, locale);

		Map returnmap = new HashMap();

		if (servicelist == null) {
			return returnmap;
		}

		Iterator it = servicelist.iterator();
		while (it.hasNext()) {
			String pkgid = (String) it.next();
			String pkg = bundle.getString("shipping.quote.services.label."
					+ pkgid);
			returnmap.put(pkgid, pkg);
		}

		return returnmap;

	}

	/**
	 * Builds a Map<String,String> of services available (id, code) example
	 * 01-EXPRESS SHIPPING
	 * 
	 * @param serviceline
	 * @param moduleid
	 * @param locale
	 * @return
	 */
	public static Map<String, String> buildServiceMapByCode(String moduleid,
			Locale locale) throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);

		String country = locale.getCountry();
		if (locale.getVariant().equals("EUR")) {
			country = "X1";
		}

		ModuleConfiguration serviceconfig = rservice.getModuleConfiguration(
				moduleid, "service", country);

		if (serviceconfig == null) {
			serviceconfig = rservice.getModuleConfiguration(moduleid,
					"service", "XX");// generic
		}

		if (serviceconfig == null) {
			throw new Exception("ModuleConfiguration does not exist for "
					+ moduleid + "-service-XX-" + locale.getCountry());
		}

		String serviceline = serviceconfig.getConfigurationValue();

		Map returnmap = getConfigurationMap(serviceline, ";", "|");

		return returnmap;

	}

	/**
	 * Builds a Map<String,String> of services available (id, label example
	 * 01-EXPRESS SHIPPING
	 * 
	 * @param serviceline
	 * @param moduleid
	 * @param locale
	 * @return
	 */
	public static Map<String, String> buildServiceMapLabelByCode(
			String moduleid, Locale locale) throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);

		String country = locale.getCountry();
		if (locale.getVariant().equals("EUR")) {
			country = "X1";
		}

		ModuleConfiguration serviceconfig = rservice.getModuleConfiguration(
				moduleid, "service", country);

		if (serviceconfig == null) {
			serviceconfig = rservice.getModuleConfiguration(moduleid,
					"service", "XX");// generic
		}

		if (serviceconfig == null) {
			throw new Exception("ModuleConfiguration does not exist for "
					+ moduleid + "-service-XX-" + locale.getCountry());
		}

		String serviceline = serviceconfig.getConfigurationValue();

		Map amap = getConfigurationMap(serviceline, ";", "|");
		Map returnMap = new HashMap();

		ResourceBundle bundle = ResourceBundle.getBundle(moduleid, locale);

		if (amap != null) {
			Iterator i = amap.keySet().iterator();
			while (i.hasNext()) {
				String key = (String) i.next();
				String pkg = bundle.getString("shipping.quote.services.label."
						+ key);
				returnMap.put(key, pkg);

			}
		}

		return returnMap;

	}

	/**
	 * Builds a Map<String,String> of services avilables ID, CODE from the
	 * service .properties file
	 * 
	 * @param serviceline
	 * @param moduleid
	 * @param locale
	 * @return
	 */
	public static Map<String, String> buildServiceMap(String moduleid,
			Locale locale) throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);

		String country = locale.getCountry();
		if (locale.getVariant().equals("EUR")) {
			country = "X1";
		}

		ModuleConfiguration serviceconfig = rservice.getModuleConfiguration(
				moduleid, "service", country);

		if (serviceconfig == null) {
			serviceconfig = rservice.getModuleConfiguration(moduleid,
					"service", "XX");// generic
		}

		if (serviceconfig == null) {
			throw new Exception("ModuleConfiguration does not exist for "
					+ moduleid + "-service-XX-" + country);
		}

		String serviceline = serviceconfig.getConfigurationValue();

		ResourceBundle bundle = ResourceBundle.getBundle(moduleid, locale);

		List pkgids = getConfigurationList(serviceline);

		// List returnlist = new ArrayList();
		Map returnmap = new HashMap();

		Iterator it = pkgids.iterator();
		while (it.hasNext()) {
			String pkgid = (String) it.next();
			String pkg = bundle.getString("shipping.quote.services.label."
					+ pkgid);
			returnmap.put(pkgid, pkg);
		}

		// return returnlist;
		return returnmap;
	}

	/**
	 * Builds a Map<String,String> of package options ID, CODE from the service
	 * .properties file the package line must be built using
	 * <ID>|<SHIPPING_CODE>;<ID>|<SHIPPING_CODE>
	 * 
	 * @param packageline
	 * @param moduleid
	 * @param locale
	 * @return
	 */
	public static Map<String, String> buildPackageMap(String moduleid,
			Locale locale) throws Exception {

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);

		ModuleConfiguration serviceconfig = rservice.getModuleConfiguration(
				moduleid, "packages", locale.getCountry());

		if (serviceconfig == null) {
			serviceconfig = rservice.getModuleConfiguration(moduleid,
					"packages", "XX");// generic
		}

		if (serviceconfig == null) {
			throw new Exception("ModuleConfiguration does not exist for "
					+ moduleid + "-packages-XX-" + locale.getCountry());
		}

		String packageline = serviceconfig.getConfigurationValue();

		ResourceBundle bundle = ResourceBundle.getBundle(moduleid, locale);
		Map packsmap = getConfigurationMap(packageline, ";", "|");

		Map returnmap = new HashMap();

		Iterator it = packsmap.keySet().iterator();
		while (it.hasNext()) {
			String pkgid = (String) it.next();
			String label = bundle
					.getString("shipping.quote.packageoption.label." + pkgid);
			returnmap.put(pkgid, label);
		}
		return returnmap;
	}

	public static List<MerchantConfiguration> arrangeConfigurationsToSave(
			int merchantid, ConfigurationResponse originalconfig,
			String moduleid, String credentiallines, String propertiesline,
			String packageOption, String servicelinedomestic,
			String servicelineintl) {

		List modulestosave = new ArrayList();
		Date date = new Date(new Date().getTime());

		if (originalconfig != null) {
			// get credentials
			MerchantConfiguration credentials = originalconfig
					.getMerchantConfiguration(moduleid,
							ShippingConstants.MODULE_SHIPPING_RT_CRED);
			if (credentials != null) {
				credentials.setConfigurationValue1(credentiallines);
				credentials.setConfigurationValue2(propertiesline);
			} else {
				credentials = new MerchantConfiguration();
				credentials
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_CRED);
				credentials.setConfigurationModule(moduleid);
				credentials.setDateAdded(date);
				credentials.setMerchantId(merchantid);
				credentials.setConfigurationValue1(credentiallines);
				credentials.setConfigurationValue2(propertiesline);
			}
			credentials.setLastModified(date);
			modulestosave.add(credentials);

			// get packages

			// PACKAGE OPTION, DOMESTIC SERVICES, INTERNATIONAL SERVICES

			MerchantConfiguration pack = originalconfig
					.getMerchantConfiguration(moduleid,
							ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
			if (pack != null) {
				if (!StringUtils.isBlank(packageOption)) {
					pack.setConfigurationValue(packageOption);
				}
				if (!StringUtils.isBlank(servicelinedomestic)) {
					pack.setConfigurationValue1(servicelinedomestic);
				}
				if (!StringUtils.isBlank(servicelineintl)) {
					pack.setConfigurationValue2(servicelineintl);
				}
			} else {
				pack = new MerchantConfiguration();
				pack
						.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
				pack.setConfigurationModule(moduleid);
				pack.setDateAdded(date);
				if (!StringUtils.isBlank(packageOption)) {
					pack.setConfigurationValue(packageOption);
				}
				pack.setMerchantId(merchantid);
				if (!StringUtils.isBlank(servicelinedomestic)) {
					pack.setConfigurationValue1(servicelinedomestic);
				}
				if (!StringUtils.isBlank(servicelineintl)) {
					pack.setConfigurationValue2(servicelineintl);
				}
			}
			pack.setLastModified(date);
			modulestosave.add(pack);

		} else {// create both entries
			MerchantConfiguration credentials = new MerchantConfiguration();
			credentials
					.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_CRED);
			credentials.setConfigurationModule(moduleid);
			credentials.setDateAdded(date);
			credentials.setMerchantId(merchantid);
			credentials.setLastModified(date);
			credentials.setConfigurationValue1(credentiallines);
			credentials.setConfigurationValue2(propertiesline);
			modulestosave.add(credentials);

			MerchantConfiguration pack = new MerchantConfiguration();
			pack
					.setConfigurationKey(ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT);
			pack.setConfigurationModule(moduleid);
			pack.setDateAdded(date);
			pack.setLastModified(date);
			pack.setMerchantId(merchantid);
			if (!StringUtils.isBlank(packageOption)) {
				pack.setConfigurationValue(packageOption);
			}
			if (!StringUtils.isBlank(servicelinedomestic)) {
				pack.setConfigurationValue1(servicelinedomestic);
			}
			if (!StringUtils.isBlank(servicelineintl)) {
				pack.setConfigurationValue2(servicelineintl);
			}
			modulestosave.add(pack);
		}

		return modulestosave;

	}

	/**
	 * returns a map index - List 0 --- ShippingPriceRegion -------------String
	 * -------------String 1 --- ShippingPriceRegion -------------String
	 * -------------String
	 */

	public static Map buildShippingPriceRegionMap(String countryIsoCode,
			String zonesConfigurationLine, String estimateConfigurationLine)
			throws Exception {

		Map returnmap = new TreeMap();
		StringTokenizer cvtk = null;
		String countryline = null;
		int i = 1;

		if (!StringUtils.isBlank(zonesConfigurationLine)) {

			cvtk = new StringTokenizer(zonesConfigurationLine, "|");

			while (cvtk.hasMoreTokens()) {
				ShippingPriceRegion spr = null;
				if (returnmap.containsKey(i)) {
					spr = (ShippingPriceRegion) returnmap.get(i);
				} else {
					spr = new ShippingPriceRegion();
				}
				countryline = cvtk.nextToken();// maxpound:price,maxpound:price...|
				if (!countryline.equals("*")) {
					StringTokenizer countrystk = new StringTokenizer(
							countryline, ";");
					String country = null;
					StringBuffer countrline = new StringBuffer();
					while (countrystk.hasMoreTokens()) {
						country = countrystk.nextToken();
						if (countryIsoCode != null
								&& country.equals(countryIsoCode)) {

						}
						// now get maxpound and price
						spr.addCountry(country);
						countrline.append(country).append(";");
					}
					String line = countrline.toString();
					spr.setCountryline(line.substring(0, line.length() - 1));
				}
				returnmap.put(i, spr);
				i++;
			}

		}

		// estimate

		if (!StringUtils.isBlank(estimateConfigurationLine)) {

			cvtk = new StringTokenizer(estimateConfigurationLine, "|");// index:<MINCOST>;<MAXCOST>|
			countryline = null;
			i = 1;
			while (cvtk.hasMoreTokens()) {

				countryline = cvtk.nextToken();// index:<MINCOST>;<MAXCOST>

				StringTokenizer indextk = new StringTokenizer(countryline, ":");// index
				String configLine = null;
				int indexCount = 1;
				ShippingPriceRegion spr = null;
				while (indextk != null && indextk.hasMoreTokens()) {
					configLine = indextk.nextToken();

					if (indexCount == 1) {// countries
						try {
							int index = Integer.parseInt(configLine);
							spr = (ShippingPriceRegion) returnmap.get(index);
							if (spr != null) {
								spr.setEstimatedTimeEnabled(true);
							}
						} catch (Exception e) {
							// log.error("Cannot parse to an integer " +
							// configLine);
						}
					}
					if (indexCount == 2) {// days
						// parse dates <mindate>;<maxdate>
						StringTokenizer datetk = new StringTokenizer(
								configLine, ";");// date
						int dateCount = 1;
						while (datetk != null && datetk.hasMoreTokens()) {
							String date = (String) datetk.nextToken();

							try {

								if (spr != null) {
									if (dateCount == 1) {
										spr.setMinDays(Integer.parseInt(date));
									}
									if (dateCount == 2) {
										spr.setMaxDays(Integer.parseInt(date));
									}
								}

							} catch (Exception e) {
								// log.error("Cannot parse integer " + date);
							}

							dateCount++;
						}
					}
					indexCount++;
				}
				i++;
			}

		}

		return returnmap;

	}

}



```
