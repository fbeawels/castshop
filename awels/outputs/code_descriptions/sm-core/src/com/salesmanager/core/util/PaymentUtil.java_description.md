# PaymentUtil.java

## Review

## 1. Summary  
`PaymentUtil` is a small, but heavily‑used helper class that sits in the **core** layer of a sales‑management system.  
Its primary responsibilities are:  

| Responsibility | Where it lives | What it does |
|----------------|---------------|--------------|
| **Detect** a credit‑card type payment module | `isPaymentModuleCreditCardType()` | Checks a list of all payment modules, looking for a module that has the service code = 2 and subtype = 1 (the convention for credit‑card modules). |
| **Build** a map of active payment methods for a merchant | `getPaymentMethods()` | Queries the reference and merchant services to retrieve payment configuration, translates it into a `PaymentMethod` domain object, enriches it with internationalised text from a `ResourceBundle`, and returns a map keyed by the payment module name. |

The class relies on a handful of **third‑party** libraries:  
* Apache Commons Lang (`StringUtils`)  
* Log4J (`Logger`)  
* The project’s own `ServiceFactory` and a set of domain/value‑object classes (`PaymentMethod`, `CoreModuleService`, etc.).

Design patterns that surface in this code are:  

* **Factory** – the static `ServiceFactory.getService(...)` calls.  
* **DAO‑like** – the `ReferenceService` and `MerchantService` are queried in a read‑only fashion.  
* **Configuration‑object** – the `MerchantConfiguration` entries are parsed into `IntegrationProperties` and finally into a `PaymentMethod`.

---

## 2. Detailed Description  

### Flow of execution

1. **`isPaymentModuleCreditCardType`**  
   * Obtains a `PaymentService` instance.  
   * Calls `getPaymentMethods()` on that service, which returns a list of `CoreModuleService` objects.  
   * Iterates over the list looking for an exact match on `coreModuleName` and the credit‑card identifiers.  
   * Returns `true` if found, otherwise `false`.  

2. **`getPaymentMethods`**  
   * Creates an empty `Map` called `payments`.  
   * Loads a `ResourceBundle` named *modules* for the supplied `Locale`.  
   * Uses `ReferenceService` to obtain a map of payment methods that are supported for the merchant’s country.  
   * Calls `MerchantService.getConfiguration()` with a `ConfigurationRequest` that filters for payment‑module related keys.  
   * Iterates over each `MerchantConfiguration` entry and:
     * If the key is `MODULE_PAYMENT_INDICATOR_NAME` it treats the entry as a “payment module” (single payment or group).  
       * Builds or re‑uses a `PaymentMethod`, populates its properties (image, name, description, enabled flag), and stores it in `payments`.  
     * If the key contains `MODULE_PAYMENT_GATEWAY` it treats the entry as a gateway module, adds CVV‑related config, sets type `1`, and stores it.  
     * If the key contains `MODULE_PAYMENT` (but not indicator) it treats the entry as a single payment module, sets type `0`, stores key/value configs, and stores it.  
   * After processing all entries, the map `payments` contains all `PaymentMethod` objects keyed by module name, but may also contain disabled modules.  
   * The method then filters out disabled modules into a new `paymentMethods` map and returns it.

### Assumptions / Constraints  

* The `ServiceFactory` is a globally accessible singleton that returns concrete service implementations.  
* `PaymentConstants` contains static key names that must match the database columns.  
* The country lookup (`CountryUtil.getCountryByIsoCode`) and reference service are guaranteed to return a non‑null map of module information.  
* The `ResourceBundle` “modules” file must exist for the supplied locale; if it does not, the code silently ignores any locale‑specific strings.  
* All configuration values are stored as strings in the database; numeric values are compared as strings (e.g., `"true"`).  

### Architecture / Design Choices  

* **Separation of concerns**: `PaymentUtil` does not perform persistence; it delegates to `ReferenceService` and `MerchantService`.  
* **No state**: All methods are static, making the class stateless.  
* **Tight coupling to the service layer**: The use of `ServiceFactory.getService()` makes unit‑testing difficult without a mocking framework.  
* **Use of raw collections**: All maps and lists are declared without generics, leading to unchecked casts and potential `ClassCastException`s at runtime.  
* **Hard‑coded logic**: The credit‑card detection logic is hard‑coded to service code = 2 and subtype = 1, which is brittle if the underlying system changes.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects | Notes |
|--------|---------|------------|--------|--------------|-------|
| `isPaymentModuleCreditCardType(String paymentModule)` | Determines if a module is a credit‑card type. | `paymentModule` – the module name to check. | `boolean` – `true` if credit‑card, `false` otherwise. | Calls `PaymentService.getPaymentMethods()`. | No external side‑effects. |
| `getPaymentMethods(int merchantId, Locale locale)` | Builds a map of active `PaymentMethod` objects for a merchant. | `merchantId` – the merchant’s database id. <br> `locale` – for localisation. | `Map<String, PaymentMethod>` – key = module name. | Calls `ReferenceService`, `MerchantService`, loads a `ResourceBundle`. | Heavy I/O; should be cached for performance. |

