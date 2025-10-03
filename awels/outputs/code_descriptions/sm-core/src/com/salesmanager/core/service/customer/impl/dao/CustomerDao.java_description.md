# CustomerDao.java

## Review

## 1. Summary  

**Purpose**  
`CustomerDao` is a Spring‑managed Hibernate DAO that encapsulates all persistence operations for the `Customer` entity and its related shopping‑cart objects (`CustomerBasket` & `CustomerBasketAttribute`). It implements the `ICustomerDao` interface and is annotated with `@Repository`, enabling Spring’s exception translation and component scanning.

**Key Components**  

| Class | Responsibility |
|-------|----------------|
| `CustomerDao` | CRUD, search, and helper methods for `Customer` and cart entities |
| `Customer` | JPA entity representing a customer record |
| `CustomerBasket` / `CustomerBasketAttribute` | Entities for the shopping‑cart model |
| `SearchCustomerCriteria` | DTO carrying filtering and pagination parameters |
| `SearchCustomerResponse` | DTO holding a list of customers + count |

**Frameworks / Libraries**  

- **Hibernate 3** (`org.hibernate.*`, `org.hibernate.criterion.*`) – low‑level ORM  
- **Spring ORM** (`org.springframework.orm.hibernate3.support.HibernateDaoSupport`) – DAO base class  
- **Spring Framework** (`@Repository`, `@Autowired`) – dependency injection & transaction management  
- **Apache Commons** (`StringUtils`, `Log`) – string handling & logging  

The class relies on the Spring `SessionFactory` injected via the constructor.

---

## 2. Detailed Description  

### Initialization  

- The DAO is instantiated by Spring, with the `SessionFactory` automatically wired through the constructor.  
- The `SessionFactory` is passed to the parent `HibernateDaoSupport` via `super.setSessionFactory(sessionFactory)`.  

### Runtime Behavior  

All methods obtain a Hibernate `Session` from the `HibernateTemplate` (inherited from `HibernateDaoSupport`) and perform operations such as `persist`, `saveOrUpdate`, `merge`, `delete`, and `createCriteria`.  

Typical execution path:

1. **Persistence Methods** (`saveShoppingCart`, `saveShoppingCartAttributes`, `persist`, `saveOrUptade`, `merge`, `delete`, `deleteAll`) – straightforward `HibernateTemplate` calls with a try/catch that logs and re‑throws runtime exceptions.  
2. **Read Methods** (`findById`, `findByMerchantId`, `findByUserNameAndPassword`, etc.) – use either HQL, Criteria API, or native SQL to fetch entities or scalar values.  
3. **Search** (`findCustomers`) – builds two separate `Criteria` objects: one for counting rows and another for retrieving the actual results, applying filters from `SearchCustomerCriteria`. Pagination is handled via `setMaxResults` / `setFirstResult`.  

### Cleanup  

There is no explicit resource cleanup; the DAO relies on Spring to manage the Hibernate `SessionFactory` and `Session` lifecycle. Transactions are expected to be handled at a higher layer (e.g., service layer annotated with `@Transactional`).  

### Assumptions & Constraints  

- The DAO assumes a single database connection per DAO instance (the default for Spring’s `HibernateDaoSupport`).  
- `SearchCustomerCriteria` must provide a non‑null `merchantId` and pagination details.  
- The code expects the database schema to match the Hibernate mappings exactly (e.g., column names like `customers_company`).  
- The DAO uses Hibernate 3, which is deprecated; newer projects should migrate to Hibernate 5+ or JPA.

---

