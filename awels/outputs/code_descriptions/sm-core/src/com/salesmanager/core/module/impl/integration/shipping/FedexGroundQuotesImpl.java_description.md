# FedexGroundQuotesImpl.java

## Review

## 1. Summary  
`FedexGroundQuotesImpl` is a shipping‑quote provider that implements the **`ShippingQuotesModule`** interface for the FedEx Ground service.  
* **Purpose** – Return shipping options for FedEx Ground, expose a friendly method description, and persist/merge configuration settings specific to FedEx Ground.  
* **Key components**  
  * **`getShippingMethodDescription`** – Retrieves the UI label for the FedEx Ground module.  
  * **`getShippingQuote`** – Delegates to `FedexRequestQuotesImpl` to build the quote collection.  
  * **`getConfiguration`** – Processes configuration key/value pairs, populates a `ConfigurationResponse`, and stores the configuration in the response.  
  * **`storeConfiguration`** – Stub for persisting configuration (currently empty).  
* **Design** – Simple procedural style with a mix of legacy Java collections (raw types, `StringTokenizer`). No heavy frameworks are used; the implementation relies on `LabelUtil`, `FedexRequestQuotesImpl`, and the core Shipping constants.

---

## 2. Detailed Description  

### Flow of execution
1. **Initialization** – The class has a single `Logger` instance; no constructor logic is required.  
2. **Quote retrieval** –  
   * `getShippingQuote` is called by the shipping engine with the current order, packages, customer and store context.  
   * It creates a new `FedexRequestQuotesImpl` and calls its `getQuote` method, passing the title label, a hard‑coded service code (`"fedexground"`), the packages and other context objects.  
   * The returned collection is forwarded back to the caller.  
   * Any exception during this process is logged and `null` is returned.  
3. **Configuration processing** –  
   * `getConfiguration` examines each `MerchantConfiguration` record it receives.  
   * For **credential** records (`MODULE_SHIPPING_RT_CRED`) it extracts API keys and properties using `ShippingUtil`, adding them to the `ConfigurationResponse`.  
   * For **package/service** records (`MODULE_SHIPPING_RT_PKG_DOM_INT`) it:
     * Adds a plain string value for package options.  
     * Parses a semicolon‑delimited string of service codes into a `Map<String,String>` and stores it.  
   * Finally it adds the raw configuration to the response.  
4. **Persisting configuration** – `storeConfiguration` is a no‑op; in a production system it would write the configuration back to the database or a properties file.

### Assumptions & Constraints
* The method assumes that `FedexRequestQuotesImpl.getQuote` will always succeed and that its return type is a raw `Collection`.  
* Configuration keys are hard‑coded strings from `ShippingConstants`; no validation of unknown keys.  
* No thread‑safety guarantees – the class is effectively stateless except for the logger.  
* The class expects that all configuration values are non‑null when used; otherwise a `NullPointerException` may be thrown.

### Architecture & Design Choices
* The implementation is deliberately **thin** – it mainly orchestrates configuration and delegates quote computation.  
* Use of **legacy APIs** (`StringTokenizer`, raw `Map`) indicates that the surrounding codebase is older and may benefit from modernization.  
* The `storeConfiguration` stub shows an intention to follow a **Repository**‑style pattern, but the current code does not fully implement it.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑effects | Notes |
|--------|---------|------------|--------|--------------|-------|
| `getShippingMethodDescription(Locale)` | Return UI label for FedEx Ground | `locale` | `String` | None | Uses `LabelUtil` |
| `getShippingQuote(ConfigurationResponse, BigDecimal, Collection<PackageDetail>, Customer, MerchantStore, Locale)` | Retrieve shipping quotes | `config`, `orderTotal`, `packages`, `customer`, `store`, `locale` | `Collection<ShippingOption>` | None | Delegates to `FedexRequestQuotesImpl`; returns `null` on exception |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | Merge merchant config into response | `configurations`, `vo` | `ConfigurationResponse` | Adds credentials, properties, package/service maps to `vo` | Handles two specific config keys; uses `ShippingUtil` |
| `storeConfiguration(int, ConfigurationResponse, HttpServletRequest)` | Persist configuration | `merchantid`, `vo`, `request` | `void` | Currently no operation | Stub for future implementation |

**Reusable helpers**  
* `ShippingUtil.getKeys(String)` – parses credentials into an `IntegrationKeys` object.  
* `ShippingUtil.getProperties(String)` – parses properties into an `IntegrationProperties` object.  
These helpers are not defined in this file but are assumed to be part of the shared shipping utilities.

---

## 4. Dependencies  

| Library / API | Type | Purpose |
|---------------|------|---------|
| `org.apache.commons.lang.StringUtils` | Third‑party (Apache Commons Lang) | String null/blank checks |
| `org.apache.log4j.Logger` | Third‑party (Log4j) | Logging |
| `com.salesmanager.core.constants.ShippingConstants` | Project | Constants for config keys |
| `com.salesmanager.core.entity.*` | Project | Domain entities (`Customer`, `MerchantStore`, `PackageDetail`, `ShippingOption`) |
| `com.salesmanager.core.module.model.integration.ShippingQuotesModule` | Project | Interface implemented |
| `com.salesmanager.core.service.common.model.*` | Project | Configuration response / keys / properties |
| `com.salesmanager.core.service.merchant.ConfigurationResponse` | Project | Holds merged configuration |
| `com.salesmanager.core.util.LabelUtil` | Project | Localized text lookup |
| `javax.servlet.http.HttpServletRequest` | Servlet API | Request context for persistence (unused) |
| `FedexRequestQuotesImpl` | Project | External class performing quote calculation |
| `ShippingUtil` | Project | Helper for parsing credentials/properties |

