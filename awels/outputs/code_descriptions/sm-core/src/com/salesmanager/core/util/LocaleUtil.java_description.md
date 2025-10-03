# LocaleUtil.java

## Review

## 1. Summary  

`LocaleUtil` is a static helper class that centralises all locale‑related logic for the *Sales Manager* web application.  
It:

| Purpose | How it is achieved | Key components |
|---------|--------------------|----------------|
| **Translate a language code → `Locale`** | `getLocale(String)` | Constants (`ENGLISH_CODE`, `FRENCH_CODE`) |
| **Retrieve the system default locale** | `getDefaultLocale()` | `PropertiesUtil`, `RefCache`, `Country` |
| **Derive a locale from a merchant store entity** | `getLocaleFromStoreEntity(MerchantStore, String)` | `RefCache`, `Country` |
| **Persist/restore locale in the HTTP session** | `setLocale(...)`, `getLocale(HttpServletRequest)` | Servlet API |
| **Propagate a locale to collections of `I18NEntity` objects** | `setLocaleToEntityCollection(...)` | `I18NEntity` |
| **Perform all locale wiring for an HTTP request** | `setLocaleForRequest(...)` | `ActionContext`, `MerchantStore`, `CountryUtil`, `LabelUtil`, `Cookie` |

The class relies on a mix of **Apache Commons** (`StringUtils`, `Configuration`), **Log4j**, **Struts/XWork2**, and the application’s own domain model (`MerchantStore`, `Country`, `Language`, `I18NEntity`, etc.).  It adopts a **utility‑class** pattern: all methods are `static`, the constructor is private, and the class is stateless.  

## 2. Detailed Description  

### Execution Flow  

1. **Initialisation** – nothing special; the class is loaded once and cached by the JVM.  
2. **Request Handling** – `setLocaleForRequest` is called by a Struts action or a servlet filter.  
   * It examines the request parameter `request_locale`.  
   * If present, it parses the string into language and country components.  
   * It validates the language against the store’s supported languages.  
   * If the locale is valid, it stores it in the Struts session and in the HTTP session.  
   * If not present, it falls back to the store’s default language/country, then to the system default.  
   * The resolved locale is then applied to:
     * the `LabelUtil` instance (for UI translations),
     * the request attribute `LANGUAGE` (human‑readable name),
     * the request attribute `LOCALE`,
     * a cookie named `LOCALE` for client‑side persistence.  
3. **Entity Locale Propagation** – When entities that implement `I18NEntity` are returned to the view, the developer can call `setLocaleToEntityCollection` to stamp each entity with the current locale (and optionally currency).  
4. **Locale Retrieval** – Other parts of the application can call `getLocale(HttpServletRequest)` or `getLocale(String)` to obtain the locale for the current user or a particular language code.  

### Dependencies & Assumptions  

* The **default country** and **default language** are stored in the application configuration (via `PropertiesUtil`).  
* The list of all countries (`RefCache.getAllcountriesmap`) is cached in a `Map<Integer,Country>`.  
* `CountryUtil.getCountryIsoCodeById` and `LanguageUtil` provide ISO codes and language look‑ups.  
* The `MerchantStore` entity exposes its supported languages and default language via maps/strings.  
* The code assumes that the HTTP session and Struts `ActionContext` are available for every request that invokes `setLocaleForRequest`.  
* No thread‑local or request‑scoped caching is used; all data is either stateless or stored in the session.  

### Architectural Choices  

* **Utility‑class pattern** – static methods, no state, no dependency injection.  
* **Lazy localisation** – locales are resolved only when needed, mostly from session data.  
* **Session‑backed locale** – the Struts/HTTP session holds the locale, so all subsequent requests can reuse it without re‑parsing.  
* **Cookie fallback** – the `LOCALE` cookie is written to the client; however, the code never reads it back, so its usefulness is limited.  

## 3. Functions / Methods  

