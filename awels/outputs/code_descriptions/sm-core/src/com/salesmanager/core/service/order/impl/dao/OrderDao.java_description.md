# OrderDao.java

## Review

## 1. Summary  

The **`OrderDao`** class is a Spring‑annotated DAO that performs CRUD and search operations on the `Order` entity using **Hibernate 3** (via `HibernateDaoSupport`).  
Key responsibilities:

| Component | Purpose |
|-----------|---------|
| `persist()` / `saveOrUpdate()` / `delete()` / `merge()` | Basic persistence helpers that wrap the corresponding HibernateTemplate methods. |
| `createRawOrder(long)` | Inserts an order row directly with a raw SQL statement (used for “raw” orders). |
| `findById(long)` | Retrieve a single order by its primary key. |
| `findOrdersByCustomer(long)` / `findOrdersByMerchant(int)` | Simple queries that return orders filtered by customer or merchant. |
| `findInvoicesByCustomer…` | Returns invoices (orders with `channel == INVOICE_CHANNEL`) optionally filtered by date. |
| `searchInvoice(...)`, `searchOrder(...)`, `searchOrderByCustomer(...)` | Rich search APIs that accept a `SearchOrdersCriteria` DTO and return a `SearchOrderResponse` (count + list). They build a **Criteria** for the row count and an **HQL** query for the data. |

The DAO relies on a mixture of Hibernate **Criteria API**, **HQL**, and a raw **SQL** query.  
It is wired with Spring (`@Repository`, constructor injection of `SessionFactory`) and uses **Apache Commons Lang** (`StringUtils`, `DateUtils`) and **Apache Commons Logging**.

---

## 2. Detailed Description  

### Execution Flow

1. **Construction** – Spring injects a `SessionFactory` and calls `setSessionFactory`.  
2. **CRUD Operations** – All use `getHibernateTemplate()` which opens a session (via the configured transaction manager) and executes the operation.  
3. **Search Operations** –  
   * Build a `Criteria` instance on `Order.class` and attach restrictions (merchant, channel, status, etc.).  
   * Simultaneously build an HQL string with parameters for pagination.  
   * Execute the count query via `criteria.setProjection(rowCount)` to obtain the total number of matching rows.  
   * Execute the data query (`Query c`) with the same parameters, applying `setMaxResults`/`setFirstResult` for pagination.  
   * Wrap the result list and count in a `SearchOrderResponse`.  
4. **Cleanup** – Sessions are automatically closed by the `HibernateTemplate` / Spring transaction boundary; no explicit cleanup code is required.

### Assumptions & Constraints

| Assumption | Implication |
|------------|-------------|
| Transactions are managed externally (Spring, JTA). | DAO methods must be called within a transactional context. |
| `SearchOrdersCriteria` always contains valid IDs; null checks are minimal. | Potential `NullPointerException` if called with a null criteria. |
| The database schema matches the `Order` entity mapping exactly. | No runtime mapping errors. |
| `OrderConstants` values are static integers (e.g., `INVOICE_CHANNEL`). | Hard‑coded constants reduce flexibility. |

### Design Choices

