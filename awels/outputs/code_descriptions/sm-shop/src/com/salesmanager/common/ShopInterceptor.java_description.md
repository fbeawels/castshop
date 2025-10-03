# ShopInterceptor.java

## Review

## 1. Summary

`ShopInterceptor` is an abstract Struts‑2 interceptor that centralises a wide range of “store‑front” concerns for every HTTP request that reaches the e‑commerce application.  
Its responsibilities include:

| Responsibility | What it does |
|----------------|--------------|
| **Configuration loading** | Retrieves merchant‑specific configuration from a cache or, if missing, falls back to the database via `MerchantService`. |
| **Shipping calculation** | Builds a `ShippingEstimate` based on country, shipping module, and various configuration flags. |
| **Portlet and label handling** | Loads portlet layout and dynamic labels (top‑navigation, custom pages, custom portlets) from cache or DB, and places them in request attributes. |
| **Request enrichment** | Adds customer, merchant, and messages to the request, and sets the page title for actions that extend `SalesManagerBaseAction`. |
| **Delegation** | Delegates to the concrete implementation via the abstract `doIntercept` method. |

The class relies heavily on a custom caching layer, a service factory pattern, and Struts‑2 `ActionInvocation`. It demonstrates a classic “pre‑process request” interceptor pattern, but mixes a lot of logic in a single method, making it hard to test and maintain.

## 2. Detailed Description

### 2.1 Flow of execution

1. **Locale & MerchantStore** – The interceptor obtains the request’s locale and the current `MerchantStore` from the session.  
2. **Configuration retrieval** – Attempts to pull a `ConfigurationResponse` from the cache (`CACHE_CONFIGURATION`). If not found, it queries the `MerchantService` and re‑caches the result.  
3. **Iterate over configurations** – For each `MerchantConfiguration` it:
   * Handles portlet configuration (`STORE_PORTLETS_`).
   * Exposes Google Analytics keys.
   * Loads shipping module names, country‑cost rules, estimate strings, and the shipping zone type.  
4. **Shipping estimate construction** – Builds a `ShippingEstimate` instance:
   * Sets locale, currency, type, and module.  
   * Parses region/cost strings into a map.  
   * Resolves country names via `ReferenceService`.  
   * Looks up shipping company logo.  
   * Calculates the zone index for the current country.  
5. **Dynamic labels** – Loads (or caches) dynamic labels needed on every page: shipping fee text, top‑navigation items, and custom portlet content.  
6. **Request attributes** – Puts the `ShippingEstimate`, customer, merchant id, page title, and any action messages into the request scope.  
7. **Delegation** – Calls the concrete interceptor implementation (`doIntercept`).  
8. **Return** – The resulting string is passed back to the framework.

### 2.2 Assumptions & constraints

| Assumption | Rationale |
|------------|-----------|
| `mStore` is always present in the session | The application guarantees a logged‑in merchant for every request. |
| Cached objects are serialisable and thread‑safe | Cache implementation must handle concurrent access. |
| String values in configuration are well‑formed | Parsing code does not guard against malformed tokens. |
| Action implements `SalesManagerBaseAction` when title needs to be set | Not all actions derive from this base class. |

### 2.3 Architecture & Design Choices

* **Interceptor‑based request pre‑processing** – A common Struts‑2 pattern.  
* **Service factory** – `ServiceFactory.getService()` is used instead of dependency injection, which reduces testability.  
* **Cache abstraction** – A custom `CacheModule` is used instead of a standard solution (EhCache, Guava).  
* **Legacy token parsing** – `StringTokenizer` and manual string manipulation are used for configuration parsing.  
* **Raw collections** – Most collections are raw types (`List`, `Map`, `Collection`) rather than generics.

## 3. Functions / Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `destroy()` | Lifecycle hook (no-op). | None | `void` | None |
| `init()` | Lifecycle hook (no‑op). | None | `void` | None |
| `baseIntercept(ActionInvocation, HttpServletRequest, HttpServletResponse)` | Core interceptor logic – loads configuration, builds shipping estimate, loads labels, enriches the request, delegates to `doIntercept`. | * `invoke` – Struts action invocation.<br>* `req` – current HTTP request.<br>* `resp` – current HTTP response. | `String` – result code from `doIntercept` | Sets numerous request attributes, logs errors, caches configuration/labels. |
| `doIntercept(ActionInvocation, HttpServletRequest, HttpServletResponse)` | Abstract extension point – concrete interceptors provide the final action result. | Same as `baseIntercept`. | `String` – action result | Up to concrete subclass. |