| Method | Signature | Purpose | Inputs | Outputs | Side‑effects |
|--------|-----------|---------|--------|---------|--------------|
| `getLocale(String lang)` | `public static Locale getLocale(String)` | Convert a 2‑letter language code to a `Locale`. | `lang` – ISO language code (`"en"`, `"fr"`, etc.) | `Locale` | Logs warning if unknown code. |
| `getDefaultLocale()` | `public static Locale getDefaultLocale()` | Return system default locale based on config. | None | `Locale` | Reads configuration; logs error if country map missing. |
| `getLocaleFromStoreEntity(MerchantStore store, String defaultLanguage)` | `public static Locale getLocaleFromStoreEntity(MerchantStore, String)` | Derive a locale from a merchant store entity. | `store`, `defaultLanguage` | `Locale` | Uses country cache; logs error if map missing. |
| `getLocale(HttpServletRequest req)` | `public static Locale getLocale(HttpServletRequest)` | Retrieve locale from HTTP session, fallback to default. | `req` | `Locale` | None. |
| `setLocale(HttpServletRequest req, Locale locale)` | `public static void setLocale(HttpServletRequest, Locale)` | Store locale in HTTP session. | `req`, `locale` | None | Sets session attribute `WW_TRANS_I18N_LOCALE`. |
| `setLocaleToEntityCollection(Collection<I18NEntity> coll, Locale locale)` | `public static void setLocaleToEntityCollection(Collection<I18NEntity>, Locale)` | Set locale for each entity in a collection. | `coll`, `locale` | None | Calls `entity.setLocale(locale)` on each element. |
| `setLocaleToEntityCollection(Collection<I18NEntity> coll, Locale locale, String currency)` | Overload with currency | Same as above but also sets currency. | `coll`, `locale`, `currency` | None | Calls `entity.setLocale(locale, currency)` on each element. |
| `setLocaleForRequest(HttpServletRequest request, HttpServletResponse response, ActionContext ctx, MerchantStore store)` | `public static void setLocaleForRequest(HttpServletRequest, HttpServletResponse, ActionContext, MerchantStore)` | Main entry point for locale handling on every request. | `request`, `response`, `ctx`, `store` | None | • Parses `request_locale`. <br>• Validates language against store.<br>• Stores locale in session, request, and cookie.<br>• Populates `LabelUtil` and request attributes. |
| `LocaleUtil()` | Private constructor | Prevent instantiation. | None | None | – |

### Reusable / Utility Methods  

* `getLocale(String)` – used throughout the application to convert a language code into a `Locale`.  
* `setLocaleToEntityCollection` – convenience for propagating locale to domain objects.  
* `setLocale(HttpServletRequest, Locale)` – stores the locale in the session; can be used by login/logout flows.  

## 4. Dependencies  

| Library / Class | Purpose | Standard / Third‑Party |
|-----------------|---------|------------------------|
| `org.apache.commons.configuration.Configuration` | Access to application properties | Third‑Party |
| `org.apache.commons.lang.StringUtils` | String helper methods | Third‑Party |
| `org.apache.log4j.Logger` | Logging | Third‑Party |
| `com.opensymphony.xwork2.ActionContext` | Struts/XWork session map | Third‑Party |
| `javax.servlet.http.*` | HTTP request/response handling | Java EE |
| Domain classes (`MerchantStore`, `Country`, `Language`, `I18NEntity`) | Business data | Application |
| `RefCache` | In‑memory cache for reference data | Application |
| `CountryUtil`, `LanguageUtil`, `LabelUtil` | Helper utilities for country/language labels | Application |
| `com.salesmanager.core.constants.Constants` | Static constant values | Application |

The code is **platform‑agnostic** in that it relies only on standard servlet APIs and common third‑party libraries. No native or OS‑specific features are used.

## 5. Additional Notes & Recommendations  

### Edge Cases & Potential Bugs  

1. **Locale construction in `getLocaleFromStoreEntity`** – `new Locale(country.getCountryIsoCode2())` creates a *language‑only* locale (ISO 639). The intended behaviour is probably `new Locale(language, country)`.  
2. **Cookie max‑age calculation** – `c.setMaxAge(2 * 24 * 24)` results in **1,152 seconds (~19 minutes)**, not two days. The correct value should be `2 * 24 * 60 * 60`.  
3. **Missing null‑checks** – `store.getCountry()` could return `null`, leading to a `NullPointerException` when used as a key.  
4. **Unnecessary raw types** – `Map`, `List`, `Iterator` are used without generics, generating unchecked warnings and making the code harder to read.  
5. **Hard‑coded locale attribute name** – `"WW_TRANS_I18N_LOCALE"` is a Struts convention; if the app moves away from Struts, this may break.  
6. **No fallback for invalid `request_locale`** – The code logs a warning but then continues with `locale = null`, which may cause a `NullPointerException` later when it attempts to call `locale.getLanguage()` on a null object.  
7. **`setLocaleForRequest` does not read the `LOCALE` cookie** – The cookie is only written, never consumed, so it provides no benefit.  

### Suggested Enhancements  

