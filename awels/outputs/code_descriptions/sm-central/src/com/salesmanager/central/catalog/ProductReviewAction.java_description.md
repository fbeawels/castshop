# ProductReviewAction.java

## Review

## 1. Summary  
`ProductReviewAction` is a Struts‑style action that handles displaying and deleting reviews for a specific product. It interacts with a `CatalogService` to query reviews (`searchProductReviewsByProduct`) and to delete a review (`deleteProductReview`). The action supports pagination (page size and start index) and language localization of review data.  

Key components  
| Component | Role |
|-----------|------|
| `PageBaseAction` | Provides common paging, messaging and localization utilities. |
| `CatalogService` | Business service that accesses the underlying persistence layer for products and reviews. |
| `SearchReviewCriteria` / `SearchReviewResponse` | DTOs used to carry search parameters and results. |
| `PropertiesHelper` | Loads configuration values from a properties file. |
| `Logger` | Emits debug/error messages. |

The code is largely straightforward, but a few design decisions and implementation details merit closer inspection.

---

## 2. Detailed Description  

### Flow of execution  
1. **Initialization** – The static block reads the configuration key `central.reviewlist.maxsize` to determine the default page size (`size`).  
2. **`reviewProduct()`** –  
   * Sets the page title.  
   * Validates that a `Product` instance has been supplied.  
   * Builds a `SearchReviewCriteria` with product ID, language ID (derived from the current locale), pagination parameters.  
   * Calls `CatalogService.searchProductReviewsByProduct(criteria)` to obtain a `SearchReviewResponse`.  
   * Stores the list of reviews and pagination metadata, then localizes each review entity via `LocaleUtil`.  
3. **`removeReview()`** –  
   * Retrieves the `Review` to be deleted.  
   * If found, invokes `CatalogService.deleteProductReview(r)` and triggers a success message.  
   * Calls `reviewProduct()` to refresh the list.

### Assumptions & constraints  
| Assumption | Impact |
|------------|--------|
| `product` is pre‑populated (e.g., via form binding or session). | No validation of product existence beyond a null check. |
| The current locale is available from `PageBaseAction`. | All language‑specific lookups rely on that locale. |
| The configuration file is readable at class‑load time. | Changing the config at runtime does **not** propagate. |
| The paging indices (`size`, `pageStartNumber`) are trusted inputs. | No bounds checking or sanitization. |

### Architecture  
The class follows a thin controller pattern: it delegates all business logic to `CatalogService` and focuses on preparing data for the view. The design relies heavily on the Struts action model (returning `SUCCESS` or `unauthorized`) and on a custom paging framework (presumably defined in `PageBaseAction`). This separation of concerns is reasonable, but the code could benefit from stronger type safety and modern Java practices (e.g., generics, optional, dependency injection).

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `reviewProduct()` | Fetches and paginates reviews for the current `product`. | None (uses `product`, `locale`). | Returns `SUCCESS` or `unauthorized`. | Sets pagination data, populates `reviews`, localizes entities, logs errors. |
| `removeReview()` | Deletes a review identified by `reviewId`. | None (uses `reviewId`). | Returns `SUCCESS` or `unauthorized`. | Deletes review via service, sets success/technical messages, refreshes review list. |
| `getProduct()` / `setProduct(Product)` | Accessor for the product under review. | None / `Product`. | `Product`. | None. |
| `getReviews()` / `setReviews(Collection)` | Accessor for the list of reviews. | None / `Collection`. | `Collection`. | None. |
| `getReviewId()` / `setReviewId(long)` | Accessor for the ID of the review to delete. | None / `long`. | `long`. | None. |

### Reusable / Utility methods  
* `LocaleUtil.setLocaleToEntityCollection(reviews, locale)` – localizes each review entity.  
* `LanguageUtil.getLanguageNumberCode(locale.getLanguage())` – maps locale language to an internal numeric code.  
* `PropertiesHelper.getConfiguration()` – loads application configuration.  

---

## 4. Dependencies  

| Library / Framework | Role | Standard / 3rd‑party |
|---------------------|------|----------------------|
| `org.apache.commons.configuration.Configuration` | Holds config values. | 3rd‑party |
| `org.apache.log4j.Logger` | Logging. | 3rd‑party |
| `com.salesmanager.central.PageBaseAction` | Base action, paging utilities. | Project‑specific |
| `com.salesmanager.core.*` | Entity and service classes (`Product`, `Review`, `CatalogService`, etc.). | Project‑specific |
| `java.util.*` | Collections, Locale. | JDK |

**Platform assumptions**  
* The action is used within a Struts 1/2 environment (method names returning string constants).  
* The locale information is provided by the base action.  
* The application uses a custom configuration file loaded via `PropertiesHelper`.

---

## 5. Additional Notes  

