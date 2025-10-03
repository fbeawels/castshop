# ReviewDao.java

## Review

## 1. Summary

**Purpose**  
`ReviewDao` is a Spring‑managed DAO that handles persistence and retrieval of `Review` entities for the SalesManager e‑commerce platform. It uses Hibernate 3 for ORM and provides CRUD operations, search helpers, and aggregate statistics (average rating, count).

**Key components**

| Component | Role |
|-----------|------|
| `ReviewDao` | DAO implementation, extends `HibernateDaoSupport` |
| `IReviewDao` | Interface (not shown) that declares the DAO contract |
| `Review`, `SearchReviewCriteria`, `SearchReviewResponse`, `Counter` | Domain and DTO classes used by the DAO |
| Hibernate `SessionFactory` | Injected via Spring; used to obtain sessions |
| `Log` | SLF4J‑style logging via Apache Commons Logging |

**Design patterns & libraries**

* **DAO pattern** – Encapsulates persistence logic.
* **Repository pattern** – Spring `@Repository` marks it for component scanning.
* **Hibernate Criteria & Query APIs** – Provides type‑unsafe query construction.
* **Exception translation** – Runtime exceptions are logged and re‑thrown (no Spring `@Transactional` error handling shown).

## 2. Detailed Description

### Initialization
* The DAO is instantiated by Spring. The constructor receives a `SessionFactory` and forwards it to `HibernateDaoSupport.setSessionFactory`.  
* No explicit `@Transactional` annotations – transactions are likely controlled by higher‑level services or Spring’s AOP.

### Core workflow

| Operation | Flow |
|-----------|------|
| **persist / saveOrUpdate / delete** | Delegates to `HibernateTemplate` methods, wrapped in a try/catch that logs errors. |
| **findByCustomerId / findByProductId** | Builds an HQL query with `join fetch` to eager‑load descriptions and customer. Executes and returns the result list. |
| **searchByProductId / searchByCustomerId** | Combines a typed query (to fetch entities) with a `Criteria` count query. Pagination parameters (`upperLimit`, `lowerLimit`) are derived from `SearchReviewCriteria`. |
| **countAverageRatingByProduct** | Builds a `Criteria` that projects `avg(reviewRating)` and `rowCount()`. Parses the resulting tuple into a `Counter` DTO. |
| **findById** | Simple `HibernateTemplate.get` call. |

### Dependencies & Assumptions

* Relies on **Hibernate 3.x** (`org.hibernate.criterion`, `org.hibernate.query`).  
* Assumes that `Review` has associations `descriptions` (likely a `Map<Language, ReviewDescription>`) and `customer`.  
* Uses language‑specific fetch (`s.id.languageId=:lId`), expecting a composite key on `ReviewDescription`.  
* `SearchReviewCriteria` is expected to provide `getProductId()`, `getQuantity()`, `getStartindex()`, etc. No validation of these fields is performed in the DAO.  
* Pagination logic assumes that `getUpperLimit` and `getLowerLimit` return valid indices relative to the total count. If the criteria values are out of range, Hibernate will simply return an empty list.

### Architecture & Design Choices

* **Legacy API** – The DAO uses deprecated Hibernate 3 APIs (`HibernateTemplate`, `Criteria`). Modern Spring applications prefer JPA (`EntityManager`) or Hibernate 5/6 `Session`.  
* **Eager fetching via HQL** – `join fetch` is used to avoid N+1 selects when loading related entities. However, it can lead to Cartesian products when a review has multiple descriptions, causing duplicate root entities; the `DISTINCT_ROOT_ENTITY` transformer mitigates this but can still be expensive.  
* **Manual count queries** – Separate `Criteria` for count/average may be more efficient if a `SELECT COUNT` or `AVG` could be combined with the main query, but Hibernate cannot do that easily with HQL, so the approach is acceptable.  
* **Exception handling** – Logging and re‑throwing is minimalistic; it would be better to translate to Spring’s `DataAccessException` hierarchy.

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑Effects |
|--------|---------|------------|---------|--------------|
| `persist(Review)` | Persist a new review. | `Review` | void | Persisted in DB |
| `saveOrUpdate(Review)` | Insert or update a review. | `Review` | void | Updated in DB |
| `delete(Review)` | Delete a review. | `Review` | void | Removed from DB |
| `deleteAll(Collection<Review>)` | Bulk delete. | `Collection<Review>` | void | All removed |
| `findByCustomerId(long id, int languageId)` | Retrieve reviews for a customer in a specific language. | `customerId`, `languageId` | `Collection<Review>` | None |
| `findByProductId(long id, int languageId)` | Retrieve reviews for a product in a specific language. | `productId`, `languageId` | `Collection<Review>` | None |
| `findByProductId(long id)` | Retrieve reviews for a product (all languages). | `productId` | `Collection<Review>` | None |
| `searchByProductId(SearchReviewCriteria)` | Search reviews for a product with pagination. | `criteria` | `SearchReviewResponse` | None |
| `searchByCustomerId(SearchReviewCriteria)` | Search reviews for a customer with pagination. | `criteria` | `SearchReviewResponse` | None |
| `findById(long)` | Load a review by primary key. | `id` | `Review` | None |
| `countAverageRatingByProduct(long)` | Compute average rating and count for a product. | `productId` | `Counter` | None |