* **Hibernate 3 Criteria** – Chosen for its programmatic filtering, but the API is dated; `Criteria` is deprecated in Hibernate 5+.  
* **Mix of Criteria + HQL** – The DAO uses a `Criteria` object for counting rows but an HQL string for fetching data. This is unnecessary and complicates maintenance.  
* **Raw SQL in `createRawOrder`** – The method bypasses Hibernate entirely and concatenates the ID into the SQL string, which is fragile and vulnerable to injection.  
* **Logging** – Uses `LogFactory.getLog` and logs at `error` level only; no debug or info logs for normal operation.

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `persist(Order)` | Persist a transient `Order`. | `transientInstance` | void | Calls `HibernateTemplate.persist`. |
| `saveOrUpdate(Order)` | Persist or update an `Order`. | `instance` | void | Calls `HibernateTemplate.saveOrUpdate`. |
| `delete(Order)` | Delete a persistent `Order`. | `persistentInstance` | void | Calls `HibernateTemplate.delete`. |
| `merge(Order)` | Merge a detached `Order`. | `detachedInstance` | `Order` (merged) | Calls `HibernateTemplate.merge`. |
| `createRawOrder(long)` | Insert a row via raw SQL, then load the entity. | `orderId` | `Order` | Executes `INSERT` then `findById`. |
| `findById(long)` | Load an `Order` by primary key. | `id` | `Order` | Calls `HibernateTemplate.get`. |
| `findOrdersByCustomer(long)` | Find all orders for a customer. | `customerId` | `List<Order>` | Criteria query. |
| `findOrdersByMerchant(int)` | Find all orders for a merchant. | `merchantId` | `List<Order>` | Criteria query. |
| `findInvoicesByCustomer(long)` | Find all invoices for a customer. | `customerId` | `Collection<Order>` | Criteria query with `channel = INVOICE_CHANNEL`. |
| `findInvoicesByCustomerAndStartDate(long, Date)` | Same as above, but start date ≥ `startDate`. | `customerId`, `startDate` | `Collection<Order>` | Criteria query with date filter. |
| `searchInvoice(SearchOrdersCriteria)` | Paginated invoice search (by merchant). | `searchCriteria` | `SearchOrderResponse` | Builds Criteria for count and HQL for data. |
| `searchOrder(SearchOrdersCriteria)` | Paginated order search (by merchant, online channel). | `searchCriteria` | `SearchOrderResponse` | Same pattern as `searchInvoice`. |
| `searchOrderByCustomer(SearchOrdersCriteria)` | Paginated order search (by customer, merchant). | `searchCriteria` | `SearchOrderResponse` | Uses a simple HQL query. |

**Reusable / Utility** – No dedicated helper methods; most logic is duplicated across search methods (criteria construction, query building).

---

## 4. Dependencies  

| Library | Version (implied) | Role |
|---------|-------------------|------|
| Spring Framework | 3.x (based on `HibernateDaoSupport`) | DAO wiring, transaction management |
| Hibernate | 3.2.0 (tools 3.2.0.beta8) | ORM, Criteria, Session |
| Apache Commons Lang | 2.x | String and date utilities |
| Apache Commons Logging | 1.x | Logging |
| Java SE | 1.5+ (Date, Collection, etc.) | Core language |

All dependencies are **third‑party**. The code is tightly coupled to Hibernate 3 and Spring 3, which are both **out of date** in contemporary Java ecosystems.

---

## 5. Additional Notes  

### 5.1 Edge Cases & Potential Bugs  

| Issue | Description | Impact |
|-------|-------------|--------|
| **Null `searchCriteria`** | Methods access `searchCriteria.getMerchantId()` before checking for null. | NPE if called with null. |
| **Parameter Binding** | HQL strings use `"and o.customerName like %:cName%"` – the `%` is concatenated *outside* the parameter placeholder, leading to an invalid query. | Query fails or returns no results. |
| **Date Handling** | When `sdate` or `edate` is null, the code still sets parameters (`c.setDate`) but uses `DateUtils.addDays` incorrectly (adds +1 or -1). | Wrong date range, potential off‑by‑one errors. |
| **SQL Injection** | `createRawOrder` concatenates `orderId` directly into SQL. | Injection risk (though `orderId` is numeric). |
| **Redundant Criteria** | Count and list queries use two different APIs (Criteria + HQL) with duplicated restriction logic. | Hard to maintain, inconsistent filtering. |
| **Duplicate Results** | Use of `Criteria.DISTINCT_ROOT_ENTITY` without proper fetching strategy may still return duplicates if eager associations exist. | Extra data transfer, potential N+1 problems. |
| **Pagination Bounds** | `searchOrder` uses `searchCriteria.getUpperLimit(count)` but `getUpperLimit` may return `max` even when `count` is smaller; no guard for negative values. | Could request more rows than exist. |
| **Hard‑coded Constants** | `OrderConstants.INVOICE_CHANNEL`, `ONLINE_CHANNEL`, etc. are used directly. | Hard to change channel semantics. |

