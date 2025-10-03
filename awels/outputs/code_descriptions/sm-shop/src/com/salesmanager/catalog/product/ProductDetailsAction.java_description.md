# ProductDetailsAction.java

## Review

## 1. Summary

The **`ProductDetailsAction`** class is a Struts‑style action that powers the product detail page of a catalog application.  
Its responsibilities include:

| Feature | What it does | Key classes it touches |
|---------|--------------|------------------------|
| Display a single product | Loads the product by SEO URL, applies locale & currency, formats price, resolves category trail, gathers options/specifications, and loads related items | `CatalogService`, `ProductUtil`, `CategoryUtil`, `LocaleUtil` |
| Display reviews | Retrieves a paginated list of reviews for the product and calculates the average rating | `CatalogService`, `SearchReviewCriteria/Response` |
| Create a review | Validates the user, constructs a `Review`/`ReviewDescription`, saves it and updates the product’s review statistics | `CatalogService`, `Customer`, `MerchantStore` |

The action uses **Apache Commons**, **log4j**, a custom `ServiceFactory`, and Spring’s `ApplicationContext` (`SpringUtil`) to obtain beans such as a cache module.  The code follows a **thin‑controller** approach: the action mixes presentation‑level state (fields that are serialized to the view) with business logic (option extraction, cache handling, price formatting).

## 2. Detailed Description

### Execution Flow

1. **`displayProduct`**  
   * Extracts the SEO URL from the request.  
   * Calls `CatalogService#getProductByMerchantIdAndSeoURLAndByLang` to fetch the product.  
   * If the product is missing, it logs a warning and returns the `"DEFAULT"` result.  
   * Sets page title, meta description, and internationalized fields.  
   * Resolves the price string using `ProductUtil`, taking into account default options.  
   * Builds the category trail via `CategoryUtil`.  
   * Pulls product attributes from the service and groups them into two collections: **read‑only specifications** and **priced options**.  
   * Loads related items from a cache; if the cache misses, it fetches them from the service and populates the cache.  
   * Returns `SUCCESS`.

2. **`displayReviews`**  
   * Builds a `SearchReviewCriteria` (product ID, language, pagination).  
   * Calls the service to retrieve reviews and updates the pagination helpers (`setListingCount`, `setPageElements`).  
   * Sets the `reviews` collection and the average rating counter.  
   * Returns `SUCCESS`.

3. **`createReview`**  
   * Requires an authenticated customer; otherwise it sets an error message and returns `INPUT`.  
   * Re‑fetches the product (to ensure it’s the latest).  
   * Validates that the review text is non‑empty.  
   * Constructs a `Review` and `ReviewDescription`, associates the customer, product, and rating.  
   * Persists it via `CatalogService#addProductReview`.  
   * Updates the product’s review count/average and saves the product.  
   * Sets a success message and returns `SUCCESS`.

4. **`reviewsForm`**  
   * Pre‑loads the product so the form can display the product name and price.

### Assumptions & Constraints

* The action assumes a **single‑threaded** web request lifecycle; it holds state in instance fields that are discarded after each call.  
* Language and currency are obtained from the session/locale; missing values are silently ignored.  
* Caching uses a custom `CacheModule`; if the cache throws, the code falls back to direct DB calls.  
* The code treats *any* exception as a fatal error and sets a generic technical message – no fine‑grained error handling.  
* All collections are raw types (`Collection`, `Set`) instead of generics.

### Architecture & Design Choices

* **MVC (Struts)** – the action is a controller that directly manipulates the view model.  
* **Service Layer** – business logic lives in `CatalogService`; however the action still performs significant data‑gathering (e.g., option grouping).  
* **Utility‑Driven** – `ProductUtil`, `CategoryUtil`, and `LocaleUtil` are stateless helpers for formatting and localization.  
* **Caching** – simple key‑based cache module; no eviction or consistency guarantees visible in this snippet.  
* **Static Configuration** – review list size is read once statically from `config`.