### Utility fragments inside `baseIntercept`

* **Configuration caching** – `cache.getFromCache` / `cache.putInCache`.
* **Shipping estimate construction** – Parsing region strings, setting country names.
* **Dynamic label handling** – Cache miss detection, fallback DB fetch.
* **Message handling** – `MessageUtil.getErrorMessage`, `getMessage`, `resetMessages`.

## 4. Dependencies

| Library | Type | Notes |
|---------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Provides safe string checks. |
| `org.apache.log4j.Logger` | Third‑party | Classic logging. |
| `javax.servlet.http` | Standard | Servlet API. |
| `com.opensymphony.xwork2` | Third‑party | Struts‑2 core. |
| `com.salesmanager.*` | Internal | Domain, service, util, model, and configuration classes. |
| `SpringUtil` | Internal wrapper | Provides Spring bean access (not full DI). |
| `CacheModule` | Internal | Custom cache abstraction. |
| `ReferenceService`, `MerchantService` | Internal | Business services. |
| `ServiceFactory` | Internal | Factory pattern for services. |

No external framework (e.g., Spring MVC) is used; the interceptor relies on a hand‑rolled service factory and a custom caching layer.

## 5. Additional Notes & Recommendations

### 5.1 Code quality & maintainability

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw collections & unchecked casts** | Compile‑time safety loss, potential `ClassCastException`. | Replace with generics (`List<MerchantConfiguration>`, `Map<String, PortletConfiguration>`, etc.). |
| **Use of `StringTokenizer`** | Legacy API; harder to read. | Use `String.split()` or `StringTokenizer` replacement (`StringTokenizer` is still fine but could be replaced for clarity). |
| **Swallowed exceptions** | Debugging is hard; silent failures may propagate. | Log exceptions at least (`log.warn(e)`), and consider re‑throwing or handling gracefully. |
| **Hard‑coded cache keys** | Magic strings risk typos. | Centralise keys in constants (`Constants.CACHE_LABELS + "_" + LabelConstants.STORE_FRONT_CUSTOM_PORTLETS + "_" + locale.getLanguage()`). |
| **Missing null checks** | Potential NPEs if session objects missing. | Validate `mStore`, `vo`, `request`, `rservice` before usage. |
| **Concurrency** | Cache writes are not synchronized. | Ensure `CacheModule` is thread‑safe or use a thread‑safe cache implementation. |
| **Business logic in interceptor** | Interceptor becomes large and difficult to test. | Extract configuration, shipping, and label logic into dedicated services/utility classes. |
| **Logging** | Only errors are logged; missing debug/info logs. | Add logging for major steps to aid troubleshooting. |
| **Hard‑coded `String` constants** | Risk of typos, not refactored. | Use enums or constant fields. |
| **No unit tests** | Hard to guarantee correctness. | Write unit tests for configuration parsing and shipping calculation. |
| **`doIntercept` signature** | Requires concrete class to repeat parameters. | Keep the method signature but document clearly. |

### 5.2 Performance

* The interceptor parses region strings on every request, which could be cached or pre‑parsed.  
* The entire dynamic label set is cached per locale; consider pre‑computing the `topNavList` and `customPortletsMap` as part of the cached object.

### 5.3 Security

* No direct user input is used for cache keys, so injection risk is low.  
* Ensure that `ConfigurationResponse` and other sensitive data are properly secured in the cache.

### 5.4 Future Enhancements

1. **Move to Spring DI** – Replace `ServiceFactory` and `SpringUtil` with proper `@Autowired` beans.  
2. **Use a well‑known caching framework** – EhCache, Caffeine, or Redis for better eviction policies.  
3. **Introduce a domain event** – When configuration changes, publish an event that clears relevant cache entries.  
4. **Refactor shipping logic** – Extract into a `ShippingEstimator` service that can be unit‑tested in isolation.  
5. **Implement a feature toggle** – To turn off heavy pre‑processing for certain paths or environments.  