### 5.2 Suggested Enhancements  

| Category | Recommendation |
|----------|----------------|
| **Modernize ORM** | Migrate to **Hibernate 5+** or **JPA 2.2** (`EntityManager`). Replace `HibernateDaoSupport` with Spring Data JPA repositories or a lightweight DAO layer. |
| **Unified Querying** | Use a **single HQL or Criteria** for both count and data queries. Hibernate allows `setProjection` and `setResultTransformer` on the same query; or use `ScrollableResults` for counting. |
| **Named Queries / Criteria API** | Extract common restriction logic into reusable methods (`addBaseRestrictions(Criteria c, SearchOrdersCriteria sc)`). |
| **Parameter Binding** | Always use `:param` placeholders and set parameters via `Query.setParameter`. Avoid concatenating `%` inside the query string. |
| **Date Range** | Normalize dates to midnight and use inclusive boundaries. Provide utility method `applyDateRange(Criteria c, SearchOrdersCriteria sc)`. |
| **Validation** | Add defensive checks for null `searchCriteria`, null dates, or negative IDs. |
| **Logging** | Add debug logs for entering/exiting methods and key parameters. |
| **Transactions** | Annotate DAO methods with `@Transactional(readOnly = true)` where appropriate. |
| **Exception Handling** | Replace generic `RuntimeException` catches with specific data access exceptions (`DataAccessException`). |
| **Code Duplication** | Refactor search methods to share a private helper that builds the base query and handles pagination. |
| **Testing** | Provide unit tests for each search method with mock sessions or an in‑memory database. |
| **Security** | Replace raw SQL in `createRawOrder` with a parameterized `SQLQuery` or use `Session.createSQLQuery` with named parameters. |

### 5.3 Future Extensions  

* **Full Text Search** – Integrate Hibernate Search / ElasticSearch for customer name/order details.  
* **Audit Logging** – Capture create/update/delete events for compliance.  
* **Soft Delete** – Add a `deleted` flag instead of physical removal.  
* **Multi‑tenant Support** – Include tenant ID in all queries.  
* **Bulk Operations** – Implement batch update/delete for large data sets.  

---

### Final Verdict  

The `OrderDao` implements the necessary data access logic but is **fragile**, **hard to maintain**, and **outdated**. The mix of Criteria, HQL, and raw SQL, along with duplicated logic and unsafe string concatenation, poses correctness and security risks. A refactor toward a modern, single‑API approach (JPA + Criteria or JPQL) with proper parameter binding, validation, and transaction management would greatly improve reliability, readability, and future‑proofing.

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
package com.salesmanager.core.service.order.impl.dao;

// Generated Oct 1, 2008 11:18:03 AM by Hibernate Tools 3.2.0.beta8

import java.util.Collection;
import java.util.Date;
import java.util.List;

import org.apache.commons.lang.StringUtils;
import org.apache.commons.lang.time.DateUtils;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.hibernate.Criteria;
import org.hibernate.Query;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.criterion.Projections;
import org.hibernate.criterion.Restrictions;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.orm.hibernate3.support.HibernateDaoSupport;
import org.springframework.stereotype.Repository;

import com.salesmanager.core.constants.OrderConstants;
import com.salesmanager.core.entity.orders.Order;
import com.salesmanager.core.entity.orders.SearchOrderResponse;
import com.salesmanager.core.entity.orders.SearchOrdersCriteria;

/**
 * Home object for domain model class Orders.
 * 
 * @see com.salesmanager.core.test.Orders
 * @author Hibernate Tools
 */
@Repository
public class OrderDao extends HibernateDaoSupport implements IOrderDao {

	private static final Log log = LogFactory.getLog(OrderDao.class);

