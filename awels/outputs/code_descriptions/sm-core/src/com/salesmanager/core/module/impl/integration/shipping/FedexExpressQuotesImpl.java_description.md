# FedexExpressQuotesImpl.java

## Review

## 1. Summary

**Purpose & Functionality**  
`FedexExpressQuotesImpl` is a shipping‑quotes provider that integrates with FedEx Express. It implements the `ShippingQuotesModule` interface, providing:

1. A human‑readable description of the shipping method (`getShippingMethodDescription`).  
2. The ability to obtain shipping quotes for a given order (`getShippingQuote`).  
3. The ability to transform merchant‑specific configuration values into a `ConfigurationResponse` (`getConfiguration`).  
4. A placeholder for persisting configuration changes (`storeConfiguration`).

**Key Components**

| Component | Role |
|-----------|------|
| `FedexRequestQuotesImpl` | External helper that actually contacts the FedEx API and returns raw quote data. |
| `ConfigurationResponse` | Container for key/value pairs that the module consumes or produces. |
| `ShippingOption` | Representation of a single shipping option returned by the FedEx API. |
| `PackageDetail` | Describes a package’s dimensions/weight to be quoted. |
| `MerchantStore`, `Customer`, `Locale` | Context objects used for locale‑specific text and merchant‑specific data. |

**Notable Patterns & Libraries**

* Uses a *configuration‑driven* design – configuration values are stored as key/value strings, parsed and mapped into the module.  
* Employs the *Strategy* pattern: `ShippingQuotesModule` defines the interface; multiple implementations can coexist.  
* External dependencies: `org.apache.commons.lang.StringUtils`, `org.apache.log4j.Logger`, `com.salesmanager.core.*` domain/service utilities.

---

## 2. Detailed Description

### Execution Flow

1. **Description** – `getShippingMethodDescription` simply fetches a label from `LabelUtil`.  
2. **Quote Retrieval** –  
   * Instantiates `FedexRequestQuotesImpl` and calls `getQuote`.  
   * Receives a raw collection (`coll`) of `ShippingOption` objects.  
   * Filters the options: only those *not* present in the merchant’s selected international services map are retained.  
   * Logs if none of the returned codes match the merchant’s configuration.  
   * Returns the filtered collection (or `null` on failure).  
3. **Configuration Building** –  
   * Parses different configuration keys: credentials, package options, domestic/international service codes.  
   * Builds maps (e.g., `service-dom-fedexexpress`, `service-intl-fedexexpress`) from semicolon‑delimited strings.  
   * Adds these maps to the `ConfigurationResponse`.  
   * Stores the raw configuration for later reference (`addMerchantConfiguration`).  
4. **Persistence Stub** – `storeConfiguration` is not yet implemented.

### Assumptions & Constraints

* `FedexRequestQuotesImpl.getQuote` is trusted to return a collection of `ShippingOption` objects; the implementation assumes that all necessary data is present in the returned objects.  
* Configuration values are simple strings; semicolons separate codes. No validation of format beyond checking for emptiness.  
* Logging is performed via Log4j; no asynchronous or error‑reporting mechanisms beyond the logger.  
* The method `getShippingQuote` returns `null` if an exception occurs, rather than an empty collection or an error object.  

### Architecture & Design Choices

* **Raw Types** – The code frequently uses raw `Map`, `Collection`, and `Iterator`. This undermines type safety and generates compiler warnings.  
* **Error Handling** – A generic `catch (Exception e)` swallows all exceptions and logs them. The caller receives `null`, making it difficult to differentiate between “no quotes” and “system error.”  
* **Filtering Logic** – The filtering uses the *international* service map for comparison, even when domestic packages are queried – likely a bug.  
* **Configuration Parsing** – The class converts semicolon‑separated strings into maps that map a key to itself. A `Set` would be more natural.

---

## 3. Functions/Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `getShippingMethodDescription(Locale)` | Returns the localized shipping method name. | `Locale` | `String` | None |
| `getShippingQuote(ConfigurationResponse, BigDecimal, Collection<PackageDetail>, Customer, MerchantStore, Locale)` | Obtains quotes from FedEx, filters them, and returns the result. | *config*: merchant configuration<br>*orderTotal*: total amount of the order<br>*packages*: list of packages to ship<br>*customer*: customer entity<br>*store*: merchant store entity<br>*locale*: locale for text | `Collection<ShippingOption>` | Logs selected/filtered services; may alter no internal state |
| `getConfiguration(MerchantConfiguration, ConfigurationResponse)` | Parses merchant configuration values and populates a `ConfigurationResponse`. | *configurations*: key/value configuration object<br>*vo*: existing configuration response | `ConfigurationResponse` (modified) | Adds new keys/values; no other side‑effects |
| `storeConfiguration(int, ConfigurationResponse, HttpServletRequest)` | Persists configuration changes (currently a stub). | *merchantid*: merchant id<br>*vo*: updated configuration response<br>*request*: HTTP request context | `void` | None (not implemented) |

**Reusable / Utility Methods** – None; the class is self‑contained. The code heavily relies on external utilities (`FedexRequestQuotesImpl`, `LabelUtil`, `ShippingUtil`, `LogMerchantUtil`).

---

## 4. Dependencies