### Utility methods

* `getUpperLimit(int)` and `getLowerLimit()` are provided by `SearchReviewCriteria`; they compute pagination offsets.

## 4. Dependencies

| External | Type | Comments |
|----------|------|----------|
| `org.hibernate` | Third‑party | Hibernate 3.x (deprecated) |
| `org.springframework.orm.hibernate3` | Third‑party | Spring’s Hibernate 3 integration |
| `org.apache.commons.logging` | Third‑party | Logging abstraction |
| `com.salesmanager.core.entity.*` | Project | Domain entities |
| `com.salesmanager.core.entity.common.Counter` | Project | Simple DTO |

No platform‑specific APIs are used; the code is portable across JDK versions that support Hibernate 3 (Java 6+).

## 5. Additional Notes & Recommendations

### Edge Cases & Limitations

| Issue | Impact | Mitigation |
|-------|--------|------------|
| **Duplicate rows** – `join fetch` with many descriptions leads to cartesian product. | Excess memory, slow queries. | Use `Criteria` with `setResultTransformer(DISTINCT_ROOT_ENTITY)` (already used) or switch to a separate DTO query. |
| **Pagination bounds** – `SearchReviewCriteria` may return invalid indices. | Empty results or `IndexOutOfBounds`. | Validate `criteria` in service layer or DAO. |
| **Deprecated APIs** – Hibernate 3 and `HibernateTemplate` are no longer supported. | Security, performance, future maintenance risk. | Migrate to JPA (`EntityManager`) or Hibernate 5+ `Session`. |
| **No transaction management** – All methods assume an external transaction. | Potential data inconsistencies if used outside a transaction. | Annotate DAO methods or service layer with `@Transactional`. |
| **Exception handling** – Only logs and re‑throws `RuntimeException`. | No Spring `DataAccessException` hierarchy. | Wrap exceptions in `DataAccessException` or use `@Repository` which already performs translation. |
| **Hard‑coded entity names** (`"com.salesmanager.core.entity.catalog.Review"`). | If package changes, break. | Use class literal (`Review.class`). |
| **Type safety** – `List` raw types in `searchByProductId` and `searchByCustomerId`. | Compile‑time warnings, risk of `ClassCastException`. | Use generics (`List<Review>`). |

### Potential Enhancements

1. **Refactor to JPA / Hibernate 5+**  
   * Replace `HibernateDaoSupport` with a `@Repository` that injects `EntityManager`.  
   * Use JPA Criteria API or JPQL, avoiding legacy `Criteria` and `Query` usage.

2. **Add Spring’s `@Transactional`**  
   * Annotate DAO or service layer to ensure atomic operations and rollback on failure.

3. **Improve Pagination**  
   * Compute `total` and `pages` inside DAO, expose a `Page<Review>` DTO (e.g., Spring Data `Page`).

4. **Generic DAO base class**  
   * Reduce duplication across DAO implementations.

5. **Better error handling**  
   * Convert `RuntimeException` to `DataAccessException` (Spring automatically does this when `@Repository` is used).

6. **Use of `Optional<Review>`**  
   * For `findById`, return `Optional<Review>` to express presence/absence.

7. **Logging enhancements**  
   * Log query parameters, execution times, and pagination metrics for performance monitoring.