Overall, `ShopInterceptor` is functional but overly monolithic. Refactoring the code into smaller, testable services will greatly improve maintainability, test coverage, and performance.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 25, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.common;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.StringTokenizer;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.ActionInvocation;
import com.salesmanager.catalog.common.PortletConfiguration;
import com.salesmanager.core.constants.ConfigurationConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.shipping.ShippingEstimate;
import com.salesmanager.core.entity.shipping.ShippingType;
import com.salesmanager.core.module.model.application.CacheModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.MerchantConfigurationUtil;
import com.salesmanager.core.util.MessageUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.ShippingUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.BaseActionAware;
import com.salesmanager.core.util.www.SalesManagerInterceptor;
import com.salesmanager.core.util.www.SessionUtil;

public abstract class ShopInterceptor extends SalesManagerInterceptor {

	private Logger log = Logger.getLogger(ShopInterceptor.class);

	public void destroy() {
		// TODO Auto-generated method stub

	}

	public void init() {
		// TODO Auto-generated method stub
	}

	public String baseIntercept(ActionInvocation invoke,
			HttpServletRequest req, HttpServletResponse resp) throws Exception {

		Locale locale = LocaleUtil.getLocale(req);

		MerchantStore mStore = SessionUtil.getMerchantStore(req);
		// languages

		Map<Integer, List> customPortletsMap = null;

		CacheModule cache = (CacheModule) SpringUtil.getBean("cache");
		ReferenceService rservice = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);

		ConfigurationResponse vo = null;

		try {
			vo = (ConfigurationResponse) cache.getFromCache(
					Constants.CACHE_CONFIGURATION, mStore);
		} catch (Exception ignore) {

		}

		if (vo == null) {

			/**
			 * MERCHANT CONFIGURATION
			 */
			ConfigurationRequest request = new ConfigurationRequest(mStore
					.getMerchantId());// get all configurations
			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			vo = mservice.getConfiguration(request);

			if (vo != null) {

				try {
					cache.putInCache(Constants.CACHE_CONFIGURATION, vo,
							Constants.CACHE_CONFIGURATION, mStore);
				} catch (Exception e) {
					log.error(e);
				}
			}
		}

		ShippingType shippingType = null;
		String shippingModule = null;// will determine shipping company name
		String shippingRegionLine = null;
		String shippingEstimateLine = null;