## 3. Functions / Methods

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `createReview()` | Handles review submission. | `product`, `reviewText`, `rating`, customer from session | Result string (`SUCCESS`/`INPUT`) | Saves review, updates product, sets messages |
| `displayReviews()` | Loads a paginated list of reviews. | `product`, page start index | Result string (`SUCCESS`) | Sets `reviews`, `counter`, pagination data |
| `reviewsForm()` | Prepares the review form view. | `product` | Result string (`SUCCESS`) | Loads product price/localization |
| `displayProduct()` | Renders the product detail page. | `product`, URL | Result string (`SUCCESS`) | Populates product details, options, specs, related items, price |
| `getOptions()`, `setOptions(...)` | Accessor for priced options | – | Collection | – |
| `getSpecifications()`, `setSpecifications(...)` | Accessor for read‑only specs | – | Collection | – |
| `getCategoryPath()`, `setCategoryPath(...)` | Accessor for breadcrumb path | – | Collection | – |
| `getProduct()`, `setProduct(...)` | Accessor for the current product | – | Product | – |
| `getProductPrice()`, `setProductPrice(...)` | Accessor for formatted price string | – | String | – |
| `getReviews()`, `setReviews(...)` | Accessor for reviews | – | Collection<Review> | – |
| `getCounter()`, `setCounter(...)` | Accessor for rating statistics | – | Counter | – |
| `getReviewText()`, `setReviewText(...)` | Accessor for review form text | – | String | – |
| `getRating()`, `setRating(...)` | Accessor for rating value | – | int | – |
| `getRelatedItems()`, `setRelatedItems(...)` | Accessor for related products | – | Collection | – |

The bulk of the logic resides in the first four public methods; the remaining are simple JavaBean getters/setters.

## 4. Dependencies

| Library / Module | Type | Purpose |
|------------------|------|---------|
| `org.apache.commons.configuration.Configuration` | Third‑party | Reads static config (review list size) |
| `org.apache.commons.lang.StringUtils` | Third‑party | String blank checks |
| `org.apache.log4j.Logger` | Third‑party | Logging |
| `com.salesmanager.common.PageBaseAction` | Internal | Base Struts action providing pagination helpers |
| `com.salesmanager.common.util.PropertiesHelper` | Internal | Loads configuration |
| `com.salesmanager.core.constants.*` | Internal | Constant values (cache keys, etc.) |
| `com.salesmanager.core.entity.*` | Internal | JPA/Hibernate entities (`Product`, `Review`, …) |
| `com.salesmanager.core.module.model.application.CacheModule` | Internal | Simple cache API |
| `com.salesmanager.core.service.ServiceFactory` | Internal | Service locator (CatalogService) |
| `com.salesmanager.core.service.catalog.CatalogService` | Internal | Business operations on catalog |
| `com.salesmanager.core.util.*` | Internal | Utility helpers for categories, language, locale, products, Spring bean lookup |
| `com.salesmanager.core.util.www.SessionUtil` | Internal | Session helper (customer, store) |
| `javax.servlet` (implied via `PageBaseAction`) | Standard | Servlet API for request/response |

All dependencies are either standard Java / servlet libraries or internal project modules; the only external third‑party libraries are Apache Commons and log4j.

## 5. Additional Notes

### Strengths
* The action cleanly separates *display* and *create* responsibilities via distinct methods.  
* It uses dedicated utilities for locale‑specific formatting and caching, keeping the code DRY in those aspects.  
* Pagination is handled by the base class, allowing consistent navigation across the site.

### Potential Issues & Edge Cases
1. **Raw Collections** – `Collection` and `Set` are used without generics; this can lead to `ClassCastException` at runtime.  
2. **Duplicate Null Check** – In `createReview` the blank check uses `StringUtils.isBlank` twice on the same field; one is redundant.  
3. **Exception Handling** – Catching `Exception` at the top level obscures the root cause.  
   * Logging is performed, but the stack trace is only logged at error level – the user sees a generic technical message.  
   * Some methods swallow exceptions silently (e.g., cache access), which may hide real problems.  
4. **Cache Miss Logic** – The code writes a `"MISSED"` flag into the cache, but it never ever removes it. Once a product has no related items, that flag stays forever, potentially blocking later updates.  
5. **Hard‑coded Strings** – Result names (`"DEFAULT"`, `"AUTHORIZATIONERROR"`, `"GENERICERROR"`) are magic strings; a constants class would improve readability.  
6. **Duplicate Null Test** – The check `if (product == null) { if (product == null) { ... }}` is obviously a copy‑paste error.  
7. **No Thread‑Safety Guarantees** – The class is instantiated per request, so state is isolated; however if the framework re‑uses action instances, the fields may leak.  
8. **Missing Unit Tests** – Given the large amount of business logic in the action, unit tests would help ensure correctness after refactoring.