8. **Unit tests**  
   * Add integration tests with an in‑memory database (e.g., H2) to validate queries.

9. **Cache**  
   * Consider second‑level caching for read‑heavy operations (e.g., average rating queries).

10. **Method documentation**  
    * Add Javadoc comments to clarify method contracts and parameters.

### Summary

`ReviewDao` is a functional, albeit legacy, DAO layer that manages review entities using Hibernate 3. While it provides the necessary CRUD and search operations, it relies on outdated APIs, has limited type safety, and offers minimal transaction support. Migrating to modern JPA/Hibernate and incorporating transaction management and proper error handling would greatly improve maintainability, performance, and future‑proofing.

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
package com.salesmanager.core.service.catalog.impl.db.dao;

// Generated Nov 11, 2009 10:40:38 AM by Hibernate Tools 3.2.4.GA

import java.util.Collection;
import java.util.Iterator;
import java.util.List;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.hibernate.Criteria;
import org.hibernate.Query;
import org.hibernate.SessionFactory;
import org.hibernate.criterion.Projections;
import org.hibernate.criterion.Restrictions;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.orm.hibernate3.support.HibernateDaoSupport;
import org.springframework.stereotype.Repository;

import com.salesmanager.core.entity.catalog.Review;
import com.salesmanager.core.entity.catalog.SearchReviewCriteria;
import com.salesmanager.core.entity.catalog.SearchReviewResponse;
import com.salesmanager.core.entity.common.Counter;

/**
 * Home object for domain model class Reviews.
 * 
 * @see com.salesmanager.core.test.Reviews
 * @author Hibernate Tools
 */
@Repository
public class ReviewDao extends HibernateDaoSupport implements IReviewDao {

	private static final Log log = LogFactory.getLog(ReviewDao.class);

