# CustomerUtil.java

## Review

## 1. Summary

`CustomerUtil` is a utility helper that lives in the *salesmanager* web‑application.  
Its primary responsibilities are:

| Method | Purpose |
|--------|---------|
| `logout()` | Cleans up a logged‑in customer’s session, logs the logout event, and records the last log‑on time in the database. |
| `setGeoLocationCustomerInformation(...)` | Updates a customer’s geographical profile (country, state/zone, city) and locale based on the values supplied by an AJAX request. |

The class pulls in several third‑party libraries:

* **Apache Commons Lang** – `StringUtils`
* **log4j** – `Logger`
* **Direct Web Remoting (DWR)** – `WebContextFactory`
* **Spring** – bean lookup via a custom `SpringUtil`
* **SalesManager core** – various domain entities (`Customer`, `MerchantStore`, …) and services (`CustomerService`, `CountryUtil`, `LocaleUtil`, …)

No design patterns are explicitly used (the class is a plain static‑method holder with one instance method that still relies on static context).

---

## 2. Detailed Description

### Execution Flow

#### `logout()`

1. Acquire the current `HttpServletRequest` and session through `WebContextFactory`.
2. Retrieve the Spring bean `customerLogon` (type `CustomerLogonModule`) and invoke its `logout(HttpServletRequest)` method – this presumably invalidates the HTTP session or removes authentication tokens.
3. Fetch the currently logged‑in `Customer` from the session via `SessionUtil`.
4. If a customer exists:
   * Load the `CustomerInfo` entity via `CustomerService`.
   * Update its `customerInfoDateOfLastLogon` to the current timestamp and persist it.
5. Any exception is caught and logged; it is *not* re‑thrown (despite the method signature declaring `throws Exception`).

#### `setGeoLocationCustomerInformation(...)`

1. Grab the current `HttpServletRequest` and session.
2. If `country` is non‑blank:
   * Resolve a `CountryDescription` by ISO code.
   * If found, populate a `Customer` object (create a new one if none exists in the session):
     - Set country name, ID, billing country info.
     - Resolve a `Zone` by region code and populate zone‑related fields.
     - Set the merchant ID from the current `MerchantStore` (if present).
     - Compute a new `Locale` from the language code (`locale.getLanguage()`) and the country code, then store it back in the request and in the `Customer`.
   * If not found, fall back to the default locale.
3. Any exception is caught and logged; it is swallowed.

### Dependencies & Constraints

* The utility depends on the DWR `WebContextFactory` to obtain the request/response context, which ties it tightly to a web environment and makes unit testing difficult without a mock web context.
* It relies on Spring’s bean container for the `CustomerLogonModule`, again via a static lookup (`SpringUtil.getBean`).
* The code assumes that `CountryUtil.getCountryByIsoCode()` and `CountryUtil.getZoneCodeByCode()` return non‑null objects if the codes are valid, but otherwise silently returns `null` – the caller must handle that.
* The `city` and `language` parameters are unused – either a bug or leftover code.

### Architectural Observations

* The class is a *utility façade* that aggregates several domain services; it would be cleaner to inject the required services (e.g., `CustomerService`, `CountryUtil`) rather than reach into static factories.
* All public methods are *non‑stateless* – `logout()` is an instance method but still uses static context internally; `setGeoLocationCustomerInformation` is static. This mixture can be confusing.
* The error handling strategy is inconsistent: the methods log and swallow exceptions but the `logout()` signature claims to throw `Exception`. This can hide runtime problems from callers.

---

## 3. Functions/Methods

| Method | Parameters | Return | Side‑Effects |
|--------|------------|--------|--------------|
| `public void logout() throws Exception` | None | void | • Calls `customerLogon.logout(req)` which likely invalidates session. <br>• Persists `CustomerInfo.lastLogon` timestamp. <br>• Logs errors. |
| `public static void setGeoLocationCustomerInformation(String country, String region, String city, String language)` | *country*: ISO 3166 alpha‑2 <br>*region*: zone/state code <br>*city*: (unused) <br>*language*: (unused) | void | • Updates `Customer` object in session with country/zone/locale.<br>• Persists locale in `HttpServletRequest` and `Customer`.<br>• Logs progress and errors. |

