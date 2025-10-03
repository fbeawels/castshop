# GlobalReferenceDao.java

## Review

## 1. Summary
The `GlobalReferenceDao` is a Spring‐managed DAO that exposes a handful of read‑only queries used throughout the SalesManager core module to fetch reference data (product types, zones, currencies, etc.).  
Key points:

| Component | Role |
|-----------|------|
| `@Repository` | Marks the class as a persistence bean for component scanning. |
| `HibernateDaoSupport` | Provides a convenient wrapper around `HibernateTemplate` (Hibernate 3). |
| `SessionFactory` | Injected via constructor to enable session creation. |
| `IGlobalReferenceDao` | Interface (not shown) that declares the public methods. |
| Methods (`getProductTypes()`, `getZones()`, …) | Each builds a Hibernate `Criteria` query, executes it, and returns the result. |

The DAO is intentionally *read‑only*; it only retrieves reference entities. No transaction demarcation or write logic is present.

---

## 2. Detailed Description
### Flow of Execution
1. **Initialization** – Spring scans the package, instantiates `GlobalReferenceDao`, injects a `SessionFactory`, and sets it on the superclass (`HibernateDaoSupport`).  
2. **Method Call** – The service layer calls one of the public methods (e.g. `getCurrencies()`).  
3. **Query Construction** – Inside the method, `super.getSession()` obtains the current `Session`.  
4. **Criteria Building** – A `Criteria` instance for the entity class is created; optional restrictions or order clauses are chained.  
5. **Execution & Return** – `list()` is called, the result (a raw `List`) is returned as a `Collection`.  
6. **Error Handling** – Any `RuntimeException` is logged via Commons‑Logging and re‑thrown.

No cleanup code is required because Hibernate sessions are managed by Spring (session per request / transaction).

### Design Choices & Constraints
- **Hibernate 3** – The DAO relies on `org.hibernate.criterion.*`, which has been deprecated since Hibernate 4.  
- **Raw Types** – `List` and `Map` are used without generics, sacrificing type safety.  
- **Single Read‑Only DAO** – All reference queries share the same DAO, keeping the API small but tightly coupling the entity classes to this bean.  
- **Logging** – Uses `LogFactory.getLog(CategoryDao.class)` – likely a copy‑paste mistake; the logger should reference `GlobalReferenceDao.class`.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `getProductTypes()` | Fetch all `ProductType` entities. | None | `Collection<ProductType>` | Logs and throws if query fails. |
| `getZones()` | Fetch all `Zone` entities, ordered by country ID then code. | None | `Collection<Zone>` | Same error handling. |
| `getCountryStatus()` | Fetch all `CentralCountryStatus` entities. | None | `Collection<CentralCountryStatus>` | |
| `getOrderStatus()` | Fetch all `OrderStatus` entities. | None | `Collection<OrderStatus>` | |
| `getCurrencies()` | Fetch all supported `Currency` entities (`supported = true`) ordered by ID. | None | `Collection<Currency>` | |
| `getMeasureUnits()` | Fetch all `CentralMeasureUnits`. | None | `Collection<CentralMeasureUnits>` | |
| `getLanguages()` | Fetch all `Language` entities. | None | `Collection<Language>` | |
| `getSupportedCreditCards()` | Build a `Map<Integer, CentralCreditCard>` keyed by card ID. | None | `Map<Integer, CentralCreditCard>` | Constructs and populates the map; uses raw types. |