### Utility / Helper Usage  
* `StringUtils.isBlank()` – defensive string checks.  
* `MerchantConfigurationUtil.getIntegrationProperties()` – parses semi‑colon separated key/value pairs into an `IntegrationProperties` object.  
* `CountryUtil.getCountryByIsoCode()` – fetches the country entity for localisation.

---

## 4. Dependencies  

| External / Third‑Party | Library / Framework | Purpose |
|------------------------|---------------------|---------|
| Apache Commons Lang | `StringUtils` | Simple string checks. |
| Log4J | `Logger` | Logging. |
| ServiceFactory | Internal | Simple service locator pattern. |
| Domain / Value Objects | `PaymentMethod`, `CoreModuleService`, `MerchantConfiguration`, `IntegrationProperties`, etc. | Business data representation. |
| `java.util` collections | `Map`, `List`, `Set` | Generic containers. |
| `java.util.ResourceBundle` | I18n | Loads module labels / texts. |

All dependencies are **standard** or **internal** to the SalesManager codebase, except for the two Apache Commons/Log4J libraries which are widely used.

---

## 5. Additional Notes  

### Strengths  

* **Clear separation** between configuration retrieval and the building of domain objects.  
* Uses **internationalisation** via `ResourceBundle`.  
* Straightforward **flag logic** for enabling/disabling modules.  

### Weaknesses / Edge Cases  

1. **Raw types** – All collections are declared without generics. This leads to unchecked casts (e.g., `(PaymentMethod) payments.get(...)`) and can mask type errors.  
2. **Null‑Pointer Risks** –  
   * `m.getConfigurationModule()` or `m.getConfigurationValue1()` may be `null` and are used as map keys.  
   * `CountryUtil.getCountryByIsoCode()` might return `null`, causing a `NullPointerException` when `countryDescription.getId()` is accessed.  
   * The `ResourceBundle` lookup is wrapped in a try/catch but the catch is empty – failures silently swallow localisation strings.  
3. **Hard‑coded logic** – Credit‑card detection is hard‑coded; if the code or subtype values change this will silently fail.  
4. **Performance** – The method performs multiple round‑trips to services and builds several intermediate maps. Caching per merchant/locale would reduce load on the database and services.  
5. **Testability** – Static methods and direct `ServiceFactory` calls make unit testing difficult; consider injecting services or using a façade.  
6. **Logging** – Only one error log statement (`Cannot load ResourceBundle checkout.properties`) but the actual property file is called “modules”; this is likely a copy‑paste error.  

### Suggested Improvements  

| Category | Recommendation |
|----------|----------------|
| **Generics** | Replace raw `Map`/`List` with typed generics (`Map<String, PaymentMethod>`, `List<CoreModuleService>`, etc.). |
| **Null Safety** | Add defensive checks (`Objects.requireNonNull`) or use `Optional` where appropriate. |
| **Refactoring** | Split the large `getPaymentMethods` into smaller, well‑named private methods: `loadModules()`, `buildPaymentMethodFromIndicator()`, `buildGatewayMethod()`, etc. |
| **Caching** | Cache the resulting map per `(merchantId, locale)` pair for a configurable TTL. |
| **Configuration** | Externalise the credit‑card service code and subtype via a constants file or database, rather than hard‑coding. |
| **Logging / Exception Handling** | Log exceptions from bundle lookup and configuration parsing; propagate them instead of swallowing silently. |
| **Unit Tests** | Introduce dependency injection (e.g., via constructor or setter) to mock services, and write tests covering each branch of the logic. |

---

### Final Verdict  

`PaymentUtil` fulfills its role of bridging service data and domain objects, but it suffers from legacy Java patterns (raw types, static façade) that hamper maintainability, testability, and safety. By modernising the codebase—introducing generics, dependency injection, clearer separation of concerns, and better error handling—the class would become far more robust and easier to evolve.

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

import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.ResourceBundle;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.payment.PaymentMethod;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.model.IntegrationProperties;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.payment.PaymentService;
import com.salesmanager.core.service.reference.ReferenceService;

public class PaymentUtil {

	private static Logger log = Logger.getLogger(PaymentUtil.class);

	public static boolean isPaymentModuleCreditCardType(String paymentModule)
			throws Exception {

		PaymentService paymentService = (PaymentService) ServiceFactory
				.getService(ServiceFactory.PaymentService);

		List payments = paymentService.getPaymentMethods();

		if (payments != null) {
			Iterator i = payments.iterator();
			while (i.hasNext()) {
				CoreModuleService cms = (CoreModuleService) i.next();
				if (cms.getCoreModuleName().equals(paymentModule)) {
					if (cms.getCoreModuleServiceCode() == 2
							&& cms.getCoreModuleServiceSubtype() == 1) {
						return true;
					}
				}
			}
		}

		return false;
	}

