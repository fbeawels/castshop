# LandingAction.java

## Review

## 1. Summary  

**Purpose**  
`LandingAction` is a Struts‑2 action that prepares data for the store’s landing page.  
It pulls two main pieces of content:

| Data | What it represents | Source |
|------|--------------------|--------|
| **Featured products** | A list of products to be highlighted on the home page | `CatalogService.getProductRelationShip` (cached) |
| **Slides** | A collection of dynamic labels that will be displayed in a slider | `ReferenceService.getDynamicLabels` (cached) |

The action also sets the page meta‑information (keywords, description, title, body text) by extracting specific dynamic labels.

**Key components**

* **Caching** – Uses a custom `CacheModule` to store/retrieve both labels and product lists, with a “missed” flag to avoid repeated lookups when data is missing.
* **ReferenceService** – Provides dynamic labels keyed by merchant/store.
* **CatalogService** – Provides product relationships (e.g., featured items).
* **Utilities** – `LocaleUtil`, `LanguageUtil`, `SessionUtil`, and `SpringUtil` are used to resolve locale, language codes, merchant context, and to obtain Spring beans.

**Patterns / Frameworks**

* **Action‑Controller** (Struts‑2) – `LandingAction` extends `SalesManagerBaseAction`.
* **Singleton / Service Factory** – Services are fetched via a custom `ServiceFactory`.
* **Cache-as-Cache** – Implements a simple cache with a “missed” flag to prevent cache thrashing.
* **Dependency Injection** – Spring is used to obtain the `CacheModule` bean.

---

## 2. Detailed Description  

### Flow of execution

1. **`displayLanding()`** is invoked by the Struts framework when the landing page is requested.
2. `reset()` is called (inherited from `SalesManagerBaseAction`) to clear any previous action state.
3. Merchant information is retrieved from the session (`SessionUtil.getMerchantStore`).  
   If a store is present, the action continues; otherwise it silently returns a success with no data.
4. **Locale & language** are resolved from the request, used to fetch the right language‑specific labels and products.
5. **Dynamic labels**  
   * Attempt to get them from cache (`Constants.CACHE_LABELS` + locale).  
   * If not cached, check a “missed” flag.  
   * If not missed, fetch the labels via `ReferenceService.getDynamicLabels`.  
   * Cache the result or set the missed flag if null.  
   * Iterate over the labels and set meta‑information or build the `slides` collection.
6. **Featured products**  
   * Try to load from cache (`Constants.CACHE_FEATURED_ITEMS`).  
   * If missing, check missed flag.  
   * If not missed, call `CatalogService.getProductRelationShip` with relationship ID `-1` (home page).  
   * Convert entity locales (`LocaleUtil.setLocaleToEntityCollection`).  
   * Cache the result or set the missed flag if none found.  
7. Assign the product collection to `featuredProducts`.
8. Return `"SUCCESS"` to forward to the Struts view.

### Dependencies and assumptions

| Dependency | Type | Notes |
|------------|------|-------|
| `CacheModule` | Spring bean | Caches per‑merchant data. |
| `ReferenceService` | Service | Must provide `getDynamicLabels`. |
| `CatalogService` | Service | Provides product relationships. |
| `ServiceFactory` | Utility | Singleton holder for services. |
| `LocaleUtil`, `LanguageUtil`, `SessionUtil`, `SpringUtil` | Utilities | Custom to the application. |
| `SalesManagerBaseAction` | Base action | Provides `getServletRequest`, `getLocale`, and possibly `reset()`. |

The code assumes:

* A `MerchantStore` is always present in the session for a valid landing page request.
* The cache module supports a `getFromCache` and `putInCache` API.
* Language codes are integers retrieved via `LanguageUtil.getLanguageNumberCode`.
* The `Product` entity can be locale‑adjusted in bulk via `LocaleUtil.setLocaleToEntityCollection`.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Returns / Side‑Effects |
|--------|---------|------------|------------------------|
| `displayLanding()` | Main action entry point. Builds meta data, slides, and featured products. | None (uses request/session context) | `String` – Struts result name (`SUCCESS`). Side‑effects: populates `featuredProducts`, `slides`, and page meta fields. |
| `getFeaturedProducts()` | Getter for the collection of featured products. | None | `Collection<Product>` |
| `setFeaturedProducts(Collection<Product>)` | Setter used by the framework to expose data to the view. | `Collection<Product>` | None |
| `getSlides()` | Getter for slide labels. | None | `Collection<DynamicLabel>` |
| `setSlides(Collection<DynamicLabel>)` | Setter for slide labels. | `Collection<DynamicLabel>` | None |

### Reusable utilities