	@Autowired
	public ReviewDao(SessionFactory sessionFactory) {
		super.setSessionFactory(sessionFactory);
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDao#persist(com
	 * .salesmanager.core.entity.catalog.Review)
	 */
	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDescriptionDao#
	 * persist(com.salesmanager.core.entity.catalog.Review)
	 */
	public void persist(Review transientInstance) {
		try {
			super.getHibernateTemplate().persist(transientInstance);
		} catch (RuntimeException re) {
			log.error("persist failed", re);
			throw re;
		}
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDao#saveOrUpdate
	 * (com.salesmanager.core.entity.catalog.Review)
	 */
	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDescriptionDao#
	 * saveOrUpdate(com.salesmanager.core.entity.catalog.Review)
	 */
	public void saveOrUpdate(Review instance) {
		try {
			super.getHibernateTemplate().saveOrUpdate(instance);
		} catch (RuntimeException re) {
			log.error("attach failed", re);
			throw re;
		}
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDao#delete(com.
	 * salesmanager.core.entity.catalog.Review)
	 */
	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDescriptionDao#
	 * delete(com.salesmanager.core.entity.catalog.Review)
	 */
	public void delete(Review persistentInstance) {
		try {
			super.getHibernateTemplate().delete(persistentInstance);
		} catch (RuntimeException re) {
			log.error("delete failed", re);
			throw re;
		}
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDao#deleteAll(java
	 * .util.Collection)
	 */
	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDescriptionDao#
	 * deleteAll(java.util.Collection)
	 */
	public void deleteAll(Collection<Review> coll) {
		try {
			super.getHibernateTemplate().deleteAll(coll);
		} catch (RuntimeException re) {
			log.error("delete failed", re);
			throw re;
		}
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDao#findByCustomerId
	 * (long, int)
	 */
	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDescriptionDao#
	 * findByCustomerId(long, int)
	 */
	public Collection<Review> findByCustomerId(long id, int languageId) {
		try {
			Query q = super
					.getSession()
					.createQuery(
							"select r from Review r left join fetch r.descriptions s join fetch r.customer c where r.customerId=:cId and s.id.languageId=:lId order by r.reviewId desc")
					.setLong("cId", id).setInteger("lId", languageId)
					.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);
			;
			return q.list();
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDao#findByProductId
	 * (long, int)
	 */
	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDescriptionDao#
	 * findByProductId(long, int)
	 */
	public Collection<Review> findByProductId(long id, int languageId) {
		try {
			Query q = super
					.getSession()
					.createQuery(
							"select r from Review r left join fetch r.descriptions s join fetch r.customer c where r.productId=:pId and s.id.languageId=:lId")
					.setLong("pId", id).setInteger("lId", languageId)
					.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);
			;
			return q.list();
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Collection<Review> findByProductId(long id) {
		try {
			Query q = super
					.getSession()
					.createQuery(
							"select r from Review r left join fetch r.descriptions s join fetch r.customer c where r.productId=:pId order by r.reviewId desc")
					.setLong("pId", id);
			return q.list();
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public SearchReviewResponse searchByProductId(SearchReviewCriteria criteria) {
		try {
			// Query q =
			// super.getSession().createQuery("select r from Review r left join fetch r.descriptions s join fetch r.customer c where r.productId=:pId and s.id.languageId=:lId order by r.reviewId desc")
			Query q = super
					.getSession()
					.createQuery(
							"select r from Review r left join fetch r.descriptions s join fetch r.customer c where r.productId=:pId order by r.reviewId desc")
					.setLong("pId", criteria.getProductId())
					// .setInteger("lId", criteria.getLanguageId());
					.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);

			Criteria c = super.getSession().createCriteria(Review.class).add(
					Restrictions.eq("productId", criteria.getProductId()))
					.addOrder(org.hibernate.criterion.Order.desc("reviewId"));

			c.setProjection(Projections.rowCount());
			Integer count = (Integer) c.uniqueResult();

			c.setProjection(null);

			int max = criteria.getQuantity();
			/*
			 * if(count<criteria.getQuantity()) {
			 * if(criteria.getStartindex()==1) { max = count; } else { max =
			 * count - criteria.getStartindex(); } }
			 */

			List list = null;
			if (count > 0) {
				q.setMaxResults(criteria.getUpperLimit(count));
				q.setFirstResult(criteria.getLowerLimit());
			}

			list = q.list();

			SearchReviewResponse response = new SearchReviewResponse();
			response.setCount(count);
			response.setReviews(list);

			return response;

		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public SearchReviewResponse searchByCustomerId(SearchReviewCriteria criteria) {
		try {
			// Query q =
			// super.getSession().createQuery("select r from Review r left join fetch r.descriptions s join fetch r.customer c where r.customerId=:cId and s.id.languageId=:lId order by r.reviewId desc")
			Query q = super
					.getSession()
					.createQuery(
							"select r from Review r join fetch r.customer c where r.customerId=:cId order by r.reviewId desc")
					.setLong("cId", criteria.getCustomerId());
			// .setInteger("lId", criteria.getLanguageId());

			Criteria c = super.getSession().createCriteria(Review.class).add(
					Restrictions.eq("customerId", criteria.getCustomerId()))
					.addOrder(org.hibernate.criterion.Order.desc("reviewId"));

			c.setProjection(Projections.rowCount());
			Integer count = (Integer) c.uniqueResult();

			c.setProjection(null);

			List list = null;
			if (count > 0) {
				q.setMaxResults(criteria.getUpperLimit(count));
				q.setFirstResult(criteria.getLowerLimit());
			}

			list = q.list();

			SearchReviewResponse response = new SearchReviewResponse();
			response.setCount(count);
			response.setReviews(list);

			return response;

		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDao#findById(long)
	 */
	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.dao.IReviewDescriptionDao#
	 * findById(long)
	 */
	public Review findById(long id) {
		try {
			Review instance = (Review) super.getHibernateTemplate().get(
					"com.salesmanager.core.entity.catalog.Review", id);
			return instance;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Counter countAverageRatingByProduct(long productId) {
		try {
			Criteria c = super.getSession().createCriteria(Review.class).add(
					Restrictions.eq("productId", productId));

			c.setProjection(Projections.projectionList().add(
					Projections.avg("reviewRating"))
					.add(Projections.rowCount()));

			List resp = c.list();

			Counter counter = new Counter();
			if (resp != null && resp.size() > 0) {
				Iterator i = resp.iterator();
				while (i.hasNext()) {
					Object[] o = (Object[]) i.next();
					counter.setAverage(((Double) o[0]));
					counter.setCount((Integer) o[1]);
				}
			}

			return counter;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

}



```
