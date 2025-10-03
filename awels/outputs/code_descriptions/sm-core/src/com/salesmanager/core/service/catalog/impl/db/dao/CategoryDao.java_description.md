# CategoryDao.java

## Review

## 1. Summary  

`CategoryDao` is a Spring‑managed Hibernate DAO that performs CRUD and query operations on the `Category` entity.  
* **Purpose** – Persist, merge, delete, and retrieve `Category` objects, including support for multi‑merchant, multi‑language, and hierarchical queries.  
* **Key components**  
  * `persist`, `saveOrUpdate`, `merge`, `delete` – standard CRUD via `HibernateTemplate`.  
  * `save` – a manual `PreparedStatement` insert used instead of the ORM.  
  * Several finder methods (`findById`, `findByMerchantId`, `findByCategoryIds`, etc.) that build HQL/Criteria queries to return collections of categories, often pre‑fetching related descriptions and parent entities.  
* **Design patterns & libraries**  
  * **DAO pattern** – encapsulates persistence logic.  
  * **Spring Repository stereotype** – `@Repository` for exception translation.  
  * **Hibernate 3** – `SessionFactory`, `Session`, `Criteria`, `HibernateTemplate`.  
  * **HQL** – string‑based queries with named parameters.  
  * **Expression / Restrictions** – legacy Criteria API.  

---

## 2. Detailed Description  

### Initialization  
* The DAO is constructed by Spring with a `SessionFactory`.  
* `setSessionFactory(sessionFactory)` is called via the superclass `HibernateDaoSupport`.  
* The `SessionFactory` is stored in the superclass and reused for each operation.

### Runtime Behaviour  
Each method obtains a Hibernate `Session` (either via `HibernateTemplate` or directly from the session factory) and executes the intended database operation:

| Method | Operation | Implementation details |
|--------|-----------|------------------------|
| `persist`, `saveOrUpdate`, `merge`, `delete` | ORM‑based CRUD | Delegated to `HibernateTemplate`. |
| `save` | Manual insert | Uses `Session.connection()` to get a JDBC connection and a `PreparedStatement`. |
| `findById` | Load single `Category` | HQL with `left join fetch` on descriptions and parent. |
| `findByCategoryIds` | Batch fetch | `DetachedCriteria` with `Expression.in`. |
| `findByMerchantId` | Merchant filtering | `Criteria` with `Restrictions.in`. |
| `findByMerchantIdAndLineage`, `findByMerchantIdAndLanguageIdAndLineage`, `findByMerchantIdAndLanguageId` | Complex hierarchical queries | HQL with `join fetch`, `like`, and language filtering. |
| `findByMerchantIdAndLanguage` | Incorrect reference | Uses `Product` entity in query – likely a bug. |
| `findSubCategories`, `findSubCategoriesByLang` | Child categories | `Criteria` or HQL with parent id filter. |
| `findCategoryByMerchantIdAndSeoURLAndByLang` | SEO lookup | HQL filtering on description’s `seUrl` and language. |
| `deleteCategories` | Batch delete | `HibernateTemplate.deleteAll`. |

### Cleanup  
* `HibernateTemplate` automatically manages sessions and transactions (when Spring transactions are configured).  
* The manual `save` method does **not** close the `PreparedStatement` or the connection, which can lead to resource leaks.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Returns | Side‑Effects | Notes |
|--------|---------|--------|---------|--------------|-------|
| `persist(Category)` | Persist a transient entity | `Category` | void | May throw runtime exception | Uses `HibernateTemplate.persist`. |
| `saveOrUpdate(Category)` | Persist or update | `Category` | void | May throw runtime exception | Uses `HibernateTemplate.saveOrUpdate`. |
| `saveOrUpdateAll(Collection<Category>)` | Batch persist/update | `Collection` | void | May throw runtime exception | Uses `HibernateTemplate.saveOrUpdateAll`. |
| `save(Category)` | Direct SQL insert | `Category` | Integer (rows affected) via `PreparedStatement.executeUpdate` | Bypasses ORM; may cause data inconsistency if entity mapping changes. | Manual JDBC code – no resource cleanup. |
| `delete(Category)` | Delete entity | `Category` | void | May throw runtime exception | Uses `HibernateTemplate.delete`. |
| `merge(Category)` | Merge detached state | `Category` | `Category` (merged) | May throw runtime exception | Uses `HibernateTemplate.merge`. |
| `findById(long)` | Retrieve by primary key | `long id` | `Category` | May throw runtime exception | HQL with fetch joins. |
| `deleteCategories(Collection<Category>)` | Batch delete | `Collection` | void | May throw runtime exception | Uses `HibernateTemplate.deleteAll`. |
| `findByCategoryIds(Collection<Long>)` | Batch fetch by IDs | `Collection<Long>` | `Collection<Category>` | May throw runtime exception | Uses `DetachedCriteria`. |
| `findByMerchantId(int)` | Find categories for merchant | `int merchantId` | `List<Category>` | May throw runtime exception | Includes global merchant ID. |
| `findByMerchantIdAndLineage(int,String)` | Find categories by lineage | `int merchantId`, `String lineage` | `Collection<Category>` | May throw runtime exception | HQL with `like`. |
| `findByMerchantIdAndLanguageIdAndLineage(int,int,String)` | Same as above with language filter | `int merchantId`, `int languageId`, `String lineage` | `Collection<Category>` | May throw runtime exception | |
| `findByMerchantIdAndLanguageId(int,int)` | All categories for merchant & language | `int merchantId`, `int languageId` | `Collection<Category>` | May throw runtime exception | |
| `findByMerchantIdAndLanguage(int,int)` | **Buggy** – uses `Product` class instead of `Category` | `int merchantId`, `int language` | `List<Category>` | May throw runtime exception | Likely copy‑paste error. |
| `findSubCategories(long)` | Children of a category | `long categoryId` | `List<Category>` | May throw runtime exception | |
| `findSubCategoriesByLang(int,long,int)` | Children with language filter | `int merchantId`, `long categoryId`, `int languageId` | `List<Category>` | May throw runtime exception | |
| `findCategoryByMerchantIdAndSeoURLAndByLang(int,String,int)` | SEO lookup | `int merchantId`, `String seUrl`, `int languageId` | `Category` | May throw runtime exception | |