*Reusable pattern*: Each method follows the same `try/catch` block wrapping a `Criteria` query. The only variation is the entity class and optional restrictions/ordering.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.springframework.orm.hibernate3.support.HibernateDaoSupport` | Spring core | Supports Hibernate 3. |
| `org.hibernate.SessionFactory` | Hibernate | Obtained via Spring. |
| `org.hibernate.criterion.*` | Hibernate | Criteria API (deprecated). |
| `org.apache.commons.logging.Log` | Commons Logging | Logging abstraction. |
| `com.salesmanager.core.entity.*` | Domain entities | JPA/Hibernate entities. |
| `com.salesmanager.core.service.catalog.impl.db.dao.CategoryDao` | Internal | Imported only for logger class; seems a copy‑paste error. |

All are **third‑party** libraries except the domain entities, which are part of the same project.

---

## 5. Additional Notes

### Strengths
- **Simplicity** – Straightforward read‑only queries; easy to understand.  
- **Single Point of Reference** – All global lookups are centralized.  
- **Transactional Safety** – Spring manages sessions; no manual cleanup.

### Weaknesses / Risks
1. **Deprecated API** – Hibernate 3 is no longer supported; upgrading to Hibernate 5/JPA Criteria would be safer.  
2. **Raw Types** – Using `List`/`Map` without generics leads to unchecked casts and potential `ClassCastException`.  
3. **Logging Bug** – Logger references `CategoryDao.class` instead of the correct class, which may mislead debugging.  
4. **Exception Handling** – Rethrowing runtime exceptions without wrapping may expose Hibernate internals to upper layers.  
5. **No Caching** – All queries hit the database every time; for static reference data, a second‑level cache or in‑memory cache would reduce load.  
6. **Hard‑coded Entity Names** – Adding a new reference type requires adding a new method; consider a generic method that accepts an entity class.  

### Suggested Enhancements
- **Switch to Hibernate 5/JPA**: Replace `HibernateDaoSupport` with `JpaRepository` or `EntityManager`‑based DAO.  
- **Add Generics**: `public <T> Collection<T> findAll(Class<T> entity)` and a generic `Map<Integer, T> findAllAsMap(Class<T> entity, String keyProperty)`.  
- **Proper Logging**: `LogFactory.getLog(GlobalReferenceDao.class)`.  
- **Cache Support**: Enable Hibernate second‑level cache for reference tables or expose a simple in‑memory cache (e.g., `ConcurrentMap`).  
- **Unit Tests**: Add tests that validate each method returns expected results and that the map keys are correct.  
- **Documentation**: Javadoc on each method explaining the entity context and ordering semantics.

Overall, the DAO fulfills its current requirement but would benefit from modernization, type safety, and performance optimizations for a production environment.

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
package com.salesmanager.core.service.reference.impl.dao;

import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.hibernate.SessionFactory;
import org.hibernate.criterion.Order;
import org.hibernate.criterion.Restrictions;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.orm.hibernate3.support.HibernateDaoSupport;
import org.springframework.stereotype.Repository;

import com.salesmanager.core.entity.orders.OrderStatus;
import com.salesmanager.core.entity.reference.CentralCountryStatus;
import com.salesmanager.core.entity.reference.CentralCreditCard;
import com.salesmanager.core.entity.reference.CentralMeasureUnits;
import com.salesmanager.core.entity.reference.Currency;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.entity.reference.ProductType;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.catalog.impl.db.dao.CategoryDao;

@Repository
public class GlobalReferenceDao extends HibernateDaoSupport implements
		IGlobalReferenceDao {

	private static final Log log = LogFactory.getLog(CategoryDao.class);

	@Autowired
	public GlobalReferenceDao(SessionFactory sessionFactory) {
		super.setSessionFactory(sessionFactory);
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @seecom.salesmanager.core.service.catalog.impl.dao.ICatalogReferenceDao#
	 * getProductTypes()
	 */
	public Collection<ProductType> getProductTypes() {

		try {

			List list = super.getSession().createCriteria(ProductType.class)
					.list();

			return list;

		} catch (RuntimeException e) {
			log.error("get failed", e);
			throw e;
		}

	}

	public Collection<Zone> getZones() {

		try {

			List list = super.getSession().createCriteria(Zone.class).addOrder(
					Order.asc("zoneCountryId")).addOrder(Order.asc("zoneCode"))
					.list();

			return list;

		} catch (RuntimeException e) {
			log.error("get failed", e);
			throw e;
		}

	}

	public Collection<CentralCountryStatus> getCountryStatus() {

		try {

			List list = super.getSession().createCriteria(
					CentralCountryStatus.class).list();
			return list;

		} catch (RuntimeException e) {
			log.error("get failed", e);
			throw e;
		}

	}

	public Collection<OrderStatus> getOrderStatus() {

		try {

			List list = super.getSession().createCriteria(OrderStatus.class)
					.list();
			return list;

		} catch (RuntimeException e) {
			log.error("get failed", e);
			throw e;
		}

	}

	public Collection<Currency> getCurrencies() {

		try {

			List list = super.getSession().createCriteria(Currency.class).add(
					Restrictions.eq("supported", Boolean.TRUE)).addOrder(
					Order.asc("currencyId")).list();
			return list;

		} catch (RuntimeException e) {
			log.error("get failed", e);
			throw e;
		}

	}

	public Collection<CentralMeasureUnits> getMeasureUnits() {

		try {

			List list = super.getSession().createCriteria(
					CentralMeasureUnits.class).list();
			return list;

		} catch (RuntimeException e) {
			log.error("get failed", e);
			throw e;
		}

	}

	public Collection<Language> getLanguages() {

		try {

			List list = super.getSession().createCriteria(Language.class)
					.list();
			return list;

		} catch (RuntimeException e) {
			log.error("get failed", e);
			throw e;
		}

	}

	public Map getSupportedCreditCards() {

		/**
		 * Credit cards
		 */
		List list = super.getSession().createCriteria(CentralCreditCard.class)
				.addOrder(Order.asc("centralCreditCardPosition")).list();
		// List cc = (List)session.createQuery(
		// "from CentralCreditCard c order by c.centralCreditCardPosition asc").list();
		Map supportedCreditCards = new HashMap();
		if (list != null && list.size() > 0) {
			Iterator ccit = list.iterator();
			while (ccit.hasNext()) {
				CentralCreditCard ccd = (CentralCreditCard) ccit.next();
				supportedCreditCards.put(ccd.getCentralCreditCardId(), ccd);
			}
		}

		return supportedCreditCards;

	}

}



```