* `reset()` – inherited from `SalesManagerBaseAction`, clears previous state (exact implementation unknown).
* `LocaleUtil.setLocaleToEntityCollection` – converts entities to the current locale.
* `SessionUtil.getMerchantStore` – fetches the store context from the session.

---

## 4. Dependencies  

| Library / Package | Role | Standard / Third‑Party |
|-------------------|------|------------------------|
| `org.apache.log4j.Logger` | Logging | Third‑party (Log4J) |
| `javax.servlet.http.HttpServletRequest` | Servlet request | Standard |
| `com.salesmanager.*` | Core application code (entities, services, utilities) | In‑house |
| `com.salesmanager.common.SalesManagerBaseAction` | Base Struts action | In‑house |
| `com.salesmanager.core.service.*` | Business services | In‑house |
| `com.salesmanager.core.util.*` | Utility helpers | In‑house |
| `com.salesmanager.core.constants.*` | Constant definitions | In‑house |
| `SpringUtil` | Spring bean lookup | In‑house wrapper around Spring |
| `CacheModule` | Custom cache implementation | In‑house |

No external dependencies beyond the standard Java EE servlet API, Struts‑2, and Log4J.

---

## 5. Additional Notes  

### Strengths  

* **Caching strategy** – Avoids repeated database hits and gracefully handles missing data via a “missed” flag.  
* **Separation of concerns** – Data retrieval is delegated to services; the action only orchestrates and populates the view model.  
* **Locale awareness** – Both labels and product data are locale‑specific, ensuring correct multi‑language support.

### Potential Issues / Edge Cases  

1. **Silent failures** – Many `catch (Exception ignore)` blocks swallow exceptions. If the cache or service throws an error, the user receives an empty page without any indication of the underlying problem.  
2. **Thread safety** – `slides` is lazily initialized without synchronization. In a multi‑threaded environment (e.g., concurrent requests for the same store/locale), two threads might instantiate separate `ArrayList` objects, though the eventual result is identical.  
3. **Hard‑coded relationship ID (`-1`)** – This magic number is used to fetch featured items. A constant would improve readability.  
4. **Missing null checks** – `getServletRequest()` and `super.getLocale()` are called without verifying that they are non‑null. If the framework fails to supply them, a `NullPointerException` may occur.  
5. **Cache key collision** – The cache key strings are built by concatenating locale language codes. If multiple locales share the same language but different country codes, they may inadvertently share a cache entry.  
6. **Unnecessary casts** – `Collection prods = null;` and later `(Collection) cache.getFromCache(...)` rely on raw types; using generics (`Collection<Product>`) would be safer.

### Suggestions for Improvement  

* **Replace empty catch blocks** with proper logging or fallback logic.  
* **Use a dedicated constant** for the featured relationship ID.  
* **Wrap the cache access in helper methods** to centralize “missed” handling and reduce duplication.  
* **Leverage generics** throughout the method to avoid unchecked casts.  
* **Introduce a data‑transfer object (DTO)** for the landing page model instead of exposing raw entity collections directly to the view.  
* **Add unit tests** that mock the services and cache to verify the caching logic and slide filtering (`dl.isVisible()`).  
* **Consider using Spring’s caching abstraction** (e.g., `@Cacheable`) to simplify cache management and enable cache configuration via annotations.  

Overall, the action performs its core function correctly but would benefit from stronger error handling, clearer constants, and modern Java best practices to improve maintainability and robustness.

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
package com.salesmanager.catalog.store;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;

import javax.servlet.http.HttpServletRequest;

import org.apache.log4j.Logger;