Both methods are *utility*; they do not return values. Their primary effect is to modify the session state and, in the case of `logout()`, persist a database record.

---

## 4. Dependencies

| Library / Module | Purpose | Standard / Third‑Party |
|------------------|---------|------------------------|
| `org.apache.commons.lang.StringUtils` | Blank string checks | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party (log4j 1.x) |
| `uk.ltd.getahead.dwr.WebContextFactory` | Access to `HttpServletRequest` & `HttpSession` | Third‑party (DWR) |
| `com.salesmanager.core.util.SpringUtil` | Spring bean lookup | Third‑party (custom) |
| `com.salesmanager.core.util.SessionUtil` | Get/set `Customer` and `MerchantStore` in session | Custom |
| `com.salesmanager.core.service.ServiceFactory` | Retrieve `CustomerService` | Custom |
| `com.salesmanager.core.service.customer.CustomerService` | CRUD for `CustomerInfo` | Custom |
| `com.salesmanager.core.util.CountryUtil` | Resolve `CountryDescription` & `Zone` | Custom |
| `com.salesmanager.core.util.LocaleUtil` | Manage `Locale` in request | Custom |
| Domain entities (`Customer`, `CustomerInfo`, `MerchantStore`, `CountryDescription`, `Zone`) | Business data | Custom |

No platform‑specific constraints beyond the requirement of a servlet container (JSP/Servlet API) and the DWR infrastructure.

---

## 5. Additional Notes & Recommendations

### Edge Cases & Potential Issues

1. **Unused Parameters** – `city` and `language` are never referenced; this could indicate incomplete logic or a bug.
2. **Null Handling** – The method assumes that `desc` and `zone` are non‑null after the lookup. If either lookup fails, the code simply skips setting those fields; this may leave a `Customer` in an inconsistent state.
3. **Locale Creation** – The line `locale = new Locale(l, country);` does not validate that `l` or `country` are legal ISO codes. An invalid code will still be accepted, potentially causing unexpected UI behavior.
4. **Exception Swallowing** – Both methods catch generic `Exception` and only log it. Callers have no indication that an operation failed, which can lead to silent bugs.
5. **Thread Safety** – The static method mutates the session and request, which is fine in a per‑request context, but the class as a whole is not thread‑safe if used in a different scope.

### Design / Architecture Enhancements

| Issue | Suggested Fix |
|-------|---------------|
| Tight coupling to DWR’s `WebContextFactory` | Inject `HttpServletRequest` (or a `RequestContext`) via constructor or method parameters, or use Spring’s `RequestContextHolder`. |
| Static bean lookup (`SpringUtil.getBean`) | Use constructor injection or Spring’s `@Autowired` to obtain `CustomerLogonModule` and other services. |
| Method signatures that claim `throws Exception` but swallow it | Either remove the `throws` clause or re‑throw a custom unchecked exception to surface failures. |
| Mixed static/instance methods | Decide on a consistent style: make all methods static *or* create a service bean with instance methods. |
| Logging with log4j 1.x | Migrate to log4j 2.x or SLF4J for better performance and API. |
| Unused parameters | Remove or implement usage of `city` and `language` (e.g., set city in `Customer` or adjust locale). |
| Error handling | Return a boolean or throw a checked exception indicating success/failure; unit tests can verify proper rollback. |

### Future Extensions

1. **Internationalization** – Allow a dynamic mapping from language code to `Locale` and store a full locale in `Customer`.
2. **Region / City Persistence** – Persist `region` and `city` in `Customer` so that address forms can be pre‑filled.
3. **Unit Testing** – Refactor to allow injection of mock `HttpServletRequest`, `Session`, and service objects; this will enable pure unit tests without a servlet container.
4. **Logging Enhancements** – Use structured logging (e.g., MDC) to attach request IDs or customer IDs for easier traceability.