		if (vo != null && vo.getMerchantConfigurationList() != null
				&& vo.getMerchantConfigurationList().size() > 0) {
			List configurations = vo.getMerchantConfigurationList();

			Iterator i = configurations.iterator();
			Map confMap = new HashMap();
			Collection prtList = new ArrayList();
			while (i.hasNext()) {

				MerchantConfiguration conf = (MerchantConfiguration) i.next();

				if (conf.getConfigurationKey().equals(
						ConfigurationConstants.STORE_PORTLETS_)) {

					Collection portletsList = MerchantConfigurationUtil
							.getConfigurationList(conf.getConfigurationValue(),
									";");
					if (portletsList != null && portletsList.size() > 0) {

						Iterator ii = portletsList.iterator();
						while (ii.hasNext()) {
							String p = (String) ii.next();
							PortletConfiguration pc = (PortletConfiguration) confMap
									.get(p);
							if (pc == null) {
								pc = new PortletConfiguration();
								pc.setModuleName(p);
								confMap.put(p, p);
							}
							prtList.add(pc);
						}
					}
					req.setAttribute("STORE_FRONT_PORTLETS_MAP", confMap);//this one is used by tag lib for getting portlet configuration
					req.setAttribute("STORE_FRONT_PORTLETS", prtList);
					continue;
				}

				else if (conf.getConfigurationKey().equals(
						ConfigurationConstants.G_API)) {
					if (!StringUtils.isBlank(conf.getConfigurationValue())) {
						req.setAttribute("ANALYTICS", conf
								.getConfigurationValue());
					}
					if (!StringUtils.isBlank(conf.getConfigurationValue1())) {
						req
								.setAttribute("G_API", conf
										.getConfigurationValue1());
					}
					continue;
				}

				else if (conf.getConfigurationKey().equals(
						ShippingConstants.MODULE_SHIPPING_RT_MODULE_INDIC_NAME)) {
					if (!StringUtils.isBlank(conf.getConfigurationValue1())) {
						shippingModule = conf.getConfigurationValue1();
					}
					continue;
				}

				else if (conf
						.getConfigurationKey()
						.equals(
								ShippingConstants.MODULE_SHIPPING_INDIC_COUNTRIES_COSTS)) {
					if (!StringUtils.isBlank(conf.getConfigurationValue())
							&& conf.getConfigurationValue().equals("true")) {
						if (!StringUtils.isBlank(conf.getConfigurationValue1())) {
							shippingRegionLine = conf.getConfigurationValue1();
						}
					}
					continue;
				}

				else if (conf.getConfigurationKey().equals(
						ShippingConstants.MODULE_SHIPPING_ESTIMATE_BYCOUNTRY)) {
					if (!StringUtils.isBlank(conf.getConfigurationValue1())) {
						shippingEstimateLine = conf.getConfigurationValue1();
					}
					continue;
				}

				else if (conf.getConfigurationKey().equals(
						ShippingConstants.MODULE_SHIPPING_ZONES_SHIPPING)) {
					if (!StringUtils.isBlank(conf.getConfigurationValue())) {
						if (ShippingConstants.INTERNATIONAL_SHIPPING
								.equals(conf.getConfigurationValue())) {
							shippingType = ShippingType.INTERNATIONAL;
						} else if (ShippingConstants.DOMESTIC_SHIPPING
								.equals(conf.getConfigurationValue())) {
							shippingType = ShippingType.NATIONAL;
						}
					}
					continue;
				}
			}// end while

			ShippingEstimate estimate = new ShippingEstimate();
			estimate.setLocale(locale);
			estimate.setCurrency(mStore.getCurrency());
			estimate.setShippingType(shippingType);
			estimate.setShippingModule(shippingModule);

			if (!StringUtils.isBlank(shippingRegionLine)) {
				Map m = ShippingUtil.buildShippingPriceRegionMap("",
						shippingRegionLine, shippingEstimateLine);
				estimate.setRegions(m);
			}

			String currentCountryIsoCode = locale.getCountry();

			CountryDescription zoneDescription = rservice
					.getCountryDescriptionByIsoCode(currentCountryIsoCode,
							LanguageUtil.getLanguageNumberCode(locale
									.getLanguage()));
			CountryDescription storeZoneDescription = rservice
					.getCountryDescriptionByCountryId(mStore.getCountry(),
							LanguageUtil.getLanguageNumberCode(locale
									.getLanguage()));
			if (zoneDescription != null) {
				estimate.setCustomerCountry(zoneDescription.getCountryName());
			}
			estimate.setStoreCountry(storeZoneDescription.getCountryName());
			if (!StringUtils.isBlank(shippingModule)) {
				CoreModuleService cms = rservice.getCoreModuleService(
						currentCountryIsoCode, shippingModule);
				if (cms != null) {
					estimate.setShippingCompanyLogo(cms
							.getCoreModuleServiceLogoPath());
				}
			}

			// iterate and get the index
			if (!StringUtils.isBlank(shippingRegionLine)) {
				StringTokenizer cvtk = new StringTokenizer(shippingRegionLine,
						"|");

				int index = 1;
				boolean countryFound = false;
				while (cvtk.hasMoreTokens() && !countryFound) {

					String countryline = cvtk.nextToken();// maxpound:price,maxpound:price...|
					if (!countryline.equals("*")) {
						StringTokenizer countrystk = new StringTokenizer(
								countryline, ";");
						String country = null;
						while (countrystk.hasMoreTokens()) {
							country = countrystk.nextToken();
							if (currentCountryIsoCode != null
									&& country.equals(currentCountryIsoCode)) {
								estimate.setCustomerZoneIndex(index);
								countryFound = true;
								break;
							}
						}
					}
					index++;
				}
			}

			/**
			 * Labels required for the 
			 * Custom portlets SECTION 75
			 * Shipping fees 02
			 * Custom page links 70
			 * Since those are displayed on each page, need to get them on each request
			 */

			Collection<DynamicLabel> dynamicLabels = null;

			try {
				dynamicLabels = (Collection) cache.getFromCache(
						Constants.CACHE_LABELS + "_" + LabelConstants.STORE_FRONT_CUSTOM_PORTLETS + "_" + locale.getLanguage(),
						mStore);
			} catch (Exception ignore) {

			}

			if (dynamicLabels == null) {

				// get from missed
				boolean missed = false;
				try {
					missed = (Boolean) cache.getFromCache(
							Constants.CACHE_LABELS + "_" + LabelConstants.STORE_FRONT_CUSTOM_PORTLETS + "_MISSED_" + locale.getLanguage(), mStore);
				} catch (Exception ignore) {

				}

				if (!missed) {

					// get all dynamic labels
					// get shipping, custom portlets, custom pages [section_id in]
					
					List sections = new ArrayList();
					sections.add(LabelConstants.SHIPPING_FEES_SECTION);
					sections.add(LabelConstants.STORE_FRONT_CUSTOM_PAGES);
					sections.add(LabelConstants.STORE_FRONT_CUSTOM_PORTLETS);
					
					dynamicLabels = rservice.getDynamicLabels(mStore
							.getMerchantId(), sections, locale);

					if (dynamicLabels != null && dynamicLabels.size() > 0) {

						try {
							cache.putInCache(Constants.CACHE_LABELS + "_" + LabelConstants.STORE_FRONT_CUSTOM_PORTLETS + "_" + locale.getLanguage(), dynamicLabels,
									Constants.CACHE_LABELS, mStore);
						} catch (Exception e) {
							log.error(e);
						}

					} else {

						try {
							cache.putInCache(Constants.CACHE_LABELS + "_" + LabelConstants.STORE_FRONT_CUSTOM_PORTLETS + "_MISSED_" + locale.getLanguage(), true,
									Constants.CACHE_LABELS, mStore);
						} catch (Exception e) {
							log.error(e);
						}

					}

				}// end missed

			}

			if (dynamicLabels != null && dynamicLabels.size() > 0) {

				Iterator it = dynamicLabels.iterator();
				List topNavList = new ArrayList();
				List customPortlets = new ArrayList();
				while (it.hasNext()) {

					DynamicLabel dl = (DynamicLabel) it.next();
					if (dl.getSectionId() == LabelConstants.SHIPPING_FEES_SECTION) {
						estimate.setDefaultShippingEstimateText(dl
								.getDynamicLabelDescription()
								.getDynamicLabelDescription());
						continue;
					}

					// custom pages
					if (dl.getSectionId() == LabelConstants.STORE_FRONT_CUSTOM_PAGES) {
						topNavList.add(dl);
						continue;
					}

					// custom portlets
					// set in a map [title, dl]
					if (dl.getSectionId() == LabelConstants.STORE_FRONT_CUSTOM_PORTLETS) {
						if (dl.isVisible()) {
							if (customPortletsMap == null) {
								customPortletsMap = new HashMap();
							}
							PortletConfiguration pc = new PortletConfiguration();
							pc.setContent(dl.getDynamicLabelDescription()
									.getDynamicLabelDescription());
							pc.setCustom(true);
							pc.setPosition(dl.getPosition());
							pc.setModuleName(dl.getTitle());
							List portlets = customPortletsMap.get(dl
									.getPosition());
							if (portlets == null) {
								portlets = new ArrayList();
								customPortletsMap.put(dl.getPosition(),
										portlets);
							}
							portlets.add(pc);
						}
						continue;
					}
				}
				if(topNavList.size()>0) {
					req.setAttribute("TOPNAV", topNavList);
				}
				if (customPortletsMap.size()>0) {
					req.setAttribute("CUSTOMPORTLETS", customPortletsMap);
				}
			}
			req.setAttribute("SHIPPING", estimate);

		}// end if