| External | Purpose | Status |
|----------|---------|--------|
| `org.apache.commons.lang.StringUtils` | String utility methods (`isBlank`) | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party |
| `com.salesmanager.core.*` | Domain entities (`Customer`, `MerchantStore`, `PackageDetail`, `ShippingOption`), utilities (`LabelUtil`, `LogMerchantUtil`, `ShippingUtil`) | Project internal |
| `javax.servlet.http.HttpServletRequest` | Web request context for storing configuration | Standard Java EE |

**Platform Specifics** – Assumes a Java EE container (servlet API) and Log4j configuration. No OS‑specific code.

---

## 5. Additional Notes & Recommendations

### 5.1 Edge Cases & Potential Failures

1. **Null Handling** –  
   * `coll` can be `null`; code checks but then `coll.size()` will throw a `NullPointerException`.  
   * `config.getConfiguration(...)` may return `null`; subsequent `containsKey` can throw `NullPointerException`.  

2. **Return Value Semantics** – Returning `null` on failure obscures the cause of failure for callers. It would be safer to return an empty collection or throw a custom checked exception.

3. **Filtering Logic** – The filtering uses `service-intl-fedexexpress` even when processing domestic services. This could filter out valid domestic options incorrectly.

4. **Duplicate Codes** – `codeList` concatenates codes separated by a dash, but does not trim the trailing dash; the log string ends with `-`.

5. **Type Safety** – Use of raw collections/iterators leads to unchecked warnings. All collections should be typed generically (e.g., `Collection<ShippingOption>`).

6. **Hardcoded Strings** – Magic strings such as `"fedexexpress"` or configuration keys should be constants.

### 5.2 Design Improvements

1. **Generics & Type Safety** – Replace raw `Collection`, `Map`, and `Iterator` with generics throughout.

2. **Refactor Configuration Parsing** – Use a helper method to parse semicolon‑delimited strings into a `Set<String>`; this simplifies the code and avoids mapping keys to themselves.

3. **Enhanced Error Handling** –  
   * Catch specific exceptions (`IOException`, `FedexException`, etc.) rather than `Exception`.  
   * Return a `Result` object or throw a custom exception to inform the caller.

4. **Logging** – Move logging into a dedicated helper that formats messages cleanly; consider using `LogMerchantUtil` consistently.

5. **Configuration Storage** – Implement `storeConfiguration` to persist changes (e.g., via a DAO or service layer).  

6. **Unit Tests** – Add tests for:
   * Successful quote retrieval and filtering.  
   * Handling of missing configuration keys.  
   * Null or empty input collections.  
   * Logging output (via Log4j appender capture).  

7. **Performance** – If the package list or returned options become large, consider streaming or parallel processing.

8. **Documentation** – Javadoc comments for public methods would aid future maintainers.

### 5.3 Future Extensions

* **International vs. Domestic Handling** – Separate methods or flags to explicitly request domestic or international quotes.  
* **Retry Logic** – Implement exponential backoff when contacting FedEx in case of transient failures.  
* **Configuration Validation** – Before using credentials or service codes, validate them against a known set.  
* **Cache** – Cache quotes for identical package configurations to reduce API calls.  
* **Multi‑Carrier Support** – Allow the module to delegate to other carriers based on service code patterns.

---

**Conclusion**  
The class provides a functional entry point for FedEx Express quotes but suffers from several code‑quality and robustness issues: raw types, generic error handling, potential bugs in filtering, and incomplete persistence. Addressing these concerns will make the module more reliable, maintainable, and easier to test.

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

public class FedexExpressQuotesImpl implements ShippingQuotesModule {

	private Logger log = Logger.getLogger(FedexExpressQuotesImpl.class);

	public String getShippingMethodDescription(Locale locale) {
		return LabelUtil.getInstance().getText(locale, "module.fedexexpress");
	}

	public Collection<ShippingOption> getShippingQuote(
			ConfigurationResponse config, BigDecimal orderTotal,
			Collection<PackageDetail> packages, Customer customer,
			MerchantStore store, Locale locale) {

		try {

			FedexRequestQuotesImpl impl = new FedexRequestQuotesImpl();

			Collection coll = impl.getQuote(LabelUtil.getInstance().getText(
					locale, "label.shipping.rates.fedexexpress.title"), null,
					"fedexexpress", packages, orderTotal, config, store,
					customer, locale);

			// loop through the collection

			Collection returnColl = null;

			if (coll != null) {

				Map selectedintlservices = (Map) config
						.getConfiguration("service-intl-fedexexpress");
				StringBuffer codeList = new StringBuffer();
				Iterator i = coll.iterator();
				while (i.hasNext()) {
					ShippingOption option = (ShippingOption) i.next();
					codeList.append(option.getOptionCode()).append("-");
					if (!selectedintlservices.containsKey(option
							.getOptionCode())) {

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
				vo.addConfiguration("fedexexpress-keys", keys);

			}

			if (!StringUtils.isBlank(configurations.getConfigurationValue2())) {

				IntegrationProperties props = ShippingUtil
						.getProperties(configurations.getConfigurationValue2());
				vo.addConfiguration("fedexexpress-properties", props);
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
				vo.addConfiguration("package-fedexexpress", configurations
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
				vo.addConfiguration("service-dom-fedexexpress", globalmap);
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
				vo.addConfiguration("service-intl-fedexexpress", globalmap);
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