## 3. Functions / Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects / Notes |
|--------|---------|--------|---------|----------------------|
| `saveShoppingCart(CustomerBasket)` | Persist a new basket | `transientInstance` | void | Calls `persist` on template; logs failures |
| `saveShoppingCartAttributes(CustomerBasketAttribute)` | Persist basket attribute | `transientInstance` | void | Same pattern as above |
| `persist(Customer)` | Persist a generic customer | `transientInstance` | void | `HibernateTemplate.persist` |
| `saveOrUptade(Customer)` | Save or update a customer | `instance` | void | `saveOrUpdate` |
| `merge(Customer)` | Merge a detached customer | `detachedInstance` | `Customer` | Returns merged instance |
| `delete(Customer)` | Delete a customer | `persistentInstance` | void | `delete` |
| `deleteAll(Collection<Customer>)` | Batch delete | `customers` | void | `deleteAll` |
| `findById(long)` | Retrieve by primary key | `id` | `Customer` | `get` |
| `findByMerchantId(int)` | All customers for a merchant | `merchantId` | `Collection<Customer>` | Criteria query |
| `findByUserNameAndPassword(String,String)` | Authenticate by username/password | `userName`,`password` | `Customer` | HQL |
| `findByUserNameAndPasswordByMerchantId(String,String,int)` | Auth with merchant scope | `userName`,`password`,`merchantId` | `Customer` | Criteria |
| `findByCompanyName(String,int)` | Find customers by company | `companyName`,`merchantId` | `Collection<Customer>` | Criteria |
| `findCustomersHavingCompany(int)` | Find customers with non‑null company | `merchantId` | `Collection<Customer>` | Criteria |
| `findUniqueCompanyName(int)` | Distinct list of company names | `merchantId` | `List<String>` | Native SQL |
| `findCustomerbyEmail(String)` | Lookup by email | `email` | `Customer` | Criteria |
| `findCustomers(SearchCustomerCriteria)` | Search with filters & pagination | `searchCriteria` | `SearchCustomerResponse` | Builds count & result criteria |
| `findCustomerbyUserName(String,int)` | Lookup by username & merchant | `userName`,`merchantId` | `Customer` | Criteria |

**Utility / Reusable Methods**  
- No dedicated helper methods; all logic is inlined in each DAO method.  

---

## 4. Dependencies  

| Library / Framework | Usage | Status |
|---------------------|-------|--------|
| `org.hibernate` (3.x) | Session, Criteria, HQL, native SQL | **Third‑party, deprecated** |
| `org.springframework.orm.hibernate3.support.HibernateDaoSupport` | Base DAO class | Third‑party |
| `org.springframework.stereotype.Repository` | Spring bean annotation | Third‑party |
| `org.springframework.beans.factory.annotation.Autowired` | Constructor injection | Third‑party |
| `org.apache.commons.lang.StringUtils` | Null/blank checks | Third‑party |
| `org.apache.commons.logging.Log` | Logging | Third‑party |
| JPA Entities (`Customer`, `CustomerBasket`, etc.) | Persistence mapping | Project code |

**Platform / Assumptions**  
- RDBMS must support the expected schema (`customers_company`, etc.).  
- Spring transaction management is assumed externally (e.g., service layer).  

---

## 5. Additional Notes  

### Code Quality & Maintainability  

1. **Deprecated Hibernate 3** – The project uses Hibernate 3 APIs (`HibernateDaoSupport`, `Criteria`, `Query`). Modern applications should migrate to Hibernate 5+ or the JPA Criteria API to avoid future compatibility issues.  
2. **Error Handling** – Each method wraps all operations in a generic `try/catch` that logs and re‑throws the exception. While this ensures errors are logged, it is redundant because Spring already translates Hibernate exceptions into `DataAccessException`. Removing the manual try/catch would reduce boilerplate and leverage Spring’s exception translation.  
3. **Naming Consistency** – Method `saveOrUptade` is misspelled; it should be `saveOrUpdate`.  
4. **SQL Injection Risk** – `findByUserNameAndPassword` uses HQL with named parameters (`:cId`, `:pId`) – safe. However, the method `findByMerchantId(int)` etc. all use Criteria which is also safe.  
5. **Performance** – The `findCustomers` method builds two separate `Criteria` objects: one for counting and one for data. This leads to two database round‑trips. It could be optimized by using a single query with `setProjection(Projections.rowCount())` and retrieving results via pagination, or by using a subquery.  
6. **Pagination Logic** – The calculation of `upperLimit` and `lowerLimit` is hidden inside `SearchCustomerCriteria`. If those methods misbehave, pagination could fail. Adding defensive checks or documentation would help.  
7. **Resource Leaks** – None detected; Hibernate sessions are managed by Spring.  
8. **Documentation** – Javadoc comments are missing. Adding method descriptions, parameter explanations, and return values would improve readability.  
9. **Testing** – No unit tests visible. Given the use of `HibernateTemplate`, DAO methods are suitable for integration testing with an in‑memory database.  

### Edge Cases & Potential Issues  

- **Null Inputs** – Many methods do not guard against null arguments (e.g., `findByUserNameAndPassword(null, null)`). This would result in `IllegalArgumentException` or `NullPointerException`.  
- **Concurrent Modifications** – `merge` and `saveOrUpdate` could lead to stale object updates if used improperly. Consider using optimistic locking.  
- **Case Sensitivity** – Username/password checks are case sensitive. Depending on requirements, this may need to be normalized.  
- **Large Result Sets** – `findUniqueCompanyName` returns all distinct company names without pagination; for merchants with thousands of customers, this could be expensive.  