		Object o = invoke.getAction();
		if (o instanceof SalesManagerBaseAction) {
			SalesManagerBaseAction action = ((SalesManagerBaseAction) invoke
					.getAction());
			action.setPageTitle(mStore.getStorename());
		}

		/** set objects in request **/
		Customer customer = SessionUtil.getCustomer(req);
		if (customer != null) {
			req.setAttribute("CUSTOMER", customer);
		}


		req.setAttribute("merchantId", mStore.getMerchantId());

		/**
		 * Messages
		 */
		String errMessage = MessageUtil.getErrorMessage(req);
		if (!StringUtils.isBlank(errMessage)) {
			SalesManagerBaseAction action = (SalesManagerBaseAction) invoke
					.getAction();
			action.addActionError(errMessage);
		}

		String message = MessageUtil.getMessage(req);
		if (!StringUtils.isBlank(message)) {
			SalesManagerBaseAction action = (SalesManagerBaseAction) invoke
					.getAction();
			action.addActionMessage(message);
		}
		MessageUtil.resetMessages(req);

		// do common stuff
		String r = doIntercept(invoke, req, resp);
		return r;

	}

	protected abstract String doIntercept(ActionInvocation invoke,
			HttpServletRequest req, HttpServletResponse resp) throws Exception;

}



```