	public static Map<String, PaymentMethod> getPaymentMethods(int merchantId,
			Locale locale) throws Exception {

		Map payments = new HashMap();

		ResourceBundle bundle = ResourceBundle.getBundle("modules", locale);
		if (bundle == null) {
			log.error("Cannot load ResourceBundle checkout.properties");
		}

		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);

		CountryDescription countryDescription = CountryUtil
				.getCountryByIsoCode(locale.getCountry(), locale);

		Map modules = new HashMap();

		if (countryDescription != null) {
			modules = rservice.getPaymentMethodsMap(countryDescription.getId()
					.getCountryId());
		}

		ConfigurationRequest requestvo = new ConfigurationRequest(merchantId,
				true, PaymentConstants.MODULE_PAYMENT);
		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		ConfigurationResponse responsevo = mservice.getConfiguration(requestvo);
		List config = responsevo.getMerchantConfigurationList();

		if (config != null) {
			Iterator it = config.iterator();
			while (it.hasNext()) {

				MerchantConfiguration m = (MerchantConfiguration) it.next();
				


				String key = m.getConfigurationKey();
				if (key.equals(PaymentConstants.MODULE_PAYMENT_INDICATOR_NAME)) {// module
																					// configured

					// if(m.getConfigurationValue().equals("true")) {

					PaymentMethod method = null;
					// try to retreive the module first
					if (payments.containsKey(m.getConfigurationValue1())) {

						method = (PaymentMethod) payments.get(m
								.getConfigurationValue1());

					} else {

						method = new PaymentMethod();

					}
					
					if(m.getConfigurationValue()!=null && m.getConfigurationValue().equals("true")) {
						//payments.remove(m
						//		.getConfigurationValue1());
						//continue;
						method.setEnabled(true);
					}

					CoreModuleService cms = (CoreModuleService) modules.get(m
							.getConfigurationValue1());
					if (cms != null) {
						method.setPaymentImage(cms
								.getCoreModuleServiceLogoPath());
					}

					method.setPaymentModuleName(m.getConfigurationValue1());
					if (bundle != null) {
						try {
							String label = bundle.getString("module."
									+ m.getConfigurationValue1());
							if (StringUtils.isBlank(label)) {
								label = "";
							}
							method.setPaymentMethodName(label);
							String text = bundle
									.getString("module.paymenttext."
											+ m.getConfigurationValue1());
							method.setPaymentModuleText(text);
						} catch (Exception e) {
						}
					}
					if (m.getConfigurationValue()!=null && m.getConfigurationValue().equals("true")) {
						method.setEnabled(true);
					}

					payments.put(m.getConfigurationValue1(), method);
					continue;
				}

				if (key.contains(PaymentConstants.MODULE_PAYMENT_GATEWAY)) {// gateway
																			// module

					PaymentMethod method = null;
					// try to retreive the module first
					if (payments.containsKey(m.getConfigurationModule())) {

						method = (PaymentMethod) payments.get(m
								.getConfigurationModule());

					} else {

						method = new PaymentMethod();

					}
					
					IntegrationProperties props = null;
					/** ASSUMING PROPERTIES ARE IN CONFIGURATION_VALUE 2 **/
					if (!StringUtils.isBlank(m.getConfigurationValue2())) {
						props = MerchantConfigurationUtil.getIntegrationProperties(m.getConfigurationValue2(), ";");
					}
					
					

					if (props != null && props.getProperties3()!=null && props.getProperties3().equals("2")) {// use
																		// cvv
						method.addConfig("CVV", "true");

					}

					// core_modules_services subtype
					method.setType(1);
					payments.put(m.getConfigurationModule(), method);
					continue;

				}

				if (key.contains(PaymentConstants.MODULE_PAYMENT)) {// single
																	// payment
																	// module

					PaymentMethod method = null;
					// try to retreive the module first
					if (payments.containsKey(m.getConfigurationModule())) {

						method = (PaymentMethod) payments.get(m
								.getConfigurationModule());

					} else {

						method = new PaymentMethod();

					}

					// core_modules_services subtype
					method.setType(0);
					method.addConfig("key", m.getConfigurationValue());
					method.addConfig("key1", m.getConfigurationValue1());
					method.addConfig("key2", m.getConfigurationValue2());

					payments.put(m.getConfigurationModule(), method);
					continue;

				}

			}
		}
		
		
		Set entries = payments.keySet();
		
		Map paymentMethods = new HashMap();
		
		for(Object o: entries) {
			String key = (String)o;
			
			PaymentMethod pm = (PaymentMethod)payments.get(key);
			
			if(pm.isEnabled()) {
				paymentMethods.put(pm.getPaymentModuleName(), pm);
			}
		}
		
		return paymentMethods;

	}

}



```
