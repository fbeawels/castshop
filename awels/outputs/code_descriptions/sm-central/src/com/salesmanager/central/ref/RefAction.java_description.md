# RefAction.java

## Review

## 1. Summary

`RefAction` is a Struts 2 action that supplies reference data for the UI (product types, order status, credit‑card information, locales, etc.).  
The class loads a handful of static lookup tables on first load, uses a `ServiceFactory` to obtain a `CatalogService` for product option types, and relies on a `RefCache` singleton to pull most of the data out of the database or cache.

Key components  
| Component | Role |
|-----------|------|
| `static` maps (`creditActionsMap`, `creditCVVMap`, `typesMap`) | Pre‑loaded locale‑specific lookup tables for credit‑card actions, CVV usage flags, and product types. |
| `ServiceFactory` | Provides a `CatalogService` instance for retrieving product option types. |
| `RefCache` | Singleton cache that exposes product types, credit cards, order status, weight units, etc. |
| `LocaleUtil` | Utility that sets the locale on entity collections. |

The class follows a very classic Struts‑style “populate the form” approach – it does **not** perform any business logic itself, just data gathering and transformation.

---

## 2. Detailed Description

### Flow of Execution

1. **Class Loading** – The static block executes once. It:
   * Loads language‑specific maps from `LanguageLabels`.
   * Populates `creditCVVMap` and `typesMap` by calling the cache for product types.
2. **Request Handling** – When a request hits this action:
   * Struts calls the action’s public methods (e.g. `getProductOptionTypes`, `getCreditCards`) to populate UI components.
   * No `execute()` method is defined – the action relies on getters only (a common Struts 2 “plain” action pattern).
3. **Cleanup** – `prepare()` is a no‑op, reflecting that there is no resource to release.

### Assumptions / Constraints

* The action is stateless apart from its instance‑level `Calendar`.  
* Locale is obtained via `super.getLocale()` (presumably from `BaseAction`).  
* All cached data is assumed to be thread‑safe; `RefCache` must guarantee immutability or internal synchronization.  
* `LanguageLabels` and `RefCache` are available on the classpath and correctly configured.

### Architecture / Design Choices

* **Struts 2 Plain Action** – Uses only getters, no explicit `execute()`.  
* **ServiceFactory Pattern** – Loose coupling to `CatalogService`.  
* **Cache‑Driven Data** – Heavy reliance on `RefCache` avoids repeated DB hits.  
* **Locale‑Aware Maps** – Static maps are pre‑localized for English & French only; other languages will fall back to `null` and cause `NullPointerException`s if accessed.

---

## 3. Functions/Methods

| Method | Purpose | Input | Output | Side‑Effects |
|--------|---------|-------|--------|--------------|
| `getProductOptionTypes()` | Fetch product option types via `CatalogService`. | None | `Collection` of option types or empty list on error | Logs error |
| `getCreditpmactions()` | Builds a map of payment transaction type keys to localized labels. | None | `Map<String,String>` | None |
| `getProducttypesmap()` | Returns locale‑specific product type map. | None | `Map` | None |
| `getCvvmap()` | Returns locale‑specific CVV usage map. | None | `Map` | None |
| `getTransactionType()` | Looks up a request attribute `transactionType` and returns a localized label. | None | `String` | None |
| `getStatus()` | Retrieves order status list localized for current request. | None | `List<OrderStatus>` | None |
| `getProductTypes()` | Returns all product types from cache. | None | `Collection` | None |
| `getCreditCards()` | Returns supported credit cards from cache. | None | `Collection<CentralCreditCard>` | None |
| `getCreditCardYears()` | Builds a list of the next ten years starting from the current year. | None | `Collection<Integer>` | None |
| `getYesno()`, `getTruefalse()`, `getSuccessfail()` | Build maps for UI radio/button labels. | None | `Map` | None |
| `getEnvironments()` | Hard‑coded mapping of environment ids to names. | None | `Map<Integer,String>` | None |
| `getCreditCardMonths()` | Returns list of 01‑12 as strings. | None | `Collection<String>` | None |
| `getWeightUnits()`, `getSizeUnits()` | Return collections of units, setting locale on each entity. | None | `Collection` | Side‑effect: modifies entity locale |
| `getAllCountries()` | Returns all countries localized for the current language. | None | `Collection` | None |
| `getCurrencies()` | Returns currency list from cache. | None | `Collection<Currency>` | None |
| `getLanguages()` | Returns all languages with indices. | None | `Collection` | None |
| `prepare()` | No‑op placeholder required by Struts. | None | None | None |

All methods are public and use raw types; generics would improve type safety.

---

## 4. Dependencies

