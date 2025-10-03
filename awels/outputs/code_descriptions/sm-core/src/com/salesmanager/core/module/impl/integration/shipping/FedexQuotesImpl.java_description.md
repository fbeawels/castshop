# FedexQuotesImpl.java

## Review

## 1. Summary  
**Purpose** – The `FedexQuotesImpl` class implements the `ShippingQuotesModule` interface for the **FedEx** shipping provider. It is responsible for:  
1. Returning a human‑readable description of the shipping method.  
2. Requesting shipping quotes from FedEx (via `FedexRequestQuotesImpl`), filtering them against the merchant’s configured services, and returning the final list of viable options.  
3. Loading and storing the merchant‑specific configuration values (credentials, package options, service codes).

**Key components**  
| Component | Role |
|-----------|------|
| `getShippingMethodDescription` | Provides the localized name of the FedEx module. |
| `getShippingQuote` | Orchestrates the quote request, filters by user selection, and logs unmatched services. |
| `getConfiguration` | Parses incoming `MerchantConfiguration` objects and populates a `ConfigurationResponse` with the necessary keys, properties, and service maps. |
| `storeConfiguration` | Stub (currently unimplemented). |

**Notable patterns / frameworks**  
- Uses a simple **factory‑like** approach to build a configuration map for FedEx.  
- Leverages **LabelUtil** for i18n, **IntegrationKeys/Properties** to abstract FedEx credentials.  
- Relies on **Apache Commons Lang** (`StringUtils`) and **Log4j** for logging.  
- No heavy frameworks; the code is mostly plain Java with a few utility helpers.

---

## 2. Detailed Description  
1. **Initialization**  
   - The class is instantiated once per request (likely via a dependency injection container).  
   - No heavy resources are created during construction; all heavy work occurs in methods.

2. **Quote Retrieval (`getShippingQuote`)**  
   - Instantiates `FedexRequestQuotesImpl` to request shipping rates.  
   - Calls `getQuote(...)` passing the title, service code `"fedex"`, packages, order total, merchant store, customer, and locale.  
   - The returned collection is expected to contain `ShippingOption` objects.  
   - It then fetches the merchant’s selected international services from the configuration (`service-intl-fedex`).  
   - Iterates over the returned options, building a `codeList` for logging and adding only those whose option code exists in the merchant’s selected services map.  
   - If none match, logs a warning with the unmatched codes.  
   - Returns the filtered collection or `null` if an exception occurs.

3. **Configuration Loading (`getConfiguration`)**  
   - Handles two distinct configuration keys:  
     * **`MODULE_SHIPPING_RT_CRED`** – loads FedEx credentials and properties into the response.  
     * **`MODULE_SHIPPING_RT_PKG_DOM_INT`** – loads package options and domestic/international service lists.  
   - Uses `StringTokenizer` to split semi‑colon separated values into maps of service codes.  
   - Adds these maps to the `ConfigurationResponse` under keys `service-dom-fedex` and `service-intl-fedex`.  
   - Always stores the original `MerchantConfiguration` in the response.

4. **Configuration Persistence (`storeConfiguration`)**  
   - Currently a no‑op; intended to persist any changes back to the data store.

5. **Cleanup**  
   - No explicit cleanup logic; resources are short‑lived and rely on GC.

**Assumptions / Constraints**  
- FedEx response codes are expected to be present in the merchant’s configuration map.  
- Configuration values are semi‑colon delimited strings.  
- Logging is sufficient for debugging; no exception is propagated to the caller (method swallows exceptions and returns `null`).  

**Architecture**  
The class follows a **service‑provider** pattern: each shipping provider implements `ShippingQuotesModule`. This allows the rest of the application to request quotes or configuration in a generic way without knowing the provider details.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `getShippingMethodDescription(Locale)` | Provides a localized description of the FedEx module. | `locale` | `String` (localized label) | None |
| `getShippingQuote(ConfigurationResponse, BigDecimal, Collection<PackageDetail>, Customer, MerchantStore, Locale)` | Requests quotes, filters by merchant selection, and logs mismatches. | `config`, `orderTotal`, `packages`, `customer`, `store`, `locale` | `Collection<ShippingOption>` or `null` | Logs via `LogMerchantUtil` |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | Parses a single merchant configuration entry and augments the response. | `configurations`, `vo` | `ConfigurationResponse` (updated) | Adds entries to `vo` |
| `storeConfiguration(int, ConfigurationResponse, HttpServletRequest)` | Persists configuration changes (currently unimplemented). | `merchantid`, `vo`, `request` | `void` | None |

### Reusable / Utility Methods  
- **`ShippingUtil.getKeys(String)`** – parses credentials string into `IntegrationKeys`.  
- **`ShippingUtil.getProperties(String)`** – parses properties string into `IntegrationProperties`.  
- **`LabelUtil.getInstance().getText(Locale, String)`** – retrieves i18n text.  

---

## 4. Dependencies  

| Dependency | Type | Usage |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String emptiness checks |
| `org.apache.log4j.Logger` | Third‑party | Logging |
| `com.salesmanager.core.util.*` | Internal | Localization, merchant logging |
| `com.salesmanager.core.module.model.integration.ShippingQuotesModule` | Internal | Interface implementation |
| `com.salesmanager.core.service.*` | Internal | Configuration handling |
| `java.util.*`, `java.math.BigDecimal` | Standard | Core data structures |
| `javax.servlet.http.HttpServletRequest` | Standard | HTTP request context (unused) |