import com.salesmanager.common.SalesManagerBaseAction;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.module.model.application.CacheModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class LandingAction extends SalesManagerBaseAction {

	private static final long serialVersionUID = 1L;

	private Logger log = Logger.getLogger(LandingAction.class);

	private Collection<Product> featuredProducts;
	
	private Collection<DynamicLabel> slides;

	public Collection<Product> getFeaturedProducts() {
		return featuredProducts;
	}

	public void setFeaturedProducts(Collection<Product> featuredProducts) {
		this.featuredProducts = featuredProducts;
	}

	private static Logger logger = Logger.getLogger(LandingAction.class);

	public String displayLanding() {

		try {

			// build top seller's products

			reset();

			HttpServletRequest req = super.getServletRequest();

			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());

			int merchantId = Constants.DEFAULT_MERCHANT_ID;

			CacheModule cache = (CacheModule) SpringUtil.getBean("cache");

			if (store != null) {

				merchantId = store.getMerchantId();
				// get header information

				// get the language
				Locale locale = LocaleUtil.getLocale(req);

				int language = LanguageUtil.getLanguageNumberCode(locale
						.getLanguage());

				ReferenceService rservice = (ReferenceService) ServiceFactory
						.getService(ServiceFactory.ReferenceService);

				Collection<DynamicLabel> dynamicLabels = null;

				try {
					dynamicLabels = (Collection) cache
							.getFromCache(Constants.CACHE_LABELS + "_"
									+ locale.getLanguage(), store);
				} catch (Exception ignore) {

				}

				if (dynamicLabels == null) {

					// get from missed
					boolean missed = false;
					try {
						missed = (Boolean) cache.getFromCache(
								Constants.CACHE_LABELS + "_MISSED_"
										+ locale.getLanguage(), store);
					} catch (Exception ignore) {

					}

					if (!missed) {
						
						List sections = new ArrayList();
						sections.add(LabelConstants.STORE_FRONT_LANDING_META_KEYWORDS);
						sections.add(LabelConstants.STORE_FRONT_LANDING_META_DESCRIPTION);
						sections.add(LabelConstants.STORE_FRONT_LANDING_PAGE_TITLE);
						sections.add(LabelConstants.STORE_FRONT_LANDING_DESCRIPTION);
						sections.add(LabelConstants.SLIDER_SECTION);
						
						dynamicLabels = rservice.getDynamicLabels(
								store.getMerchantId(), sections, locale);

						if (dynamicLabels == null) {
							try {
								cache.putInCache(Constants.CACHE_LABELS
										+ "_MISSED_" + locale.getLanguage(),
										true, Constants.CACHE_LABELS, store);
							} catch (Exception e) {
								log.error(e);
							}
						}

					}
				}

				if (dynamicLabels != null && dynamicLabels.size() > 0) {

					Iterator i = dynamicLabels.iterator();

					while (i.hasNext()) {

						DynamicLabel dl = (DynamicLabel) i.next();

						if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_META_KEYWORDS) {

							this.setMetaKeywords(dl
									.getDynamicLabelDescription()
									.getDynamicLabelDescription());

						}

						else if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_META_DESCRIPTION) {

							this.setMetaDescription(dl
									.getDynamicLabelDescription()
									.getDynamicLabelDescription());

						}

						else if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_PAGE_TITLE) {

							this.setPageTitle(dl.getDynamicLabelDescription()
									.getDynamicLabelDescription());

						}

						else if (dl.getSectionId() == LabelConstants.STORE_FRONT_LANDING_DESCRIPTION) {

							this.setPageText(dl.getDynamicLabelDescription()
									.getDynamicLabelDescription());

						}
						
						else if (dl.getSectionId() == LabelConstants.SLIDER_SECTION) {
							if(dl.isVisible()) {
								if(slides==null) {
									slides=new ArrayList();
								}
								slides.add(dl);
							}

						}
					}
				}

				Collection prods = null;

				try {
					prods = (Collection) cache.getFromCache(
							Constants.CACHE_FEATURED_ITEMS + "_"
									+ locale.getLanguage(), store);
				} catch (Exception ignore) {

				}

				if (prods == null) {

					// get it from missed cache
					boolean missed = false;
					try {
						missed = (Boolean) cache.getFromCache(
								Constants.CACHE_FEATURED_ITEMS + "_MISSED_"
										+ locale.getLanguage(), store);
					} catch (Exception ignore) {

					}

					if (!missed) {

						// get featured items
						CatalogService cService = (CatalogService) ServiceFactory
								.getService(ServiceFactory.CatalogService);
						// -1 means relationship is attached to home page

						prods = cService
								.getProductRelationShip(
										-1,
										merchantId,
										CatalogConstants.PRODUCT_RELATIONSHIP_FEATURED_ITEMS,
										super.getLocale().getLanguage(), true);

						if (prods != null && prods.size() > 0) {

							LocaleUtil.setLocaleToEntityCollection(prods, super
									.getLocale(), store.getCurrency());

							try {
								cache.putInCache(Constants.CACHE_FEATURED_ITEMS
										+ "_" + locale.getLanguage(), prods,
										Constants.CACHE_PRODUCTS, store);
							} catch (Exception e) {
								log.error(e);
							}

						} else {

							try {
								cache.putInCache(Constants.CACHE_FEATURED_ITEMS
										+ "_MISSED_" + locale.getLanguage(),
										true, Constants.CACHE_PRODUCTS, store);
							} catch (Exception e) {
								log.error(e);
							}

						}

					}
				}

				this.setFeaturedProducts(prods);
			}

		} catch (Exception e) {
			logger.error(e);
		}

		return SUCCESS;

	}

	public Collection<DynamicLabel> getSlides() {
		return slides;
	}

	public void setSlides(Collection<DynamicLabel> slides) {
		this.slides = slides;
	}

}



```
