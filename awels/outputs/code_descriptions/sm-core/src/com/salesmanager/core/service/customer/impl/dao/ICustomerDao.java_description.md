# ICustomerDao.java

## Review

## 1. Summary  
**Purpose** – The `ICustomerDao` interface defines a contract for CRUD and query operations on customer‑related entities (`Customer`, `CustomerBasket`, `CustomerBasketAttribute`) within the SalesManager core module.  
**Key Components**  
- **CRUD methods** for persisting, updating, merging, and deleting `Customer` objects.  
- **Shopping cart helpers** (`saveShoppingCart`, `saveShoppingCartAttributes`).  
- **Search & lookup** utilities that support various filters (by id, merchant id, company name, email, user‑name/password, etc.).  
- **Bulk operations** such as `deleteAll`.  
- **Result wrappers** (`SearchCustomerResponse`) that encapsulate paging, sorting, or filtering results.  
**Design Patterns & Frameworks**  
- *DAO* (Data Access Object) pattern: decouples persistence logic from business logic.  
- Likely used with **JPA/Hibernate** (given methods like `merge`, `persist`, `saveOrUptade`), although the interface itself is framework‑agnostic.  
- Uses **collections** (`Collection`, `List`) to return multiple results, adhering to Java Collections Framework conventions.

---

## 2. Detailed Description  
The interface describes all interactions a service layer would have with the persistence layer regarding customers and their shopping carts.

1. **Initialization** – No explicit init logic; implementing classes will configure entity managers, sessions, or DAOs via dependency injection (Spring, CDI, etc.).  
2. **Runtime Behavior** –  
   - **Create**: `saveShoppingCart`, `saveShoppingCartAttributes`, `persist`, `saveOrUptade`.  
   - **Read**: `findById`, `findByMerchantId`, `findUniqueCompanyName`, `findByUserNameAndPassword`, `findByUserNameAndPasswordByMerchantId`, `findByCompanyName`, `findCustomersHavingCompany`, `findCustomerbyEmail`, `findCustomerbyUserName`.  
   - **Update**: `merge` (detached to persistent), `saveOrUptade`.  
   - **Delete**: `delete`, `deleteAll`.  
   - **Search**: `findCustomers` returns a `SearchCustomerResponse` encapsulating paging/sorting.  
3. **Cleanup** – Not defined; closing of sessions or transactions is expected to be handled by the transaction manager or by the concrete DAO implementation.  

### Assumptions & Constraints  
- **Uniqueness**: Methods like `findUniqueCompanyName` imply uniqueness constraints per merchant.  
- **Merchant scoping**: Many methods require a `merchantId` to isolate data per tenant.  
- **Password handling**: `findByUserNameAndPassword` assumes plain password comparison; secure implementations should hash/compare accordingly.  
- **Nullability**: Implementations must decide how to handle null/empty results (e.g., return `null` vs. empty collections).  

### Architecture  
A classic layered architecture:  
- **DAO Layer** (`ICustomerDao`) – abstraction.  
- **Service Layer** – would inject this interface to perform business operations.  
- **Persistence Layer** – concrete DAO implementation using JPA/Hibernate, JDBC, or another ORM.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `saveShoppingCart(CustomerBasket)` | Persist a new shopping cart | `CustomerBasket` instance | `void` | Stores cart in DB |
| `saveShoppingCartAttributes(CustomerBasketAttribute)` | Persist cart attribute | `CustomerBasketAttribute` instance | `void` | Stores attribute |
| `persist(Customer)` | Persist a new customer | `Customer` instance | `void` | Stores customer |
| `saveOrUptade(Customer)` | Upsert a customer (create or update) | `Customer` instance | `void` | Persists or updates record |
| `merge(Customer)` | Merge a detached customer into persistence context | Detached `Customer` | Managed `Customer` | Returns managed entity |
| `delete(Customer)` | Delete a customer | `Customer` instance | `void` | Removes record |
| `findById(long)` | Retrieve by primary key | `id` | `Customer` | `null` if not found |
| `findByMerchantId(int)` | All customers belonging to a merchant | `merchantId` | `Collection<Customer>` | Ordered arbitrarily |
| `findUniqueCompanyName(int)` | Retrieve distinct company names for a merchant | `merchantId` | `List<String>` | Unique values |
| `findByUserNameAndPassword(String,String)` | Auth lookup (non‑merchant scoped) | `userName`, `password` | `Customer` | `null` if credentials invalid |
| `findByUserNameAndPasswordByMerchantId(String,String,int)` | Auth lookup scoped by merchant | `userName`, `password`, `merchantId` | `Customer` | `null` if invalid |
| `findByCompanyName(String,int)` | Search customers by company name & merchant | `companyName`, `merchantId` | `Collection<Customer>` | Case‑sensitive/insensitive per implementation |
| `findCustomersHavingCompany(int)` | All customers that have a company record | `merchantId` | `Collection<Customer>` | |
| `findCustomerbyEmail(String)` | Lookup by email (global) | `email` | `Customer` | `null` if none |
| `findCustomerbyUserName(String,int)` | Lookup by username scoped to merchant | `userName`, `merchantId` | `Customer` | `null` if none |
| `deleteAll(Collection<Customer>)` | Bulk delete | Collection of `Customer` | `void` | Removes all specified |
| `findCustomers(SearchCustomerCriteria)` | Paginated/filtered search | Criteria object | `SearchCustomerResponse` | Encapsulates results + metadata |