### Suggested Enhancements
| Area | Recommendation |
|------|----------------|
| **Generics** | Replace raw collections with typed generics (`Collection<ProductOptionDescriptor>`, `Collection<Review>`, etc.). |
| **Business Logic Extraction** | Move option grouping, price formatting, and related‑items retrieval into dedicated service or helper classes. Keep the action thin. |
| **Error Handling** | Create custom exceptions for domain errors (e.g., `ProductNotFoundException`) and handle them in a single `@ExceptionHandler`. |
| **Configuration** | Use Spring’s `@Value` or a dedicated `@ConfigurationProperties` bean instead of the static block. |
| **Cache Refactor** | Use a proper cache abstraction that supports expiration and eviction; avoid manual `"MISSED"` flags. |
| **Internationalization** | Store result names in a `ResultConstants` class; use `ResourceBundle` for error keys. |
| **Unit Tests** | Add JUnit tests for `createReview`, `displayReviews`, and `displayProduct` using mocks for `CatalogService`, `CacheModule`, and `SessionUtil`. |
| **Documentation** | Add Javadoc comments to public methods; explain the expected request parameters and the resulting view model. |
| **Code Clean‑up** | Remove duplicated null checks, unused imports, and redundant code blocks. |

### Future Extensions
* **Review Moderation** – Add an approval workflow for reviews before they are shown.  
* **Dynamic Pricing** – Expand the options logic to handle discounts or bundled pricing.  
* **SEO Friendly URLs** – Expose a method that validates and canonicalizes product URLs.  
* **Analytics** – Hook into product view events for tracking and recommendation engines.  

Overall, the `ProductDetailsAction` achieves its functional goals but would benefit significantly from a refactor that isolates business logic, enforces type safety, and adopts modern error handling patterns.

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
package com.salesmanager.catalog.product;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Set;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.common.PageBaseAction;
import com.salesmanager.common.util.PropertiesHelper;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.ProductAttribute;
import com.salesmanager.core.entity.catalog.ProductOption;
import com.salesmanager.core.entity.catalog.ProductOptionDescriptor;
import com.salesmanager.core.entity.catalog.ProductOptionValue;
import com.salesmanager.core.entity.catalog.Review;
import com.salesmanager.core.entity.catalog.ReviewDescription;
import com.salesmanager.core.entity.catalog.SearchReviewCriteria;
import com.salesmanager.core.entity.catalog.SearchReviewResponse;
import com.salesmanager.core.entity.common.Counter;
import com.salesmanager.core.entity.common.I18NEntity;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.module.model.application.CacheModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.CategoryUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.ProductUtil;
import com.salesmanager.core.util.SpringUtil;
import com.salesmanager.core.util.www.SessionUtil;

/**
 * Product details and product reviews
 * 
 * @author Carl Samson
 * 
 */
public class ProductDetailsAction extends PageBaseAction {

	private static Logger logger = Logger.getLogger(ProductDetailsAction.class);
	private static Configuration config = PropertiesHelper.getConfiguration();

	private static int size = 0;

	static {

		size = config.getInt("catalog.reviewslist.maxsize", 10);

	}

	private Product product;
	private String productPrice;

	private Collection categoryPath;// category trail

	private Collection<ProductOptionDescriptor> specifications = new ArrayList();// read
																					// only
																					// attributes
	private Collection<ProductOptionDescriptor> options = new ArrayList();// priced
																			// options

	private Collection<Product> relatedItems;



	// review tab
	private Collection reviews;// review tab

	private Counter counter;// review tab (average rating)

	// create review
	private String reviewText;// review form

	private int rating = 1;// review form

	public String createReview() {

		try {

			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());

			// requires Customer
			Customer customer = SessionUtil.getCustomer(super
					.getServletRequest());
			if (customer == null) {
				super.setMessage("message.review.loggedin");
				return INPUT;
			}

			// product details
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			product = cservice.getProduct(this.getProduct().getProductId());
			product.setLocale(super.getLocale());

			// create review

			// check text
			if (StringUtils.isBlank(this.getReviewText())
					|| StringUtils.isBlank(this.getReviewText())) {
				super.setErrorMessage("error.messag.review");
				return INPUT;
			}

			Review r = new Review();
			r.setCustomerId(customer.getCustomerId());
			r.setCustomerName(customer.getName());
			r.setDateAdded(new Date());
			r.setLastModified(new Date());
			r.setProductId(product.getProductId());
			r.setProductName(product.getName());
			r.setReviewRating(this.getRating());

			r.setLocale(super.getLocale());

			ReviewDescription description = new ReviewDescription();
			description.setReviewText(this.getReviewText());

			Set s = new HashSet();
			s.add(description);

			r.setDescriptions(s);
			
			cservice.addProductReview(store, r);
			
			counter = cservice.countAverageRatingPerProduct(this.getProduct()
					.getProductId());
			
			if(counter!=null) {
			
				double average = counter.getAverage();
				BigDecimal bdaverage = new BigDecimal(average);
				bdaverage.setScale(2, BigDecimal.ROUND_HALF_EVEN);
				product.setProductReviewCount(counter.getCount());
				product.setProductReviewAvg(bdaverage);
				cservice.saveOrUpdateProduct(product);
			}
			
			super.setMessage("message.review.created");

		} catch (Exception e) {
			logger.error(e);
			super.setTechnicalMessage();
			return INPUT;
		}