| Library / Package | Role | Standard / 3rd‑party |
|-------------------|------|-----------------------|
| `org.apache.log4j.Logger` | Logging | 3rd‑party |
| `javax.servlet.http.HttpServletRequest` | Servlet request handling | Java EE |
| `com.salesmanager.core.*` | Core domain entities, services, caches | 3rd‑party (SalesManager) |
| `com.salesmanager.central.*` | Base action, context, constants | 3rd‑party (SalesManager) |
| `java.util.*` | Collections, Calendar, Locale | Standard |

No external framework beyond Struts 2 and the SalesManager core library. No platform‑specific code beyond `HttpServletRequest`.

---

## 5. Additional Notes

### Strengths
* **Simple, readable** – The action is a thin façade over cache lookups.
* **Localization support** – Most lookups are locale‑aware.
* **Caching** – Reduces database load by using a dedicated `RefCache`.

### Weaknesses / Risks
1. **Raw Types** – The use of non‑generic `Map`, `Collection`, and `Iterator` leads to unchecked casts and compiler warnings.
2. **Hardcoded Locale Support** – The static block only supports English & French. Any other locale will result in `null` maps, potentially causing `NullPointerException`s in the UI.
3. **Thread Safety** – The static maps are populated only once, which is safe; however, the `Calendar` instance (`cal`) is a mutable object but is never shared across threads, so it is fine.
4. **No Validation / Error Handling** – Methods that interact with external services (`CatalogService`, `RefCache`) swallow exceptions and return empty collections, hiding the problem from the caller.
5. **Deprecated `prepare()`** – The method exists only to satisfy Struts; it could be removed if Struts no longer requires it.
6. **Hardcoded Environment Map** – `getEnvironments()` returns numeric keys wrapped in `Integer` objects unnecessarily. A simple `Map<Integer, String>` literal or an `enum` would be clearer.
7. **Locale Setting on Collections** – `LocaleUtil.setLocaleToEntityCollection(coll, super.getLocale());` mutates the entities; this side effect may be surprising if the same collection is cached and reused.

### Suggested Enhancements
* **Add Generics** – Update all collection and map usages to use generics (e.g. `Map<Integer, String>`).
* **Externalize Locale Support** – Move the language keys (`en`, `fr`) into a configuration file or use `Locale.getISOLanguage()` to look up maps dynamically.
* **Inject Services** – Replace `ServiceFactory.getService` with dependency injection (e.g. Spring) to improve testability.
* **Return Optional / Throw Exceptions** – Instead of swallowing exceptions, either propagate them or return `Optional` to signal failure clearly.
* **Unit Tests** – Create tests for each getter using a mock `RefCache` and locale context.
* **Refactor `getEnvironments()`** – Use an enum `Environment { PRODUCTION, TEST }` with `ordinal()` or a constant map.
* **Document Methods** – Add Javadoc comments to explain each getter’s intent and the data it returns.

Overall, `RefAction` is functional but would benefit from modernization (generics, dependency injection, clearer error handling) to improve robustness, maintainability, and type safety.

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
package com.salesmanager.central.ref;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.apache.log4j.Logger;

import com.salesmanager.central.BaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.PaymentConstants;
import com.salesmanager.core.entity.orders.OrderStatus;
import com.salesmanager.core.entity.reference.CentralCreditCard;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;

public class RefAction extends BaseAction {

	private Logger log = Logger.getLogger(RefAction.class);
	private java.util.Calendar cal = new java.util.GregorianCalendar();

	private static Map creditActionsMap = new HashMap();
	private static Map creditCVVMap = new HashMap();
	private static Map typesMap = new HashMap();

	static {
		//Map actionsen = LanguageLabels.getCreditCardActions("en");
		//creditActionsMap.put("en", actionsen);
		//Map actionsfr = LanguageLabels.getCreditCardActions("fr");
		//creditActionsMap.put("fr", actionsfr);
		
		
		Map cvven = LanguageLabels.useCVV("en");
		creditCVVMap.put("en", cvven);
		Map cvvfr = LanguageLabels.useCVV("fr");
		creditCVVMap.put("fr", cvvfr);
		Collection types = com.salesmanager.core.service.cache.RefCache
				.getProductTypes();
		Map typen = LanguageLabels.getProductTypes("en", types);
		typesMap.put("en", typen);
		Map typfr = LanguageLabels.getProductTypes("fr", types);
		typesMap.put("fr", typfr);
	}

	public Collection getProductOptionTypes() {

		try {
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			return cservice.getProductOptionTypes();
		} catch (Exception e) {
			log.error(e);
		}
		return new ArrayList();

	}

	public Map getCreditpmactions() {
		
		Map cardactions = new HashMap();
		
		LabelUtil label = LabelUtil.getInstance();
		label.setLocale(super.getLocale());


		cardactions.put(PaymentConstants.PREAUTH, label.getText("label.payment.gateway.transactiontype.1"));
		cardactions.put(PaymentConstants.SALE, label.getText("label.payment.gateway.transactiontype.0"));
		
		return cardactions;
	}