**Reusable / Utility Methods** – None defined; implementations may expose helpers internally (e.g., query builders).  

---

## 4. Dependencies  

| Category | Item | Type | Remarks |
|----------|------|------|---------|
| **Core Java** | `java.util.Collection`, `java.util.List` | Standard | |
| **Domain Entities** | `Customer`, `CustomerBasket`, `CustomerBasketAttribute`, `SearchCustomerCriteria`, `SearchCustomerResponse` | Project | Must be JPA entities or POJOs |
| **Persistence** | Methods such as `merge`, `persist` | Implicit | Suggests JPA/Hibernate usage |
| **Security** | None explicitly; `findByUserNameAndPassword` hints at authentication logic | — | Implementations must handle password hashing |

No external frameworks are imported directly; the interface stays framework‑agnostic.  

---

## 5. Additional Notes  

### Edge Cases & Potential Issues  
1. **Password Security** – The method signatures expose plain‑text passwords. Implementations should avoid storing or comparing raw passwords; a separate authentication service is preferable.  
2. **Null Handling** – Consistent contract for `null` vs. empty collections should be documented; callers may otherwise face `NullPointerException`.  
3. **Concurrent Updates** – `merge` and `saveOrUptade` might need optimistic locking; otherwise concurrent modifications could overwrite each other.  
4. **Bulk Operations** – `deleteAll` could be expensive for large collections; consider batch processing or native SQL for performance.  
5. **Pagination** – `findCustomers` returns a custom response; ensure it includes total count, page size, and current page for client‑side navigation.  

### Future Enhancements  
- **Generic DAO Base** – Extract common CRUD methods into a generic base interface to reduce boilerplate across entities.  
- **Specification / Criteria API** – Replace `SearchCustomerCriteria` with JPA Criteria API or a specification pattern for type‑safe querying.  
- **Soft Delete** – Introduce a `deleted` flag to avoid physical removal; adapt `delete` methods accordingly.  
- **Audit Trail** – Add methods or fields to capture creation/update timestamps and users.  
- **Async Support** – Provide CompletableFuture or reactive variants for non‑blocking I/O.  

### Suggested Naming Fix  
`saveOrUptade` contains a typo; it should be `saveOrUpdate`.  

Overall, the interface is well‑structured, clearly separates concerns, and lays a solid foundation for implementing customer‑related persistence logic in the SalesManager core module.

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

import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.customer.CustomerBasket;
import com.salesmanager.core.entity.customer.CustomerBasketAttribute;
import com.salesmanager.core.entity.customer.SearchCustomerCriteria;
import com.salesmanager.core.entity.customer.SearchCustomerResponse;

public interface ICustomerDao {

	public abstract void saveShoppingCart(CustomerBasket transientInstance);

	public abstract void saveShoppingCartAttributes(
			CustomerBasketAttribute transientInstance);

	public void persist(Customer transientInstance);

	public void saveOrUptade(Customer instance);

	public Customer merge(Customer detachedInstance);

	public void delete(Customer persistentInstance);

	public Customer findById(long id);

	public Collection<Customer> findByMerchantId(int merchantId);

	public List<String> findUniqueCompanyName(int merchantId);

	public Customer findByUserNameAndPassword(String userName, String password);

	public Customer findByUserNameAndPasswordByMerchantId(String userName,
			String password, int merchantId);

	public Collection<Customer> findByCompanyName(String companyName,
			int merchantId);

	public Collection<Customer> findCustomersHavingCompany(int merchantId);

	public Customer findCustomerbyEmail(final String email);

	public Customer findCustomerbyUserName(final String userName,
			final int merchantId);

	public void deleteAll(Collection<Customer> customers);

	public SearchCustomerResponse findCustomers(
			SearchCustomerCriteria searchCriteria);

}


```