Reusable utilities:  
* The HQL/Criteria fragments are duplicated across methods – a helper method to build common predicates would reduce repetition.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| **Spring ORM** (`org.springframework.orm.hibernate3.*`) | Third‑party | Provides `HibernateDaoSupport` and `HibernateTemplate`. |
| **Hibernate 3.2.0.b9** (`org.hibernate.*`) | Third‑party | Legacy Criteria API (`Expression`, `Restrictions`), deprecated in newer Hibernate versions. |
| **Apache Commons Logging** (`org.apache.commons.logging.*`) | Third‑party | Simple logging abstraction. |
| **Java SQL** (`java.sql.*`) | JDK | Used for manual JDBC insertion. |
| **SalesManager Core** (`com.salesmanager.core.*`) | Internal | Entity classes and constants. |

No platform‑specific libraries; however, the code relies on a `SessionFactory` bean configured elsewhere (likely via Spring).

---

## 5. Additional Notes & Recommendations  

### Code Quality Issues  
1. **Mixed Persistence Approaches** – The `save` method bypasses Hibernate entirely, risking mapping drift and violating transaction semantics.  
2. **Resource Leak** – `PreparedStatement` and `Connection` are never closed. In a long‑running application this will exhaust JDBC resources.  
3. **Deprecated APIs** – The Criteria API (`Expression.in`, `Expression`, `Restrictions`) is obsolete; consider migrating to JPA Criteria or native HQL.  
4. **Raw Types** – Methods return `Collection` or `List` without generics, reducing type safety.  
5. **Duplicate Logic** – Similar query fragments are repeated; extract reusable predicates or use a query builder.  
6. **Buggy Method** – `findByMerchantIdAndLanguage` references the `Product` entity instead of `Category`.  
7. **Exception Handling** – All methods catch `RuntimeException` and re‑throw it after logging. This is unnecessary and can mask underlying checked exceptions; let Spring translate exceptions automatically.  
8. **Missing Transactionality** – No explicit transaction demarcation; rely on Spring’s declarative transactions. Ensure that configuration is in place.  

### Suggested Enhancements  
- **Unify persistence strategy**: Remove the manual JDBC `save` method; use `HibernateTemplate.save` or the newer `JpaRepository` if moving to Spring Data JPA.  
- **Close resources**: If manual JDBC is required, wrap in try‑with‑resources or Spring’s `JdbcTemplate`.  
- **Migrate to Hibernate 5/6 or JPA**: Replace `Expression`, `Restrictions`, and `Criteria` with `CriteriaBuilder`/`CriteriaQuery` or JPQL.  
- **Type safety**: Use generics (`Collection<Category>`, `List<Category>`) throughout.  
- **Refactor query building**: Create private helper methods that return `Criteria` or `Query` objects with common predicates.  
- **Fix bug**: Correct the entity reference in `findByMerchantIdAndLanguage`.  
- **Unit tests**: Add DAO integration tests using an in‑memory database (H2) to verify query correctness.  
- **Logging**: Replace `Log` with `SLF4J` for better abstraction and optional binding.  

By addressing these points the DAO will become more maintainable, safer, and future‑proof.

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

// Generated Aug 7, 2008 10:59:18 AM by Hibernate Tools 3.2.0.b9

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;