	public Map getProducttypesmap() {
		String lang = super.getLocale().getLanguage();
		return (Map) typesMap.get(lang);
	}

	public Map getCvvmap() {
		String lang = super.getLocale().getLanguage();
		return (Map) creditCVVMap.get(lang);
	}

	public String getTransactionType() {
		String trtype = (String) this.getServletRequest().getAttribute(
				"transactionType");
		if (trtype == null) {
			trtype = "";
		}
		return LabelUtil.getInstance().getText(super.getServletRequest(),
				"label.payment.gateway.transactiontype." + trtype);
	}

	public List getStatus() {
		
		Locale locale = LocaleUtil.getLocale(super.getServletRequest());
		String lang = locale.getLanguage();
		Map smap = com.salesmanager.core.service.cache.RefCache
				.getOrderstatuswithlang(LanguageUtil
						.getLanguageNumberCode(lang));
		Iterator i = smap.keySet().iterator();
		List l = new ArrayList();
		while (i.hasNext()) {
			int keyid = (Integer) i.next();
			l.add((OrderStatus) smap.get(keyid));
		}
		return l;
	}

	public Collection getProductTypes() {
		return com.salesmanager.core.service.cache.RefCache.getProductTypes();
	}

	public Collection getCreditCards() {

		List l = new ArrayList();
		Map ccmap = com.salesmanager.core.service.cache.RefCache
				.getSupportedCreditCards();

		if (ccmap != null) {
			Iterator i = ccmap.keySet().iterator();

			while (i.hasNext()) {
				int key = (Integer) i.next();
				l.add((CentralCreditCard) ccmap.get(key));
			}
		}
		return l;

	}

	public Collection getCreditCardYears() {
		List l = new ArrayList();
		int yearNow = cal.get(java.util.Calendar.YEAR);
		for (int i = 0; i < 10; i++) {
			int y = yearNow + i;
			l.add(y);
		}
		return l;
	}

	public Map getYesno() {
		HttpServletRequest req = this.getServletRequest();
		Locale loc = super.getLocale();
		return LanguageLabels.buildYesNo(loc);
	}

	public Map getTruefalse() {
		HttpServletRequest req = this.getServletRequest();
		Locale loc = super.getLocale();
		return LanguageLabels.buildTrueFalse(loc);
	}

	public Map getSuccessfail() {
		HttpServletRequest req = this.getServletRequest();
		Locale loc = super.getLocale();
		return LanguageLabels.buildSuccessFail(loc.getLanguage());
	}

	public Map getEnvironments() {
		HttpServletRequest req = this.getServletRequest();
		Locale loc = req.getLocale();
		Map env = new HashMap();
		env.put(new Integer(1).intValue(), "Production");
		env.put(new Integer(2).intValue(), "Test");

		return env;
	}

	public Collection getCreditCardMonths() {
		List l = new ArrayList();
		l.add("01");
		l.add("02");
		l.add("03");
		l.add("04");
		l.add("05");
		l.add("06");
		l.add("07");
		l.add("08");
		l.add("09");
		l.add("10");
		l.add("11");
		l.add("12");
		return l;
	}

	public Collection getWeightUnits() {
		// set lang to all objects
		Collection coll = com.salesmanager.core.service.cache.RefCache
				.getWeightunits().values();
		LocaleUtil.setLocaleToEntityCollection(coll, super.getLocale());
		return coll;
	}

	public Collection getSizeUnits() {
		Collection coll = com.salesmanager.core.service.cache.RefCache
				.getSizeunits().values();
		LocaleUtil.setLocaleToEntityCollection(coll, super.getLocale());
		return coll;
	}

	public Collection getAllCountries() {
		Context ctx = (Context) this.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Map countries = com.salesmanager.core.service.cache.RefCache
				.getAllcountriesmap(LanguageUtil.getLanguageNumberCode(ctx
						.getLang()));
		return countries.values();
	}

	public Collection getCurrencies() {

		Map currenciesMap = com.salesmanager.core.service.cache.RefCache
				.getCurrenciesListWithCodes();
		if (currenciesMap != null) {
			List returnlist = new ArrayList();
			Iterator i = currenciesMap.keySet().iterator();
			while (i.hasNext()) {
				String key = (String) i.next();
				com.salesmanager.core.entity.reference.Currency c = (com.salesmanager.core.entity.reference.Currency) currenciesMap
						.get(key);
				returnlist.add(c);
			}

			return returnlist;
		} else {
			return new ArrayList();
		}
	}

	public Collection getLanguages() {

		return com.salesmanager.core.service.cache.RefCache
				.getLanguageswithindex().values();

	}

	/** required by struts **/
	public void prepare() {

	}

}



```