| Area | Suggested Change | Benefit |
|------|------------------|---------|
| **Generics** | Replace raw `Map`, `List`, `Iterator` with typed variants. | Eliminates warnings, safer casts. |
| **Error handling** | Return a default locale or throw a custom exception when `locale` ends up `null` after parsing. | Avoids NPEs downstream. |
| **Configuration** | Externalise the cookie max‑age and session attribute key into constants. | Centralised configuration, easier refactor. |
| **Locale building** | Use `Locale.forLanguageTag()` or `new Locale(language, country)` consistently. | Correct locale representation. |
| **Cookie handling** | Read the `LOCALE` cookie on incoming requests to bootstrap the locale if session data is missing. | Improves user experience when sessions expire. |
| **Testing** | Unit tests covering all parsing scenarios, including malformed `request_locale` strings. | Ensure robustness against malformed input. |
| **Logging** | Use structured logs (e.g., log4j2) and add context identifiers (e.g., store ID, user ID). | Easier debugging in production. |

### Design Observations  

* The class embodies a **stateless utility** pattern, which is simple but limits testability. Consider injecting dependencies (`RefCache`, `PropertiesUtil`) to facilitate mocking.  
* The locale resolution logic is heavily coupled to the store’s supported languages. A more modular approach could delegate language‑country validation to a dedicated service (`LocaleValidator`).  
* The use of `ActionContext` ties the method to Struts; if the application evolves to a different MVC framework, this coupling will force a rewrite.  

Overall, `LocaleUtil` fulfills its role in centralising locale management, but it would benefit from modern Java best‑practices, stronger type safety, and more robust error handling. Addressing the highlighted issues will improve maintainability, testability, and user experience.

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
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.ActionContext;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.common.I18NEntity;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.service.cache.RefCache;

public class LocaleUtil {

	private static Logger log = Logger.getLogger(LocaleUtil.class);

	private LocaleUtil() {
	}

	public static Locale getLocale(String lang) {

		if (StringUtils.isBlank(lang)) {
			return getDefaultLocale();
		}

		if (lang.equals(Constants.ENGLISH_CODE)) {
			return Locale.ENGLISH;
		} else if (lang.equals(Constants.FRENCH_CODE)) {
			return Locale.FRENCH;
		} else {
			log.warn("Resources for this language " + lang
					+ " may not be handled by this system");
			return new Locale(lang);
		}

	}

	public static Locale getDefaultLocale() {
		Configuration conf = PropertiesUtil.getConfiguration();
		int defaultCountryId = conf.getInt("core.system.defaultcountryid", 38);
		
		Map countriesMap = RefCache.getAllcountriesmap(Constants.ENGLISH);
		
		if(countriesMap==null) {
			log.error("Cannot get object from database, check your database configuration");
		}
		
		Country country = (Country) countriesMap.get(defaultCountryId);
		Locale locale = new Locale(conf.getString(
				"core.system.defaultlanguage", Constants.ENGLISH_CODE), country
				.getCountryIsoCode2());
		return locale;
	}

	public static Locale getLocaleFromStoreEntity(MerchantStore store,
			String defaultLanguage) {
		Map countriesMap = RefCache.getAllcountriesmap(LanguageUtil
				.getLanguageNumberCode(defaultLanguage));
		if (countriesMap == null) {
			log.error("Cannnot get a Map for language code "
					+ defaultLanguage);
			return getDefaultLocale();
		}
		Country country = (Country) countriesMap.get(store.getCountry());
		Locale locale = new Locale(country.getCountryIsoCode2());
		return locale;
	}

	public static Locale getLocale(HttpServletRequest req) {
		Locale locale = (Locale) req.getSession().getAttribute(
				"WW_TRANS_I18N_LOCALE");

		if (locale == null) {
			locale = LocaleUtil.getDefaultLocale();
		}
		return locale;
	}

	public static void setLocale(HttpServletRequest req, Locale locale) {
		req.getSession().setAttribute("WW_TRANS_I18N_LOCALE", locale);
	}

	public static void setLocaleToEntityCollection(Collection<I18NEntity> coll,
			Locale locale) {
		if (coll != null) {
			Iterator i = coll.iterator();
			while (i.hasNext()) {
				I18NEntity entity = (I18NEntity) i.next();
				entity.setLocale(locale);
			}
		}
	}

	public static void setLocaleToEntityCollection(Collection<I18NEntity> coll,
			Locale locale, String currency) {
		if (coll != null) {
			Iterator i = coll.iterator();
			while (i.hasNext()) {
				I18NEntity entity = (I18NEntity) i.next();
				entity.setLocale(locale, currency);
			}
		}
	}