import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.hibernate.Criteria;
import org.hibernate.HibernateException;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.criterion.DetachedCriteria;
import org.hibernate.criterion.Expression;
import org.hibernate.criterion.Restrictions;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.orm.hibernate3.HibernateCallback;
import org.springframework.orm.hibernate3.support.HibernateDaoSupport;
import org.springframework.stereotype.Repository;

import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.entity.catalog.Category;

/**
 * Home object for domain model class Category.
 * 
 * @see com.salesmanager.core.service.catalog.impl.Category
 * @author Hibernate Tools
 */
@Repository
public class CategoryDao extends HibernateDaoSupport implements ICategoryDao {

	private static final Log log = LogFactory.getLog(CategoryDao.class);

	@Autowired
	public CategoryDao(SessionFactory sessionFactory) {
		super.setSessionFactory(sessionFactory);
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @seecom.salesmanager.core.service.catalog.impl.ICategoryDao#persist(com.
	 * salesmanager.core.entity.catalog.Category)
	 */
	public void persist(Category transientInstance) {
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
	 * com.salesmanager.core.service.catalog.impl.ICategoryDao#saveOrUpdate(
	 * com.salesmanager.core.entity.catalog.Category)
	 */
	public void saveOrUpdate(Category instance) {
		try {
			super.getHibernateTemplate().saveOrUpdate(instance);
		} catch (RuntimeException re) {
			log.error("attach failed", re);
			throw re;
		}
	}

	public void saveOrUpdateAll(Collection<Category> instances) {
		try {
			super.getHibernateTemplate().saveOrUpdateAll(instances);
		} catch (RuntimeException re) {
			log.error("attach failed", re);
			throw re;
		}
	}

