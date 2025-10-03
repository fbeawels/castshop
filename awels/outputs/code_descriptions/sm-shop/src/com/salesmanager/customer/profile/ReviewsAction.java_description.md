# ReviewsAction.java

## Review

## 1. Summary  

`ReviewsAction` is a Struts‑2 action class (extends `PageBaseAction`) that manages the display and deletion of product reviews for a customer. The action:

* **displayReviews** – retrieves the current customer’s reviews, paginates them, and exposes the list to the view layer.
* **removeReview** – deletes a single review identified by `reviewId`.
* Internally it relies on a `CatalogService` (via `ServiceFactory`) to fetch/delete reviews and on helper utilities for locale, language, and session handling.

Key components:
* `SearchReviewCriteria` & `SearchReviewResponse` – DTOs for search queries.
* `PageBaseAction` – provides pagination helpers (`setSize`, `setPageStartNumber`, etc.).
* Configuration via `PropertiesHelper` to determine page size.

The design follows a classic service‑layer separation: the action delegates business logic to `CatalogService`, keeping the controller thin.  

---

## 2. Detailed Description  

### Core Flow  

| Step | Method | Description |
|------|--------|-------------|
| **Init** | `displayReviews` (or `removeReview`) | Called by Struts. Sets pagination parameters and invokes helper methods. |
| **Query** | `getCriteria` | Builds a `SearchReviewCriteria` from the current request context (merchant, customer, language). |
| **Service Call** | `reviewsQuery` | Calls `CatalogService.searchProductReviewsByCustomer`. Receives a `SearchReviewResponse`. |
| **Populate** | In `reviewsQuery` | Assigns the review list to `reviews`, localises entities, updates pagination counters (`setListingCount`, `setRealCount`, `setPageElements`). |
| **Deletion** | `removeReview` | Retrieves the review via `CatalogService.getProductReview`, verifies existence, then calls `deleteProductReview`. Sets a user message. |
| **Return** | Both actions | Return `SUCCESS` or `AUTHORIZATIONERROR` as per Struts conventions. |

### Assumptions & Constraints  

* The user must be logged in and the session contains a `Customer` and `MerchantStore` (via `SessionUtil`).
* Reviews are filtered by the logged‑in customer; there is no cross‑customer visibility.
* The `size` (page size) is read once at class load time from the configuration property `catalog.reviewslist.maxsize`.  
* The action is tightly coupled to the Struts 2 framework (`PageBaseAction`), so unit testing would require mocking the action context.
* No explicit error handling for database connection failures or service timeouts – they are caught generically and result in a technical error message.

### Architecture  

The class follows a typical MVC controller pattern:

* **Model** – `Review`, `SearchReviewCriteria`, `SearchReviewResponse`.  
* **View** – JSPs (not shown) that read `reviews` and pagination info.  
* **Controller** – `ReviewsAction` handling HTTP requests.  
* **Service** – `CatalogService` encapsulates business logic.  

This separation keeps the action focused on request/response orchestration.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `removeReview()` | Deletes a review specified by `reviewId`. | `reviewId` from form/parameter | `SUCCESS` or `AUTHORIZATIONERROR` | Deletes review, sets user message, logs errors. |
| `displayReviews()` | Fetches paginated list of reviews for the logged‑in customer. | Pagination params via `PageBaseAction` | `SUCCESS` | Populates `reviews`, pagination counters. |
| `getCriteria(int startIndex)` | Builds `SearchReviewCriteria` from request context. | `startIndex` | `SearchReviewCriteria` | No external side‑effects. |
| `reviewsQuery(SearchReviewCriteria criteria)` | Executes the service query and processes the response. | `criteria` | void | Sets `reviews`, updates pagination counters, localises entities. |
| `setCurrentEntity(String)` / `getCurrentEntity()` | Getter/Setter for `currentEntity` (unused). | – | – | – |
| `setReviews(Collection)` / `getReviews()` | Getter/Setter for review collection. | – | – | – |
| `setReviewId(long)` / `getReviewId()` | Getter/Setter for review ID. | – | – | – |

*Reusable utilities:* `SessionUtil`, `LanguageUtil`, `LocaleUtil` – used by multiple actions across the application.  

---

## 4. Dependencies  

| Library / Framework | Role | Status |
|---------------------|------|--------|
| **Apache Commons Configuration** | Reads `catalog.reviewslist.maxsize` at class load. | Third‑party |
| **Apache Log4j** | Logging (`Logger`). | Third‑party |
| **Struts‑2** (`PageBaseAction`) | MVC action base class providing pagination helpers. | Framework |
| **SalesManager Core** | Domain entities (`Review`, `SearchReviewCriteria`, `SearchReviewResponse`), service interfaces (`CatalogService`), utilities (`LanguageUtil`, `LocaleUtil`, `SessionUtil`). | Internal project modules |
| **Java Collections** | `Collection`, `Iterator`. | Standard |

The code assumes a servlet environment where `HttpServletRequest` is available via `super.getServletRequest()` and a session containing a `Customer` and `MerchantStore`.

---

## 5. Additional Notes  

### Strengths  