*No platform‑specific or cloud‑specific dependencies are present.*  

---

## 5. Additional Notes  

### Strengths
* **Separation of concerns** – quote calculation is delegated to `FedexRequestQuotesImpl`.  
* **Use of constants** – configuration keys are pulled from a central constants class.  
* **Internationalization support** – labels and descriptions are fetched via `LabelUtil`.

### Weaknesses & Edge Cases  
1. **Raw collections** – `getQuote` returns a raw `Collection`; this loses type safety and may cause `ClassCastException` downstream.  
2. **Error handling** – `getShippingQuote` swallows all exceptions and returns `null`. Downstream code must handle `null` which can mask real problems.  
3. **Hard‑coded string parsing** – `StringTokenizer` is outdated; using `String.split(";")` would be clearer and less error‑prone.  
4. **Null checks** – The code assumes non‑null `configurations.getConfigurationValue*` when calling `StringUtils.isBlank`. If any of those are `null`, a `NullPointerException` will occur.  
5. **Missing persistence** – `storeConfiguration` is empty; the module cannot save its own configuration.  
6. **No validation** – Service codes and package options are accepted verbatim; malformed values could propagate errors to the shipping API.  
7. **Thread safety** – The class is effectively stateless but the logger is not `static`. While this is fine, making it `static` could reduce overhead.

### Suggested Enhancements
* Replace raw types with generics throughout (`Collection<ShippingOption>`, `Map<String,String>`).  
* Add robust error handling: log the stack trace and rethrow a custom `ShippingException` instead of returning `null`.  
* Validate configuration values before adding them to the response.  
* Implement `storeConfiguration` to persist the merged settings, perhaps delegating to a DAO or configuration service.  
* Modernize string parsing (`String.split`) and consider using `java.util.stream` for readability.  
* Make the logger `static final` and consider using SLF4J instead of Log4j (modern logging façade).  
* Add unit tests for each method, covering normal paths and failure scenarios.  

---  

**Overall assessment:**  
The class fulfills its basic contract and integrates with the rest of the shipping module. However, it relies on legacy patterns (raw collections, `StringTokenizer`) and lacks proper error handling and persistence logic. Modernizing the code and tightening its API contracts would improve reliability, maintainability, and testability.

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
package com.salesmanager.core.module.impl.integration.shipping;

import java.math.BigDecimal;
import java.util.Collection;
import java.util.HashMap;
import java.util.Locale;
import java.util.Map;
import java.util.StringTokenizer;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.shipping.PackageDetail;
import com.salesmanager.core.entity.shipping.ShippingOption;
import com.salesmanager.core.module.model.integration.ShippingQuotesModule;
import com.salesmanager.core.service.common.model.IntegrationKeys;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.util.LabelUtil;

public class FedexGroundQuotesImpl implements ShippingQuotesModule {

	private Logger log = Logger.getLogger(FedexGroundQuotesImpl.class);

	public String getShippingMethodDescription(Locale locale) {
		return LabelUtil.getInstance().getText(locale, "module.fedexground");
	}

	public Collection<ShippingOption> getShippingQuote(
			ConfigurationResponse config, BigDecimal orderTotal,
			Collection<PackageDetail> packages, Customer customer,
			MerchantStore store, Locale locale) {

		try {

			FedexRequestQuotesImpl impl = new FedexRequestQuotesImpl();

			Collection coll = impl.getQuote(LabelUtil.getInstance().getText(
					locale, "label.shipping.rates.fedexexground.title"), null,
					"fedexground", packages, orderTotal, config, store,
					customer, locale);
			return coll;

		} catch (Exception e) {
			log.error(e);
		}

		return null;

	}

	public ConfigurationResponse getConfiguration(
			MerchantConfiguration configurations, ConfigurationResponse vo)
			throws Exception {
		// TODO Auto-generated method stub
		if (configurations.getConfigurationKey().equals(
				ShippingConstants.MODULE_SHIPPING_RT_CRED)) {// handle
																// credentials

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {

				IntegrationKeys keys = ShippingUtil.getKeys(configurations
						.getConfigurationValue1());
				vo.addConfiguration("fedexground-keys", keys);
			}

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {

				IntegrationProperties props = ShippingUtil
						.getProperties(configurations.getConfigurationValue2());
				vo.addConfiguration("fedexground-properties", props);
			}

		}

		if (configurations.getConfigurationKey().equals(
				ShippingConstants.MODULE_SHIPPING_RT_PKG_DOM_INT)) {// handle
																	// packages
																	// &
																	// services
			Map domesticmap = null;
			Map globalmap = null;
			// PKGOPTIONS
			if (!StringUtils.isBlank(configurations.getConfigurationValue())) {
				vo.addConfiguration("package-fedexground", configurations
						.getConfigurationValue());
			}

			// global
			if (!StringUtils.isBlank(configurations.getConfigurationValue1())) {
				globalmap = new HashMap();
				String intl = configurations.getConfigurationValue1();
				StringTokenizer st = new StringTokenizer(intl, ";");
				while (st.hasMoreTokens()) {
					String token = st.nextToken();
					globalmap.put(token, token);
				}
				vo.addConfiguration("service-fedexground", globalmap);
			}
		}

		vo.addMerchantConfiguration(configurations);
		return vo;
	}

	public void storeConfiguration(int merchantid, ConfigurationResponse vo, HttpServletRequest request)
			throws Exception {
		// TODO Auto-generated method stub

	}

}



```