	public void save(final Category instance) {
		getHibernateTemplate().execute(new HibernateCallback() {
			public Object doInHibernate(Session session)
					throws HibernateException, SQLException {
				Connection con = session.connection();
				PreparedStatement ps = con
						.prepareStatement("insert into categories(categories_id,categories_image,parent_id,"
								+ "sort_order,date_added,last_modified,categories_status,visible,RefCategoryID,"
								+ "RefCategoryLevel,RefCategoryName,RefCategoryParentID,RefExpired,merchantid,depth,"
								+ "lineage) values(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)");
				ps.setLong(1, instance.getCategoryId());
				ps.setString(2, instance.getCategoryImage());
				ps.setLong(3, instance.getParentId());
				ps.setInt(4, (instance.getSortOrder() == null ? 0 : instance
						.getSortOrder()));
				ps.setDate(5, new java.sql.Date(instance.getDateAdded()
						.getTime()));
				ps.setDate(6, new java.sql.Date(instance.getLastModified()
						.getTime()));
				ps.setBoolean(7, instance.isCategoryStatus());
				ps.setBoolean(8, instance.isVisible());
				ps.setLong(9, instance.getRefCategoryId());
				ps.setInt(10, instance.getRefCategoryLevel());
				ps.setString(11, instance.getRefCategoryName());
				ps.setString(12, instance.getRefCategoryParentId());
				ps.setString(13, instance.getRefExpired());
				ps.setLong(14, instance.getMerchantId());
				ps.setInt(15, instance.getDepth());
				ps.setString(16, instance.getLineage());
				return ps.executeUpdate();
			}

		});
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @seecom.salesmanager.core.service.catalog.impl.ICategoryDao#delete(com.
	 * salesmanager.core.entity.catalog.Category)
	 */
	public void delete(Category persistentInstance) {
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
	 * @seecom.salesmanager.core.service.catalog.impl.ICategoryDao#merge(com.
	 * salesmanager.core.entity.catalog.Category)
	 */
	public Category merge(Category detachedInstance) {
		try {
			Category result = (Category) super.getHibernateTemplate().merge(
					detachedInstance);
			return result;
		} catch (RuntimeException re) {
			log.error("merge failed", re);
			throw re;
		}
	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.ICategoryDao#findById(int)
	 */
	public Category findById(long id) {
		try {

			return (Category) super
					.getSession()
					.createQuery(
							"select c from Category c left join fetch c.descriptions s join fetch c.parent where c.categoryId=:cId")
					.setLong("cId", id).uniqueResult();

		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public void deleteCategories(Collection<Category> categories) {

		try {
			super.getHibernateTemplate().deleteAll(categories);
		} catch (RuntimeException e) {
			log.error(e);
			throw e;

		}

	}

	public Collection<Category> findByCategoryIds(Collection<Long> categoryIds) {

		try {

			if (categoryIds == null)
				return null;

			DetachedCriteria crit = DetachedCriteria.forClass(Category.class);
			crit.add(Expression.in("categoryId", categoryIds));
			Collection result = this.getHibernateTemplate()
					.findByCriteria(crit);

			return result;

		} catch (RuntimeException e) {
			log.error("get failed", e);
			throw e;
		}

	}

	/*
	 * (non-Javadoc)
	 * 
	 * @see
	 * com.salesmanager.core.service.catalog.impl.ICategoryDao#findByMerchantId
	 * (int)
	 */
	public List<Category> findByMerchantId(int merchantid) {
		try {
			List values = new ArrayList();
			values.add(Constants.GLOBAL_MERCHANT_ID);
			values.add(merchantid);
			List list = super.getSession().createCriteria(Category.class).add(
					Restrictions.in("merchantId", values)).list();

			return list;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Collection<Category> findByMerchantIdAndLineage(int merchantId,
			String lineage) {
		try {
			List list = super
					.getSession()
					.createQuery(
							"select c from Category c left join fetch c.descriptions s join fetch c.parent where c.merchantId=:mId and c.lineage like :lin order by c.lineage, c.sortOrder")
					.setInteger("mId", merchantId).setString(
							"lin",
							new StringBuffer().append(lineage).append("%")
									.toString()).setResultTransformer(
							Criteria.DISTINCT_ROOT_ENTITY).list();

			return list;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Collection<Category> findByMerchantIdAndLanguageIdAndLineage(
			int merchantId, int languageId, String lineage) {
		try {

			List merchants = new ArrayList();
			merchants.add(Constants.GLOBAL_MERCHANT_ID);
			merchants.add(merchantId);

			List list = super
					.getSession()
					.createQuery(
							"select c from Category c left join fetch c.descriptions s join fetch c.parent where c.merchantId in (:mId) and s.id.languageId=:lId and c.lineage like :lin order by c.lineage, c.sortOrder")
					.setParameterList("mId", merchants).setInteger("lId",
							languageId).setString(
							"lin",
							new StringBuffer().append(lineage).append("%")
									.toString()).list();

			return list;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Collection<Category> findByMerchantIdAndLanguageId(int merchantId,
			int languageId) {
		try {

			List merchants = new ArrayList();
			merchants.add(Constants.GLOBAL_MERCHANT_ID);
			merchants.add(merchantId);

			List list = super
					.getSession()
					.createQuery(
							"select c from Category c left join fetch c.descriptions s join fetch c.parent where c.merchantId in (:mId) and s.id.languageId=:lId order by c.lineage, c.sortOrder")
					.setParameterList("mId", merchants).setInteger("lId",
							languageId).list();

			return list;
		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public List<Category> findByMerchantIdAndLanguage(int merchantId,
			int language) {
		try {

			Collection ids = new ArrayList();
			ids.add(Constants.GLOBAL_MERCHANT_ID);
			ids.add(merchantId);

			Criteria query = super.getSession().createCriteria(
					com.salesmanager.core.entity.catalog.Product.class).add(
					Expression.in("merchantId", ids)).setResultTransformer(
					Criteria.DISTINCT_ROOT_ENTITY);

			Criteria descCriteria = query.createCriteria("descriptions");
			descCriteria.add(Restrictions.eq("id.languageId", language));

			List list = query.list();

			return list;

		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public List<Category> findSubCategories(long categoryId) {
		try {

			List list = super.getSession().createCriteria(Category.class).add(
					Restrictions.eq("parentId", categoryId))
					.setResultTransformer(Criteria.DISTINCT_ROOT_ENTITY).list();

			return list;

		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public List<Category> findSubCategoriesByLang(int merchantId,
			long categoryId, int languageId) {
		try {

			List merchants = new ArrayList();
			merchants.add(Constants.GLOBAL_MERCHANT_ID);
			merchants.add(merchantId);

			List list = super
					.getSession()
					.createQuery(
							"select c from Category c left join fetch c.descriptions s join fetch c.parent where c.merchantId in (:mId) and c.parentId=:pId and s.id.languageId=:lId order by c.lineage, c.sortOrder")
					.setParameterList("mId", merchants).setInteger("lId",
							languageId).setLong("pId", categoryId).list();

			return list;

		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

	public Category findCategoryByMerchantIdAndSeoURLAndByLang(int merchantId,
			String seUrl, int languageId) {

		try {

			Category c = null;

			List list = super
					.getSession()
					.createQuery(
							"select c from Category c left join fetch c.descriptions s where c.merchantId=:mId and s.seUrl=:sText and s.id.languageId=:lId order by c.sortOrder")
					.setInteger("mId", merchantId).setString("sText", seUrl)
					.setInteger("lId", languageId).list();

			if (list != null && list.size() > 0) {
				c = (Category) list.get(0);// get first item
			}

			return c;

		} catch (RuntimeException re) {
			log.error("get failed", re);
			throw re;
		}
	}

}



```