	public static void setLocaleForRequest(HttpServletRequest request,
			HttpServletResponse response, ActionContext ctx, MerchantStore store)
			throws Exception {

		/**
		 * LOCALE
		 */

		Map sessions = ctx.getSession();

		if (ctx == null) {
			throw new Exception(
					"This request was not made inside Struts request, ActionContext is null");
		}

		Locale locale = null;

		// check in http request
		String req_locale = (String) request.getParameter("request_locale");
		if (!StringUtils.isBlank(req_locale)) {
			
			String l = null;
			String c = null;
			
			if(req_locale.length()==2) {//assume it is the language
				l = req_locale;
				c = CountryUtil.getCountryIsoCodeById(store.getCountry());
			}
			
			if(req_locale.length()==5) {
				
				try {
					l = req_locale.substring(0, 2);
					c = req_locale.substring(3);
				} catch (Exception e) {
					log.warn("Invalid locale format " + req_locale);
					l = null;
					c = null;
				}
				
			}


			if(l!=null && c != null) {
			
				String storeLang = null;
				Map languages = store.getGetSupportedLanguages();
				if (languages != null && languages.size() > 0) {
					Iterator i = languages.keySet().iterator();
					while (i.hasNext()) {
						Integer langKey = (Integer) i.next();
						Language lang = (Language) languages.get(langKey);
						if (lang.getCode().equals(l)) {
							storeLang = l;
							break;
						}
					}
				}
	
				if (storeLang == null) {
					l = store.getDefaultLang();
					if (StringUtils.isBlank(l)) {
						l = LanguageUtil.getDefaultLanguage();
					}
				}
	
				locale = new Locale(l, c);
				if (StringUtils.isBlank(locale.getLanguage())
						|| StringUtils.isBlank(locale.getCountry())) {
					log.error("Language or Country is not set in the new locale "
							+ req_locale);
					return;
				}
				sessions.put("WW_TRANS_I18N_LOCALE", locale);
			
			}
		}

		locale = (Locale) sessions.get("WW_TRANS_I18N_LOCALE");
		request.getSession().setAttribute("WW_TRANS_I18N_LOCALE", locale);

		if (locale == null) {

			String c = CountryUtil.getCountryIsoCodeById(store.getCountry());
			String lang = store.getDefaultLang();
			if (!StringUtils.isBlank(c) && !StringUtils.isBlank(lang)) {
				locale = new Locale(lang, c);
			} else {
				locale = LocaleUtil.getDefaultLocale();
				String langs = store.getSupportedlanguages();
				if (!StringUtils.isBlank(langs)) {
					Map languages = store.getGetSupportedLanguages();
					String defaultLang = locale.getLanguage();
					if (languages != null && languages.size() > 0) {
						Iterator i = languages.keySet().iterator();
						String storeLang = "";
						while (i.hasNext()) {
							Integer langKey = (Integer) i.next();
							Language l = (Language) languages.get(langKey);
							if (l.getCode().equals(defaultLang)) {
								storeLang = defaultLang;
								break;
							}
						}
						if (!storeLang.equals(defaultLang)) {
							defaultLang = storeLang;
						}
					}

					if (!StringUtils.isBlank(defaultLang)
							&& !StringUtils.isBlank(c)) {
						locale = new Locale(defaultLang, c);
					}

				}
			}

			sessions.put("WW_TRANS_I18N_LOCALE", locale);
		}

		if (locale != null) {
			LabelUtil label = LabelUtil.getInstance();
			label.setLocale(locale);
			String lang = label.getText("label.language."
					+ locale.getLanguage());
			request.setAttribute("LANGUAGE", lang);
		}

		if (store.getLanguages() == null || store.getLanguages().size() == 0) {

			// languages
			if (!StringUtils.isBlank(store.getSupportedlanguages())) {
				List languages = new ArrayList();
				List langs = LanguageUtil.parseLanguages(store
						.getSupportedlanguages());
				for (Object o : langs) {
					String lang = (String) o;
					Language l = LanguageUtil.getLanguageByCode(lang);
					if (l != null) {
						l.setLocale(locale, store.getCurrency());
						languages.add(l);
					}
				}
				store.setLanguages(languages);
			}
		}

		request.setAttribute("LOCALE", locale);
		Cookie c = new Cookie("LOCALE", locale.getLanguage() + "_"
				+ locale.getCountry());
		c.setPath("/");
		c.setMaxAge(2 * 24 * 24);
		response.addCookie(c);

	}

}



```