	@Autowired
	public OrderDao(SessionFactory sessionFactory) {
		super.setSessionFactory(sessionFactory);
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.order.impl.IOrderDao#persist(com.salesmanager
	 * .core.entity.orders.Order)
	 */
	public void persist(Order transientInstance) {
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
	 * @seecom.salesmanager.core.service.order.impl.IOrderDao#saveOrUpdate(com.
	 * salesmanager.core.entity.orders.Order)
	 */
	public void saveOrUpdate(Order instance) {
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
	 * com.salesmanager.core.service.order.impl.IOrderDao#delete(com.salesmanager
	 * .core.entity.orders.Order)
	 */
	public void delete(Order persistentInstance) {
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
	 * com.salesmanager.core.service.order.impl.IOrderDao#merge(com.salesmanager
	 * .core.entity.orders.Order)
	 */
	public Order merge(Order detachedInstance) {
		try {
			Order result = (Order) super.getHibernateTemplate().merge(
					detachedInstance);
			return result;
		} catch (RuntimeException re) {
			log.error("merge failed", re);
			throw re;
		}
	}

	public Order createRawOrder(long orderId) {
		try {
			Session session = getHibernateTemplate().getSessionFactory()
					.getCurrentSession();

			session.createSQLQuery(
					"INSERT INTO orders(orders_id) values (" + orderId + ")")
					.executeUpdate();

			Order result = findById(orderId);
			return result;
		} catch (Exception re) {
			log.error("merge failed", re);
			throw new RuntimeException(re);
		}
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see com.salesmanager.core.service.order.impl.IOrderDao#findById(int)
	 */
	public Order findById(long id) {
		try {
			Order instance = (Order) super.getHibernateTemplate().get(
					"com.salesmanager.core.entity.orders.Order", id);

			return instance;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	@SuppressWarnings("unchecked")
	public List<Order> findOrdersByCustomer(long customerId) {

		Criteria criteria = super.getSession().createCriteria(Order.class).add(
				Restrictions.eq("customerId", customerId)).addOrder(
				org.hibernate.criterion.Order.desc("orderId"))
				.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);

		return criteria.list();

	}

	public List<Order> findOrdersByMerchant(int merchantId) {

		Criteria criteria = super.getSession().createCriteria(Order.class).add(
				Restrictions.eq("merchantId", merchantId)).addOrder(
				org.hibernate.criterion.Order.desc("orderId"))
				.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);

		return criteria.list();
	}

	@SuppressWarnings("unchecked")
	public Collection<Order> findInvoicesByCustomer(long customerId) {
		Criteria criteria = super.getSession().createCriteria(Order.class).add(
				Restrictions.eq("customerId", customerId)).add(
				Restrictions.eq("channel", OrderConstants.INVOICE_CHANNEL))
				.addOrder(org.hibernate.criterion.Order.desc("orderId"));

		return criteria.list();

	}

	@SuppressWarnings("unchecked")
	public Collection<Order> findInvoicesByCustomerAndStartDate(
			long customerId, Date startDate) {
		Criteria criteria = super.getSession().createCriteria(Order.class).add(
				Restrictions.eq("customerId", customerId)).add(
				Restrictions.eq("channel", OrderConstants.INVOICE_CHANNEL))
				.add(Restrictions.ge("datePurchased", startDate)).addOrder(
						org.hibernate.criterion.Order.desc("orderId"));

		return criteria.list();

	}

	public SearchOrderResponse searchInvoice(SearchOrdersCriteria searchCriteria) {

		Criteria criteria = super
				.getSession()
				.createCriteria(Order.class)
				.add(
						Restrictions.eq("merchantId", searchCriteria
								.getMerchantId()))
				.add(Restrictions.eq("channel", OrderConstants.INVOICE_CHANNEL))
				.add(
						Restrictions.eq("orderStatus",
								OrderConstants.STATUSINVOICED)).addOrder(
						org.hibernate.criterion.Order.desc("orderId"))
				.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);

		StringBuffer q = new StringBuffer();

		q.append(" select o from Order o where o.merchantId=:mId");
		q.append(" and channel=:channel and orderStatus=:status");

		if (searchCriteria != null) {

			if (!StringUtils.isBlank(searchCriteria.getCustomerName())) {
				q.append(" and o.customerName like %:cName%");
			}

			if (searchCriteria.getOrderId() != -1) {
				q.append(" and o.orderId=:oId");
			}

			if (searchCriteria.getEdate() != null
					|| searchCriteria.getSdate() != null) {
				if (searchCriteria.getSdate() != null) {
					q.append(" and o.datePurchased > :sDate");
				} else {
					q.append(" and o.datePurchased > :sDate");
				}
				if (searchCriteria.getEdate() != null) {
					q.append(" and o.datePurchased < :eDate");
				} else {
					q.append(" and o.datePurchased < :eDate");
				}
			}
		}
		q.append(" order by o.orderId desc");

		Query c = super.getSession().createQuery(q.toString());
		c.setInteger("channel", OrderConstants.INVOICE_CHANNEL);
		c.setInteger("status", OrderConstants.STATUSINVOICED);
		c.setInteger("mId", searchCriteria.getMerchantId());

		if (searchCriteria != null) {

			if (!StringUtils.isBlank(searchCriteria.getCustomerName())) {
				criteria.add(Restrictions.like("customerName", "%"
						+ searchCriteria.getCustomerName() + "%"));
				c.setString("cName", "%" + searchCriteria.getCustomerName()
						+ "%");
			}

			if (searchCriteria.getOrderId() != -1) {
				criteria.add(Restrictions.eq("orderId", searchCriteria
						.getOrderId()));
				c.setLong("oId", searchCriteria.getOrderId());
			}

			if (searchCriteria.getEdate() != null
					|| searchCriteria.getSdate() != null) {
				if (searchCriteria.getSdate() != null) {
					criteria.add(Restrictions.ge("datePurchased",
							searchCriteria.getSdate()));
					c.setDate("sDate", searchCriteria.getSdate());
				} else {
					criteria.add(Restrictions.ge("datePurchased", DateUtils
							.addDays(new Date(), -1)));
					c.setDate("sDate", DateUtils.addDays(new Date(), -1));
				}
				if (searchCriteria.getEdate() != null) {
					criteria.add(Restrictions.le("datePurchased",
							searchCriteria.getEdate()));
					c.setDate("eDate", searchCriteria.getEdate());
				} else {
					criteria.add(Restrictions.ge("datePurchased", DateUtils
							.addDays(new Date(), +1)));
					c.setDate("eDate", DateUtils.addDays(new Date(), +1));
				}
			}
		}

		criteria.setProjection(Projections.rowCount());
		Integer count = (Integer) criteria.uniqueResult();

		criteria.setProjection(null);

		int max = searchCriteria.getQuantity();

		List list = null;
		if (max != -1 && count > 0) {
			list = c.setMaxResults(searchCriteria.getUpperLimit(count))
					.setFirstResult(searchCriteria.getLowerLimit()).list();
		} else {
			list = c.list();
		}

		SearchOrderResponse response = new SearchOrderResponse();
		response.setCount(count);
		response.setOrders(list);

		return response;

	}