* **Clear separation** of concerns: the action delegates business logic to the service layer.  
* **Configurable page size** via external property file.  
* **Internationalisation support**: reviews are localised before being returned to the view.  

### Weaknesses & Edge Cases  

1. **Hard‑coded pagination logic** – uses a static `size` that never adapts to user preference.  
2. **No validation for `reviewId`** – negative or zero IDs will lead to service call failures.  
3. **Error handling** is generic; any exception returns a technical error message without specific context.  
4. **`currentEntity` field is unused** – appears to be a leftover from code duplication.  
5. **Thread‑safety** – the static `size` is read‑only after class loading, so safe; however, `reviews` is instance‑specific and fine.  
6. **Potential N+1** – in `reviewsQuery`, the code iterates over `reviews` but does nothing; perhaps intended for side‑effects like lazy loading.  
7. **Testing difficulty** – dependencies on `SessionUtil` and `ServiceFactory` require extensive mocking.  

### Suggested Improvements  

* **Parameterize page size** per user or via request parameter.  
* **Validate `reviewId`** before attempting deletion; return a friendly message if invalid.  
* **More granular exception handling** – catch specific service exceptions (e.g., `ReviewNotFoundException`).  
* **Remove unused fields** (`currentEntity`).  
* **Replace the empty iterator loop** with a meaningful operation or remove it.  
* **Use dependency injection** (e.g., Spring) for `CatalogService` to ease unit testing.  
* **Add Javadoc** and method comments for better maintainability.  
* **Internationalise error messages** rather than hard‑coding strings.  

Overall, the class is straightforward and functional but would benefit from a few refactors to improve clarity, robustness, and testability.

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
package com.salesmanager.customer.profile;

import java.util.Collection;
import java.util.Iterator;

import org.apache.commons.configuration.Configuration;
import org.apache.log4j.Logger;

import com.salesmanager.common.PageBaseAction;
import com.salesmanager.common.util.PropertiesHelper;
import com.salesmanager.core.entity.catalog.Review;
import com.salesmanager.core.entity.catalog.SearchReviewCriteria;
import com.salesmanager.core.entity.catalog.SearchReviewResponse;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.www.SessionUtil;

public class ReviewsAction extends PageBaseAction {

	private static int size = 20;
	private Logger log = Logger.getLogger(ReviewsAction.class);
	private static Configuration config = PropertiesHelper.getConfiguration();

	private long reviewId;// for deletion

	private String currentEntity;// not required for orders

	private Collection reviews;

	static {
		size = config.getInt("catalog.reviewslist.maxsize", 10);
	}

	public String removeReview() {

		try {

			CatalogService cservice = (CatalogService) ServiceFactory
					.getService(ServiceFactory.CatalogService);
			Review r = cservice.getProductReview(this.getReviewId());

			if (r == null) {
				log.warn("No review exist for review id " + this.getReviewId());
				return "AUTHORIZATIONERROR";
			}

			cservice.deleteProductReview(r);
			super.setMessage("messages.review.removed");

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public String displayReviews() {

		try {

			super.setSize(size);
			super.setPageStartNumber();

			SearchReviewCriteria crit = getCriteria(super.getPageStartIndex());
			reviewsQuery(crit);

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	private SearchReviewCriteria getCriteria(int startIndex) {

		MerchantStore store = SessionUtil.getMerchantStore(super
				.getServletRequest());
		Customer customer = SessionUtil.getCustomer(super.getServletRequest());

		SearchReviewCriteria criteria = new SearchReviewCriteria();

		criteria.setLanguageId(LanguageUtil.getLanguageNumberCode(super
				.getLocale().getLanguage()));
		criteria.setMerchantId(store.getMerchantId());
		criteria.setQuantity(size);
		criteria.setStartindex(startIndex);
		criteria.setCustomerId(customer.getCustomerId());

		return criteria;

	}

	private void reviewsQuery(SearchReviewCriteria criteria) throws Exception {

		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);

		SearchReviewResponse resp = cservice
				.searchProductReviewsByCustomer(criteria);
		if (resp != null) {
			reviews = resp.getReviews();

			LocaleUtil.setLocaleToEntityCollection(reviews, super.getLocale());

			if (reviews != null && reviews.size() > 0) {
				Iterator i = reviews.iterator();
				while (i.hasNext()) {
					Review r = (Review) i.next();

				}

			}

			super.setListingCount(resp.getCount());
			super.setRealCount(reviews.size());
			super.setPageElements();

			/*
			 * if(reviews==null || reviews.size()==0) {
			 * this.setFirstItem(firstItem); this.setLastItem(listingCount); }
			 * else { if(this.getPageStartIndex()==0) {
			 * this.setFirstItem(firstItem); } else {
			 * 
			 * this.setFirstItem(this.getPageStartIndex() * size +1);
			 * 
			 * }
			 * 
			 * if(listingCount<size) { this.setLastItem(listingCount); } else {
			 * this.setLastItem(size); } }
			 */
		}

	}

	public String getCurrentEntity() {
		return currentEntity;
	}

	public void setCurrentEntity(String currentEntity) {
		this.currentEntity = currentEntity;
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