**Platform Assumptions**  
- Running in a servlet container (due to `HttpServletRequest`).  
- `FedexRequestQuotesImpl` must be available and correctly configured.  

---

## 5. Additional Notes  

### Strengths  
- Clear separation of concerns: quote request, configuration loading, and persistence.  
- Uses i18n and logging utilities consistently.  
- The filtering logic is straightforward and easily maintainable.  

### Weaknesses / Edge Cases  
1. **Exception Handling** – Swallows all exceptions in `getShippingQuote`, logs only to `log.error`. The caller receives `null` without context. It would be better to propagate a custom exception or return an empty collection with a proper error message.  
2. **Null Checks** – The method assumes `coll` is non‑null; if `impl.getQuote` returns `null`, a `NullPointerException` could occur when checking `coll.size()`.  
3. **Hardcoded Service Code `"fedex"`** – The call to `impl.getQuote` uses `"fedex"` literally; if the service code ever changes, this would need manual refactor.  
4. **StringTokenizer** – Deprecated; consider `String.split(";")` or `StringTokenizer` replacement for better readability.  
5. **`storeConfiguration`** – Empty implementation may lead to configuration changes not persisting; should either be removed or documented.  
6. **Logging Format** – `LogMerchantUtil.log` is called only when *no* services match; if services match but some are filtered out, no log is produced, which might be confusing.  
7. **Thread‑Safety** – The class is stateless except for the logger; safe for concurrent use.  

### Potential Enhancements  
- **Return Detailed Errors** – Replace `null` returns with a structured error object.  
- **Configuration Validation** – Validate the format of `configurationValue1/2` before parsing to avoid runtime errors.  
- **Unit Tests** – Add tests covering all branches: matching services, no matches, malformed configuration, exception paths.  
- **Use Java 8+ Features** – Stream API for filtering, `Map.ofEntries` for immutable maps, `Optional` for safer null handling.  
- **Implement `storeConfiguration`** – Persist changes to the database or configuration store.  

Overall, the implementation achieves its primary goal but would benefit from improved error handling, modernization of string processing, and completion of the persistence method.

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
import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
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
import com.salesmanager.core.util.LogMerchantUtil;

public class FedexQuotesImpl implements ShippingQuotesModule {

	private Logger log = Logger.getLogger(FedexQuotesImpl.class);

	public String getShippingMethodDescription(Locale locale) {
		return LabelUtil.getInstance().getText(locale, "module.fedex");
	}

	public Collection<ShippingOption> getShippingQuote(
			ConfigurationResponse config, BigDecimal orderTotal,
			Collection<PackageDetail> packages, Customer customer,
			MerchantStore store, Locale locale) {

		try {

			FedexRequestQuotesImpl impl = new FedexRequestQuotesImpl();

			Collection coll = impl.getQuote(LabelUtil.getInstance().getText(
					locale, "label.shipping.rates.fedex.title"), null, "fedex",
					packages, orderTotal, config, store, customer, locale);

			// loop through the collection

			Collection returnColl = null;

			if (coll != null && coll.size()>0) {

				Map selectedintlservices = (Map) config
						.getConfiguration("service-intl-fedex");
				StringBuffer codeList = new StringBuffer();
				Iterator i = coll.iterator();
				while (i.hasNext()) {
					ShippingOption option = (ShippingOption) i.next();
					codeList.append(option.getOptionCode()).append("-");

					// filter against user selection
					if (selectedintlservices.containsKey(option.getOptionCode()
							.trim())) {

						if (returnColl == null) {
							returnColl = new ArrayList();
						}
						returnColl.add(option);
					}
				}

				if (coll.size() == 0) {
					LogMerchantUtil
							.log(
									store.getMerchantId(),
									" none of the service code returned by fedex ["
											+ codeList.toString()
											+ "] for this shipping is in your selection list");
				}
			}

			return returnColl;

		} catch (Exception e) {
			log.error(e);
		}

		return null;

	}

	public ConfigurationResponse getConfiguration(
			MerchantConfiguration configurations, ConfigurationResponse vo)
			throws Exception {
		if (configurations.getConfigurationKey().equals(
				ShippingConstants.MODULE_SHIPPING_RT_CRED)) {// handle
																// credentials

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {

				IntegrationKeys keys = ShippingUtil.getKeys(configurations
						.getConfigurationValue1());
				vo.addConfiguration("fedex-keys", keys);

			}

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {

				IntegrationProperties props = ShippingUtil
						.getProperties(configurations.getConfigurationValue2());
				vo.addConfiguration("fedex-properties", props);
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
				vo.addConfiguration("package-fedex", configurations
						.getConfigurationValue());
			}
			// domestic
			if (!StringUtils.isBlank(configurations.getConfigurationValue1())) {
				globalmap = new HashMap();
				String intl = configurations.getConfigurationValue1();
				StringTokenizer st = new StringTokenizer(intl, ";");
				while (st.hasMoreTokens()) {
					String token = st.nextToken();
					globalmap.put(token, token);
				}
				vo.addConfiguration("service-dom-fedex", globalmap);
			}
			// international
			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {
				globalmap = new HashMap();
				String intl = configurations.getConfigurationValue2();
				StringTokenizer st = new StringTokenizer(intl, ";");
				while (st.hasMoreTokens()) {
					String token = st.nextToken();
					globalmap.put(token, token);
				}
				vo.addConfiguration("service-intl-fedex", globalmap);
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