	public SearchOrderResponse searchOrder(SearchOrdersCriteria searchCriteria) {

		Criteria criteria = super.getSession().createCriteria(Order.class).add(
				Restrictions.eq("merchantId", searchCriteria.getMerchantId()))
				.add(Restrictions.eq("channel", OrderConstants.ONLINE_CHANNEL))
				.addOrder(org.hibernate.criterion.Order.desc("orderId"))
				.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);

		StringBuffer q = new StringBuffer();

		q.append(" select o from Order o where o.merchantId=:mId");
		q.append(" and channel=:channel");

		if (searchCriteria != null) {

			if (!StringUtils.isBlank(searchCriteria.getCustomerName())) {
				q.append(" and o.customerName like :cName");
			}

			if (searchCriteria.getOrderId() != -1) {
				q.append(" and o.orderId= :oId");
			}

			if (searchCriteria.getEdate() != null
					|| searchCriteria.getSdate() != null) {
				if (searchCriteria.getSdate() != null) {
					q.append(" and o.datePurchased > :sDate");
				} else {
					q.append(" and o.datePurchased > :sDate");
				}
				if (searchCriteria.getEdate() != null) {
					q.append(" and o.datePurchased < :eDate");
				} else {
					q.append(" and o.datePurchased < :eDate");
				}
			}
		}
		q.append(" order by o.orderId desc");