### 5.1 Code‑quality observations  
1. **Logger initialization** – Uses `EditProductAction.class` instead of `ProductReviewAction.class`. Likely a copy‑paste oversight.  
2. **Generics** – The `reviews` field and its getter/setter use raw `Collection`. Modern Java code should use `List<Review>` to enforce type safety.  
3. **Exception handling** – All caught exceptions are logged but the action still returns `SUCCESS`. The view may display stale data or misleading status messages. Consider propagating errors or setting an error flag.  
4. **Static configuration** – `size` is loaded once; dynamic changes to the config file won’t be reflected without restarting the application.  
5. **Thread safety** – The static `size` and `config` objects are effectively read‑only after initialization, so they’re thread‑safe. However, the `reviews` collection is instance‑specific and thus safe per request.  
6. **Null handling** – `reviewProduct()` only checks that `product` is not null, but it does not guard against `product.getProductId()` being null or zero.  
7. **Pagination logic** – The method sets `super.setSize(size)` and `super.setPageStartNumber()` but does not validate that `size` is positive or that the start index is within bounds.  
8. **Localization** – Calls to `LocaleUtil.setLocaleToEntityCollection` assume that all review fields require localization. If the Review entity contains non‑localized fields, this may be unnecessary overhead.  

### 5.2 Edge cases  
* **Empty review list** – `response.getReviews()` may return `null` or an empty collection. The code assumes a non‑null collection, which could lead to `NullPointerException`.  
* **Large product IDs** – If `product.getProductId()` returns a value outside the supported range, the service may throw an exception.  
* **Concurrent deletion** – Two users deleting the same review simultaneously could result in a race condition (the second deletion would log “No review exist…”).  
* **Locale not supported** – `LanguageUtil.getLanguageNumberCode` may return an invalid code if the locale isn’t mapped; the service might return no results or throw an exception.

### 5.3 Future enhancements  
| Enhancement | Rationale |
|-------------|-----------|
| **Use dependency injection** – Inject `CatalogService` instead of retrieving it via a static `ServiceFactory`. Improves testability and decouples the action from the service locator pattern. |
| **Adopt generics** – Change `Collection reviews` to `List<Review>`. |
| **Refactor error handling** – Introduce a standardized error message framework; return error results instead of `SUCCESS` when an exception occurs. |
| **Update logging** – Use Log4j 2 or SLF4J, and correct the logger class reference. |
| **Dynamic config reload** – Watch the configuration file for changes or provide a runtime refresh method for `size`. |
| **Add unit tests** – Mock the service layer to validate pagination and deletion logic. |
| **Input validation** – Sanitize and validate `reviewId`, `size`, and page indices to prevent invalid queries. |
| **Asynchronous deletion** – For high‑traffic scenarios, consider async or batch deletion to reduce lock contention. |
| **Internationalization** – Centralize all i18n keys in a resource bundle and expose them via a utility class. |

---  

**Overall**, `ProductReviewAction` fulfills its basic responsibilities, but modernizing the code (generics, DI, improved error handling) and tightening the contract around pagination and localization would make it more robust, maintainable, and testable.

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
package com.salesmanager.central.catalog;

import java.util.Collection;
import java.util.Locale;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import com.salesmanager.central.PageBaseAction;
import com.salesmanager.central.util.PropertiesHelper;
import com.salesmanager.core.entity.catalog.Product;
import com.salesmanager.core.entity.catalog.Review;
import com.salesmanager.core.entity.catalog.SearchReviewCriteria;
import com.salesmanager.core.entity.catalog.SearchReviewResponse;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;

public class ProductReviewAction extends PageBaseAction {

	private static Logger log = Logger.getLogger(EditProductAction.class);
	private static Configuration config = PropertiesHelper.getConfiguration();

	private Product product;
	private Collection reviews;
	private long reviewId;// for deletion

	private static int size = 0;

	static {

		size = config.getInt("central.reviewlist.maxsize", 10);

	}

	public String reviewProduct() {
		
		super.setPageTitle("label.product.review");

		try {

			if (this.getProduct() == null) {
				return "unauthorized";
			}

			Locale locale = super.getLocale();

			super.setSize(size);
			super.setPageStartNumber();

			SearchReviewCriteria criteria = new SearchReviewCriteria();
			criteria.setProductId(this.getProduct().getProductId());
			criteria.setLanguageId(LanguageUtil.getLanguageNumberCode(locale
					.getLanguage()));
			criteria.setQuantity(this.getSize());
			criteria.setStartindex(this.getPageCriteriaIndex());

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			SearchReviewResponse response = cservice
					.searchProductReviewsByProduct(criteria);
			reviews = response.getReviews();

			super.setListingCount(response.getCount());
			super.setRealCount(reviews.size());

			super.setPageElements();

			LocaleUtil.setLocaleToEntityCollection(reviews, super.getLocale());

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;

	}

	public String removeReview() {
		
		super.setPageTitle("label.product.review");

		try {

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			Review r = cservice.getProductReview(this.getReviewId());

			if (r == null) {
				log.warn("No review exist for review id " + this.getReviewId());
				return "unauthorized";
			}

			cservice.deleteProductReview(r);
			super.setSuccessMessage();
			reviewProduct();

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public Product getProduct() {
		return product;
	}

	public void setProduct(Product product) {
		this.product = product;
	}

	public Collection getReviews() {
		return reviews;
	}

	public void setReviews(Collection reviews) {
		this.reviews = reviews;
	}

	public long getReviewId() {
		return reviewId;
	}

	public void setReviewId(long reviewId) {
		this.reviewId = reviewId;
	}

}



```
