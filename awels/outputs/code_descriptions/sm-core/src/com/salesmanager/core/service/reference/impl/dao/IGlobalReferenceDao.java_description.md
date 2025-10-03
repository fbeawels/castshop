# IGlobalReferenceDao.java

## Review

## 1. Summary
The file defines **`IGlobalReferenceDao`**, a pure interface intended to expose read‑only access to a wide range of reference data that is shared across the application.  
Key responsibilities include retrieving lists or maps of:

- Product types, currencies, languages, zones, measurement units, country status, order status, and supported credit card codes.  

The interface is designed for a *DAO* (Data Access Object) pattern, allowing concrete implementations to retrieve data from any source (database, XML, cache, etc.) while keeping the rest of the code decoupled from the persistence mechanism.

**Notable design choices**

- **Stateless DAO interface** – no stateful methods, only getters, implying read‑only, cache‑friendly usage.
- **Use of generic `Collection` and `Map`** – allows flexibility in concrete collection types but sacrifices type safety for the map.
- **Domain‑specific entities** – each method returns domain objects (`ProductType`, `Currency`, etc.) indicating the DAO belongs to the *core* service layer.

No frameworks or libraries are explicitly referenced; it relies only on Java SE collections and the domain entity classes.

---

## 2. Detailed Description
### Core Components

| Component | Purpose |
|-----------|---------|
| **`IGlobalReferenceDao`** | Contract for all reference‑data access. |
| **Entity classes** (`ProductType`, `Currency`, etc.) | Domain objects representing reference data. |

### Execution Flow (when implemented)

1. **Initialization** – A concrete implementation (e.g., `GlobalReferenceDaoImpl`) would be instantiated, typically by a dependency injection framework (Spring, CDI, etc.) or manually.
2. **Data retrieval** – Whenever the application needs any reference data, it calls the appropriate getter. The implementation would:
   - Query the underlying data source (SQL, NoSQL, etc.).
   - Map results to entity objects.
   - Return a collection or map.
3. **Caching** – In production, implementations often cache results to avoid repeated database round‑trips; the interface does not specify this, leaving it to the concrete class.
4. **Cleanup** – Not required; the interface has no resources to close. If the concrete class manages connections, it must handle them internally.

### Assumptions & Constraints

- **Read‑only**: All methods return collections/maps but never modify them. The caller must treat returned collections as read‑only or explicitly clone them if modification is needed.
- **No generics for Map**: `public Map getSupportedCreditCards();` – this loses compile‑time key/value type safety. It assumes a specific map type (likely `Map<String, String>` or `Map<Integer, String>`).
- **Entity immutability**: The contract assumes that the returned entities are either immutable or that callers won’t alter them in a way that affects persistence.  
- **No pagination**: All methods return *all* reference data. For large datasets, this could be a performance issue unless the implementation handles lazy loading or pagination internally.

---

## 3. Functions/Methods

| Method | Signature | Purpose | Input | Output | Side Effects |
|--------|-----------|---------|-------|--------|--------------|
| `getProductTypes()` | `Collection<ProductType> getProductTypes()` | Returns all product type reference objects. | None | `Collection<ProductType>` | None |
| `getSupportedCreditCards()` | `Map getSupportedCreditCards()` | Returns a map of supported credit card codes to descriptions. | None | `Map` (unspecified key/value types) | None |
| `getZones()` | `Collection<Zone> getZones()` | Retrieves all geographical zones. | None | `Collection<Zone>` | None |
| `getLanguages()` | `Collection<Language> getLanguages()` | Returns all supported languages. | None | `Collection<Language>` | None |
| `getMeasureUnits()` | `Collection<CentralMeasureUnits> getMeasureUnits()` | Retrieves measurement unit reference data. | None | `Collection<CentralMeasureUnits>` | None |
| `getCurrencies()` | `Collection<Currency> getCurrencies()` | Returns all supported currencies. | None | `Collection<Currency>` | None |
| `getOrderStatus()` | `Collection<OrderStatus> getOrderStatus()` | Returns all possible order statuses. | None | `Collection<OrderStatus>` | None |
| `getCountryStatus()` | `Collection<CentralCountryStatus> getCountryStatus()` | Retrieves country status reference data. | None | `Collection<CentralCountryStatus>` | None |

### Reusable/Utility Methods
The interface itself contains only data retrieval methods; no reusable utilities are present. However, a concrete implementation might expose helper methods (e.g., `findProductTypeByCode`) that could be used across services.

---

## 4. Dependencies

| Dependency | Category | Notes |
|------------|----------|-------|
| `java.util.Collection` | Standard Java | Basic collection interface. |
| `java.util.Map` | Standard Java | Generic map; type parameters omitted. |
| Domain entities (`ProductType`, `Currency`, `Language`, etc.) | Application | Plain Java objects (POJOs) defined in `com.salesmanager.core.entity.reference`. |
| `OrderStatus` | Domain | Reference to an order status entity. |

No external libraries or frameworks are declared. The interface expects that the consuming implementation will handle all underlying persistence concerns (JDBC, JPA, Hibernate, etc.).

---

## 5. Additional Notes

### Strengths
- **Simplicity & clarity**: The interface cleanly separates the data‑access layer from business logic.
- **Extensibility**: Adding new reference data only requires a new method; existing code is unaffected.
- **Decoupling**: Clients depend only on the interface, not on specific implementations.

### Weaknesses & Edge Cases
- **Type safety for the credit‑card map** – the raw `Map` makes callers vulnerable to `ClassCastException`. Prefer `Map<String, String>` or a typed DTO.
- **Potential performance issues** – retrieving all records in one call may be inefficient for large tables (e.g., all currencies if many exist). Pagination or lazy loading might be needed.
- **Immutability concerns** – If returned collections are modified by callers, the underlying data store may be unintentionally changed or the state may become inconsistent. It’s advisable to return unmodifiable wrappers or copies.
- **Error handling** – The interface doesn’t declare any checked exceptions. Implementations must decide whether to wrap checked exceptions into unchecked ones or to document runtime failures.

### Future Enhancements
1. **Typed Maps** – Replace `Map` with `Map<String, String>` or a dedicated DTO for supported credit cards.
2. **Pagination Support** – Add overloaded methods or query parameters for large datasets.
3. **Caching Annotations** – In an implementation layer, use annotations (e.g., Spring’s `@Cacheable`) to memoize results.
4. **Versioning** – If reference data changes over time, consider a versioning mechanism or effective‑date filters.
5. **Generic Interface** – Create a generic `IReferenceDao<T>` to reduce boilerplate for similar retrieval methods.

Overall, the interface is well‑structured for a typical enterprise Java application, but careful attention to type safety, performance, and immutability will help maintain robustness as the codebase evolves.

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
import java.util.Map;

import com.salesmanager.core.entity.orders.OrderStatus;
import com.salesmanager.core.entity.reference.CentralCountryStatus;
import com.salesmanager.core.entity.reference.CentralMeasureUnits;
import com.salesmanager.core.entity.reference.Currency;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.entity.reference.ProductType;
import com.salesmanager.core.entity.reference.Zone;

public interface IGlobalReferenceDao {

	public Collection<ProductType> getProductTypes();

	public Map getSupportedCreditCards();

	public Collection<Zone> getZones();

	public Collection<Language> getLanguages();

	public Collection<CentralMeasureUnits> getMeasureUnits();

	public Collection<Currency> getCurrencies();

	public Collection<OrderStatus> getOrderStatus();

	public Collection<CentralCountryStatus> getCountryStatus();

}


```