		Query c = super.getSession().createQuery(q.toString());
		c.setInteger("channel", OrderConstants.ONLINE_CHANNEL);
		c.setInteger("mId", searchCriteria.getMerchantId());

		if (searchCriteria != null) {

			if (!StringUtils.isBlank(searchCriteria.getCustomerName())) {
				criteria.add(Restrictions.like("customerName", "%"
						+ searchCriteria.getCustomerName() + "%"));
				c.setString("cName", "%" + searchCriteria.getCustomerName()
						+ "%");
			}

			if (searchCriteria.getOrderId() != -1) {
				criteria.add(Restrictions.eq("orderId", searchCriteria
						.getOrderId()));
				c.setLong("oId", searchCriteria.getOrderId());
			}

			if (searchCriteria.getEdate() != null
					|| searchCriteria.getSdate() != null) {
				if (searchCriteria.getSdate() != null) {
					criteria.add(Restrictions.ge("datePurchased",
							searchCriteria.getSdate()));
					c.setDate("sDate", searchCriteria.getSdate());
				} else {
					criteria.add(Restrictions.ge("datePurchased", DateUtils
							.addDays(new Date(), -1)));
					c.setDate("sDate", DateUtils.addDays(new Date(), -1));
				}
				if (searchCriteria.getEdate() != null) {
					criteria.add(Restrictions.le("datePurchased",
							searchCriteria.getEdate()));
					c.setDate("eDate", searchCriteria.getEdate());
				} else {
					criteria.add(Restrictions.ge("datePurchased", DateUtils
							.addDays(new Date(), +1)));
					c.setDate("eDate", DateUtils.addDays(new Date(), +1));
				}
			}
		}

		criteria.setProjection(Projections.rowCount());
		Integer count = (Integer) criteria.uniqueResult();

		criteria.setProjection(null);

		int max = searchCriteria.getQuantity();

		List list = null;
		if (max != -1 && count > 0) {
			c.setMaxResults(searchCriteria.getUpperLimit(count));
			c.setFirstResult(searchCriteria.getLowerLimit());
			list = c.list();
		} else {
			list = c.list();
		}

		SearchOrderResponse response = new SearchOrderResponse();
		response.setCount(count);
		response.setOrders(list);

		return response;

	}

	public SearchOrderResponse searchOrderByCustomer(
			SearchOrdersCriteria searchCriteria) {

		Criteria criteria = super.getSession().createCriteria(Order.class).add(
				Restrictions.eq("customerId", searchCriteria.getCustomerId()))
				.add(
						Restrictions.eq("merchantId", searchCriteria
								.getMerchantId())).add(
						Restrictions.eq("channel",
								OrderConstants.ONLINE_CHANNEL)).addOrder(
						org.hibernate.criterion.Order.desc("orderId"));



		StringBuffer q = new StringBuffer();

		q.append(" select o from Order o where o.merchantId=:mId");
		q.append(" and channel=:channel");

		q.append(" and o.customerId = :cId");
		q.append(" and o.merchantId = :mId");
		q.append(" order by o.orderId desc");

		Query query = super.getSession().createQuery(q.toString());
		query.setInteger("channel", OrderConstants.ONLINE_CHANNEL);
		query.setInteger("mId", searchCriteria.getMerchantId());
		query.setLong("cId", searchCriteria.getCustomerId());
		query.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);

		criteria.setProjection(Projections.rowCount());
		Integer count = (Integer) criteria.uniqueResult();

		criteria.setProjection(null);

		int max = searchCriteria.getQuantity();

		List list = null;
		if (max != -1 && count > 0) {
			query.setMaxResults(searchCriteria.getUpperLimit(count));
			query.setFirstResult(searchCriteria.getLowerLimit());
			list = query.list();
		} else {
			list = query.list();
		}

		SearchOrderResponse response = new SearchOrderResponse();
		response.setCount(count);
		response.setOrders(list);

		return response;

	}

}



```