### Future Enhancements  

1. **Upgrade to Hibernate 5 / JPA** – Replace deprecated APIs, use `EntityManager` instead of `HibernateTemplate`.  
2. **Spring Data JPA** – Re‑implement DAO using Spring Data repositories to reduce boilerplate.  
3. **Batch Operations** – Introduce batch insert/update for large collections.  
4. **Caching** – Add second‑level cache (e.g., Ehcache) for frequently accessed customers.  
5. **Method Extraction** – Extract common query logic (e.g., filtering by merchant) into reusable helper methods or Specification objects.  
6. **Parameter Validation** – Add `Objects.requireNonNull` or custom validation to guard against null inputs.  
7. **Enhanced Logging** – Include method names and parameter values in logs for easier debugging.  
8. **Unit Tests** – Write comprehensive tests using Spring's `@DataJpaTest` or a dedicated `HibernateTemplate` test configuration.  

---  

**Verdict**  
The DAO provides a complete set of CRUD and search operations for the `Customer` domain. It follows standard patterns for a Spring/Hibernate 3 application. However, the code would benefit from modernization (Hibernate 5/JPA), removal of redundant error handling, improved naming, and added defensive coding to address edge cases. With these adjustments, the DAO would become more maintainable, performant, and aligned with current best practices.

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
package com.salesmanager.core.service.customer.impl.dao;

import java.util.Collection;
import java.util.List;

import org.apache.commons.lang.StringUtils;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.hibernate.Criteria;
import org.hibernate.Hibernate;
import org.hibernate.Query;
import org.hibernate.SessionFactory;
import org.hibernate.criterion.Projections;
import org.hibernate.criterion.Restrictions;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.orm.hibernate3.support.HibernateDaoSupport;
import org.springframework.stereotype.Repository;

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.customer.CustomerBasket;
import com.salesmanager.core.entity.customer.CustomerBasketAttribute;
import com.salesmanager.core.entity.customer.SearchCustomerCriteria;
import com.salesmanager.core.entity.customer.SearchCustomerResponse;

@Repository
public class CustomerDao extends HibernateDaoSupport implements ICustomerDao {

	private static final Log log = LogFactory.getLog(CustomerDao.class);