---

**Verdict** – The class fulfills its intended role but is tightly coupled to the servlet/DWR environment and Spring static lookup, which makes it hard to test and maintain. Refactoring towards dependency injection, better error handling, and removing unused parameters would significantly improve the code quality and reliability.

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

import java.util.Date;
import java.util.Locale;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import uk.ltd.getahead.dwr.WebContextFactory;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.customer.CustomerInfo;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.module.model.application.CustomerLogonModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class CustomerUtil {

	private static Logger log = Logger.getLogger(CustomerUtil.class);

	public void logout() throws Exception {

		try {

			HttpServletRequest req = WebContextFactory.get()
					.getHttpServletRequest();
			HttpSession session = WebContextFactory.get().getSession();

			CustomerLogonModule logon = (CustomerLogonModule) com.salesmanager.core.util.SpringUtil
					.getBean("customerLogon");
			logon.logout(req);

			Customer customer = SessionUtil.getCustomer(req);

			if (customer != null) {

				// get CustomerInfo
				CustomerService cservice = (CustomerService) ServiceFactory
						.getService(ServiceFactory.CustomerService);
				CustomerInfo customerInfo = cservice
						.findCustomerInfoById(customer.getCustomerId());

				if (customerInfo == null) {
					customerInfo = new CustomerInfo();
					customerInfo.setCustomerInfoId(customer.getCustomerId());
				}

				customerInfo.setCustomerInfoDateOfLastLogon(new Date());
				cservice.saveOrUpdateCustomerInfo(customerInfo);

			}

		} catch (Exception e) {
			log.error(e);
		}

	}

	public static void setGeoLocationCustomerInformation(String country,
			String region, String city, String language) {

		HttpServletRequest req = WebContextFactory.get()
				.getHttpServletRequest();
		HttpSession session = WebContextFactory.get().getSession();

		try {

			log.info("Setting LOCALE Country -> " + country + " region -> "
					+ region + " city -> " + city);

			if (!StringUtils.isBlank(country)) {

				CountryDescription desc = CountryUtil.getCountryByIsoCode(
						country, req.getLocale());

				if (desc != null) {

					log
							.info(" Country Description -> "
									+ desc.getCountryName());

					Customer customer = SessionUtil.getCustomer(req);
					if (customer == null) {
						customer = new Customer();
					}
					customer.setCountryName(desc.getCountryName());
					customer.setCustomerBillingCountryName(desc
							.getCountryName());
					customer.setCustomerBillingCountryId(desc.getId()
							.getCountryId());
					customer.setCustomerCountryId(desc.getId().getCountryId());

					// get the zone
					Zone zone = CountryUtil.getZoneCodeByCode(region, req
							.getLocale());
					if (zone != null) {
						customer.setCustomerBillingZoneId(zone.getZoneId());
						customer.setStateProvinceName(zone.getZoneName());
						customer.setCustomerZoneId(zone.getZoneId());
						customer.setCustomerState(zone.getZoneName());
					}

					MerchantStore store = SessionUtil.getMerchantStore(req);
					if (store != null) {
						customer.setMerchantId(store.getMerchantId());
					}

					// set Locale
					Locale locale = LocaleUtil.getLocale(req);
					log.info("Actual locale (" + locale.toString());
					String l = locale.getLanguage();

					locale = new Locale(l, country);
					log.info("Setting locale (" + l + "_" + country + ")");
					log.info("New locale (" + locale.toString() + ")");

					LocaleUtil.setLocale(req, locale);

					customer.setLocale(locale);
					customer.setCustomerLang(locale.getLanguage());

					SessionUtil.setCustomer(customer, req);

				} else {
					log.info("Setting default locale (1)");
					Locale locale = LocaleUtil.getDefaultLocale();
					LocaleUtil.setLocale(req, locale);
				}

			}

		} catch (Exception e) {
			log.error(e);
		}
	}

}



```