		return SUCCESS;

	}

	public String displayReviews() {

		if (this.getProduct() == null) {
			return "AUTHORIZATIONERROR";
		}

		try {

			Locale locale = super.getLocale();

			setSize(size);

			SearchReviewCriteria criteria = new SearchReviewCriteria();
			criteria.setProductId(this.getProduct().getProductId());
			criteria.setLanguageId(LanguageUtil.getLanguageNumberCode(locale
					.getLanguage()));
			criteria.setQuantity(this.getSize());
			criteria.setStartindex(super.getPageStartIndex());

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);

			SearchReviewResponse response = cservice
					.searchProductReviewsByProduct(criteria);
			reviews = response.getReviews();

			super.setListingCount(response.getCount());
			super.setRealCount(reviews.size());
			super.setPageElements();

			LocaleUtil.setLocaleToEntityCollection(reviews, super.getLocale());

			// calculate average
			counter = cservice.countAverageRatingPerProduct(this.getProduct()
					.getProductId());

		} catch (Exception e) {
			logger.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;
	}

	public String reviewsForm() {

		try {

			MerchantStore store = SessionUtil.getMerchantStore(super
					.getServletRequest());

			// product details
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			product = cservice.getProduct(this.getProduct().getProductId());
			product.setLocale(super.getLocale(), store.getCurrency());

		} catch (Exception e) {
			logger.error(e);
		}

		return SUCCESS;

	}

	public String displayProduct() {

		try {

			CacheModule cache = (CacheModule) SpringUtil.getBean("cache");

			String url = super.getRequestedEntityId();
			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			MerchantStore store = (MerchantStore) super.getServletRequest()
					.getSession().getAttribute("STORE");
			Locale locale = (Locale) super.getLocale();
			product = cservice.getProductByMerchantIdAndSeoURLAndByLang(store
					.getMerchantId(), url, locale.getLanguage());

			if (product == null) {
				if (product == null) {
					logger.warn("Product having seUrl " + url
							+ " does not exist");
					return "DEFAULT";
				}
			}

			if(!StringUtils.isBlank(product.getProductDescription().getProductTitle())) {
				this.setPageTitle(product.getProductDescription().getProductTitle());
			} else {
				this.setPageTitle(product.getName());
			}

			this.setMetaDescription(product.getDescription());

			((I18NEntity) product).setLocale(super.getLocale(), store
					.getCurrency());

			Set prices = product.getPrices();

			LocaleUtil.setLocaleToEntityCollection(prices, locale, store
					.getCurrency());

			// for category trail
			categoryPath = CategoryUtil.getCategoryPath(super.getLocale()
					.getLanguage(), store.getMerchantId(), product
					.getMasterCategoryId());

			// options - attributes
			Collection attributes = cservice.getProductAttributes(product
					.getProductId(), locale.getLanguage());

			Collection defaultOptions = new ArrayList();

			if (attributes != null && attributes.size() > 0) {

				// extract read only
				Iterator i = attributes.iterator();

				long lastOptionId = -1;
				long lastSpecificationOptionId = -1;
				ProductOptionDescriptor pod = null;

				while (i.hasNext()) {

					ProductAttribute pa = (ProductAttribute) i.next();

					ProductOption po = pa.getProductOption();
					ProductOptionValue pov = pa.getProductOptionValue();
					if (po != null) {

						if (pa.isAttributeDisplayOnly()) {

							if (lastSpecificationOptionId == -1) {
								lastSpecificationOptionId = po
										.getProductOptionId();
								pod = new ProductOptionDescriptor();
								pod.setOptionType(po.getProductOptionType());
								pod.setName(po.getName());
								specifications.add(pod);
							} else {
								if (pa.getOptionId() != lastOptionId) {
									lastSpecificationOptionId = po
											.getProductOptionId();
									pod = new ProductOptionDescriptor();
									pod
											.setOptionType(po
													.getProductOptionType());
									pod.setName(po.getName());
									specifications.add(pod);
								}
							}

						} else {// option

							if (lastOptionId == -1) {
								lastOptionId = po.getProductOptionId();
								pod = new ProductOptionDescriptor();
								pod.setOptionType(po.getProductOptionType());
								pod.setName(po.getName());
								options.add(pod);
								if (pa.isAttributeDefault()) {
									defaultOptions.add(pa);
								}

							} else {
								if (pa.getOptionId() != lastOptionId) {
									lastOptionId = po.getProductOptionId();
									pod = new ProductOptionDescriptor();
									pod
											.setOptionType(po
													.getProductOptionType());
									pod.setName(po.getName());
									options.add(pod);
									if (pa.isAttributeDefault()) {
										defaultOptions.add(pa);
									}
								}
							}

						}

						pod.addValue(pa);
						pod.setOptionId(pa.getOptionId());
						if (pa.isAttributeDefault()) {
							pod.setDefaultOption(pa.getProductAttributeId());
						}
					}
				}

			}

			if (defaultOptions != null && defaultOptions.size() > 0) {
				this.setProductPrice(ProductUtil
						.formatHTMLProductPriceWithAttributes(
								super.getLocale(), store.getCurrency(), this
										.getProduct(), defaultOptions, true));
			} else {
				this.setProductPrice(ProductUtil.formatHTMLProductPrice(super
						.getLocale(), store.getCurrency(), this.getProduct(),
						true, false));
			}

			// related items
			relatedItems = null;
			try {
				relatedItems = (Collection) cache.getFromCache(
						Constants.CACHE_RELATED_ITEMS + product.getProductId()
								+ "_" + locale.getLanguage(), store);
			} catch (Exception ignore) {

			}

			if (relatedItems == null) {

				// get it from missed cache
				boolean missed = false;
				try {
					missed = (Boolean) cache.getFromCache(
							Constants.CACHE_RELATED_ITEMS
									+ product.getProductId() + "_MISSED_"
									+ locale.getLanguage(), store);
				} catch (Exception ignore) {

				}

				if (!missed) {

					Collection r = cservice
							.getProductRelationShip(
									this.getProduct().getProductId(),
									store.getMerchantId(),
									CatalogConstants.PRODUCT_RELATIONSHIP_RELATED_ITEMS,
									super.getLocale().getLanguage(), true);

					if (r != null && r.size() > 0) {

						LocaleUtil.setLocaleToEntityCollection(r, super
								.getLocale(), store.getCurrency());

						relatedItems = r;

						try {
							cache.putInCache(Constants.CACHE_RELATED_ITEMS
									+ product.getProductId() + "_"
									+ locale.getLanguage(), relatedItems,
									Constants.CACHE_PRODUCTS, store);
						} catch (Exception ignore) {

						}

					} else {

						try {
							cache.putInCache(Constants.CACHE_RELATED_ITEMS
									+ product.getProductId() + "_MISSED_"
									+ locale.getLanguage(), true,
									Constants.CACHE_PRODUCTS, store);
						} catch (Exception ignore) {

						}

					}

				}

			}

		} catch (Exception e) {
			logger.error(e);
			List msg = new ArrayList();
			msg.add(e.getMessage());
			super.setActionErrors(msg);
			return "GENERICERROR";
		}

		return SUCCESS;

	}

	public Collection getOptions() {
		return options;
	}

	public void setOptions(Collection options) {
		this.options = options;
	}

	public Collection getSpecifications() {
		return specifications;
	}

	public void setSpecifications(Collection specifications) {
		this.specifications = specifications;
	}


	public Collection getCategoryPath() {
		return categoryPath;
	}

	public void setCategoryPath(Collection categoryPath) {
		this.categoryPath = categoryPath;
	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public String getProductPrice() {
		return productPrice;
	}

	public void setProductPrice(String productPrice) {
		this.productPrice = productPrice;
	}

	public Collection<Review> getReviews() {
		return reviews;
	}

	public void setReviews(Collection<Review> reviews) {
		this.reviews = reviews;
	}

	public Counter getCounter() {
		return counter;
	}

	public void setCounter(Counter counter) {
		this.counter = counter;
	}

	public String getReviewText() {
		return reviewText;
	}

	public void setReviewText(String reviewText) {
		this.reviewText = reviewText;
	}

	public int getRating() {
		return rating;
	}

	public void setRating(int rating) {
		this.rating = rating;
	}

	public Collection getRelatedItems() {
		return relatedItems;
	}

	public void setRelatedItems(Collection relatedItems) {
		this.relatedItems = relatedItems;
	}

}



```