	@Autowired
	public CustomerDao(SessionFactory sessionFactory) {
		super.setSessionFactory(sessionFactory);
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.customer.impl.ICustomerDao#saveShoppingCart
	 * (com.salesmanager.core.entity.customer.CustomerBasket)
	 */
	public void saveShoppingCart(CustomerBasket transientInstance) {
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
	 * @seecom.salesmanager.core.service.customer.impl.ICustomerDao#
	 * saveShoppingCartAttributes
	 * (com.salesmanager.core.entity.customer.CustomerBasketAttribute)
	 */
	public void saveShoppingCartAttributes(
			CustomerBasketAttribute transientInstance) {
		try {
			super.getHibernateTemplate().persist(transientInstance);
		} catch (RuntimeException re) {
			log.error("persist failed", re);
			throw re;
		}
	}

	public void persist(Customer transientInstance) {
		try {
			super.getHibernateTemplate().persist(transientInstance);
		} catch (RuntimeException re) {
			log.error("persist failed", re);
			throw re;
		}
	}

	public void saveOrUptade(Customer instance) {
		try {
			super.getHibernateTemplate().saveOrUpdate(instance);
		} catch (RuntimeException re) {
			log.error("attach failed", re);
			throw re;
		}
	}

	public Customer merge(Customer detachedInstance) {
		try {
			Customer result = (Customer) super.getHibernateTemplate().merge(
					detachedInstance);
			return result;
		} catch (RuntimeException re) {
			log.error("merge failed", re);
			throw re;
		}
	}

	public void delete(Customer persistentInstance) {
		try {
			super.getHibernateTemplate().delete(persistentInstance);
		} catch (RuntimeException re) {
			log.error("delete failed", re);
			throw re;
		}
	}

	public void deleteAll(Collection<Customer> customers) {
		try {
			super.getHibernateTemplate().deleteAll(customers);
		} catch (RuntimeException re) {
			log.error("deleteAll failed", re);
			throw re;
		}
	}

	public Customer findById(long id) {
		try {
			Customer instance = (Customer) super.getHibernateTemplate().get(
					"com.salesmanager.core.entity.customer.Customer", id);

			return instance;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Collection<Customer> findByMerchantId(int merchantId) {
		try {
			List list = super.getSession().createCriteria(Customer.class).add(
					Restrictions.eq("merchantId", merchantId)).list();
			return list;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Customer findByUserNameAndPassword(String userName, String password) {
		try {

			Customer customer = (Customer) super
					.getSession()
					.createQuery(
							"select c from Customer c where c.customerNick=:cId and c.customerPassword=:pId")
					.setString("cId", userName).setString("pId", password)
					.uniqueResult();

			return customer;

		} catch (Exception e) {
			log.error("get failed", e);
			throw new RuntimeException(e);
		}
	}

	public Customer findByUserNameAndPasswordByMerchantId(String userName,
			String password, int merchantId) {
		try {
			Customer customer = (Customer) super.getSession().createCriteria(
					Customer.class).add(
					Restrictions.eq("customerNick", userName)).add(
					Restrictions.eq("customerPassword", password)).add(
					Restrictions.eq("merchantId", merchantId)).uniqueResult();
			return customer;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Collection<Customer> findByCompanyName(String companyName,
			int merchantId) {
		try {
			List l = super.getSession().createCriteria(Customer.class).add(
					Restrictions.eq("customerCompany", companyName)).add(
					Restrictions.eq("merchantId", merchantId)).list();

			return l;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Collection<Customer> findCustomersHavingCompany(int merchantId) {
		try {
			List l = super.getSession().createCriteria(Customer.class).add(
					Restrictions.isNotNull("customerCompany")).add(
					(Restrictions.eq("merchantId", merchantId))).list();

			return l;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public List<String> findUniqueCompanyName(int merchantId) {

		try {

			Query q = super
					.getSession()
					.createSQLQuery(
							"select distinct c.customers_company from customers c where c.merchantId=:p and c.customers_company IS NOT NULL order by c.customers_company asc")
					.addScalar("customers_company", Hibernate.STRING)
					.setParameter("p", merchantId);

			List entries = q.list();

			return entries;

		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}

	}

	@SuppressWarnings("unchecked")
	public Customer findCustomerbyEmail(final String email) {

		Customer c = (Customer) super.getSession().createCriteria(
				Customer.class).add(
				Restrictions.eq("customerEmailAddress", email)).uniqueResult();

		return c;
	}

	public SearchCustomerResponse findCustomers(
			SearchCustomerCriteria searchCriteria) {

		Criteria criteria = super.getSession().createCriteria(Customer.class)
				.add(
						Restrictions.eq("merchantId", searchCriteria
								.getMerchantId())).addOrder(
						org.hibernate.criterion.Order.desc("customerLastname"))
				.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);

		Criteria query = super.getSession().createCriteria(Customer.class).add(
				Restrictions.eq("merchantId", searchCriteria.getMerchantId()))
				.addOrder(
						org.hibernate.criterion.Order.desc("customerLastname"))
				.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY);

		if (searchCriteria != null) {

			if (!StringUtils.isBlank(searchCriteria.getCustomerName())) {
				criteria.add(Restrictions.like("customerLastname", "%"
						+ searchCriteria.getCustomerName() + "%"));
				query.add(Restrictions.like("customerLastname", "%"
						+ searchCriteria.getCustomerName() + "%"));
			}

			if (!StringUtils.isBlank(searchCriteria.getEmail())) {
				criteria.add(Restrictions.like("customerEmailAddress", "%"
						+ searchCriteria.getEmail() + "%"));
				query.add(Restrictions.like("customerEmailAddress", "%"
						+ searchCriteria.getEmail() + "%"));
			}

			if (!StringUtils.isBlank(searchCriteria.getCompanyName())) {
				criteria.add(Restrictions.like("customerCompany", "%"
						+ searchCriteria.getCompanyName() + "%"));
				query.add(Restrictions.like("customerCompany", "%"
						+ searchCriteria.getCompanyName() + "%"));
			}

		}

		criteria.setProjection(Projections.rowCount());
		Integer count = (Integer) criteria.uniqueResult();

		criteria.setProjection(null);

		int max = searchCriteria.getQuantity();

		List list = null;
		if (count > 0) {
			query.setMaxResults(searchCriteria.getUpperLimit(count));
			query.setFirstResult(searchCriteria.getLowerLimit());
		}

		list = query.list();

		SearchCustomerResponse response = new SearchCustomerResponse();
		response.setCount(count);
		response.setCustomers(list);

		return response;

	}

	public Customer findCustomerbyUserName(final String userName,
			final int merchantId) {

		Customer c = (Customer) super.getSession().createCriteria(
				Customer.class).add(Restrictions.eq("customerNick", userName))
				.add(Restrictions.eq("merchantId", merchantId)).uniqueResult();

		return c;
	}

}



```
