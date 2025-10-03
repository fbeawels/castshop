# SearchOrdersCriteria.java

## Review

## 1. Summary
`SearchOrdersCriteria` is a simple DTO used to encapsulate filtering parameters for searching orders in the SalesManager system.  
- **Key fields**: date range (`sdate`, `edate`), order identifiers (`orderId`, `customerId`) and customer name (`customerName`).  
- **Utility methods**: `isSet()` determines whether any filter has been applied, `resetCriteria()` clears all filters, and two helper getters (`getStartDateString()`, `getEndDateString()`) format the dates via `DateUtil`.  
- **Design pattern**: The class follows the *Value Object* pattern – it only holds data without any business logic.  
- **Frameworks/Libraries**: It relies on a custom `DateUtil` helper and extends a project‑specific `SearchCriteria` base class.

## 2. Detailed Description
1. **Construction & State**  
   - All fields are public‑private members with getters/setters.  
   - Primitive longs are initialized to `-1` (`orderId`) and `0` (`customerId`) – the `-1` convention signals an unset value for order ID.  
   - Date fields start as `null`.  

2. **Execution Flow**  
   - A consumer creates an instance, sets any combination of fields, and passes it to a DAO/service that interprets the filters.  
   - `isSet()` can be used early to skip the query if no criteria exist.  
   - `resetCriteria()` restores the default “no‑filter” state, useful for re‑using the same object across multiple requests.  

3. **Assumptions & Constraints**  
   - The code assumes `DateUtil.formatDate` can handle `null` dates gracefully.  
   - The DAO must interpret negative or zero IDs as “unset”.  
   - Thread safety is not guaranteed; callers should not share mutable instances across threads without external synchronization.

4. **Architecture & Design Choices**  
   - By extending `SearchCriteria`, the class inherits pagination or sorting fields (not shown), allowing it to plug into a generic search framework.  
   - The use of primitive types (`long`) instead of boxed types (`Long`) improves performance but sacrifices nullability, hence the sentinel `-1`.  

## 3. Functions/Methods
| Method | Purpose | Parameters | Return | Side Effects |
|--------|---------|------------|--------|--------------|
| `getEdate()` | Retrieve the end date filter. | – | `Date` | – |
| `setEdate(Date edate)` | Set the end date filter. | `Date` | – | updates internal state |
| `getSdate()` | Retrieve the start date filter. | – | `Date` | – |
| `setSdate(Date sdate)` | Set the start date filter. | `Date` | – | updates internal state |
| `getCustomerName()` | Get customer name filter. | – | `String` | – |
| `setCustomerName(String customerName)` | Set customer name filter. | `String` | – | updates internal state |
| `getStartDateString()` | Format `sdate` using `DateUtil`. | – | `String` | – |
| `getEndDateString()` | Format `edate` using `DateUtil`. | – | `String` | – |
| `isSet()` | Check if any filter is active. | – | `boolean` | – |
| `resetCriteria()` | Clear all filters to default state. | – | – | resets all fields |
| `getOrderId()` | Retrieve order ID filter. | – | `long` | – |
| `setOrderId(long orderId)` | Set order ID filter. | `long` | – | updates internal state |
| `getCustomerId()` | Retrieve customer ID filter. | – | `long` | – |
| `setCustomerId(long customerId)` | Set customer ID filter. | `long` | – | updates internal state |

*Reusable / Utility Methods*  
- `isSet()` and `resetCriteria()` are generic helpers for any filter object that follows the same sentinel‑value pattern.

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `com.salesmanager.core.entity.common.SearchCriteria` | Project‑specific base class | Provides pagination/sorting; not shown here. |
| `com.salesmanager.core.util.DateUtil` | Project‑specific utility | Handles date formatting; assumed to be thread‑safe. |
| `java.util.Date` | Standard JDK | Legacy date type; modern code prefers `java.time`. |
| No external libraries or APIs are referenced. |

## 5. Additional Notes
### Strengths
- **Simplicity**: Clear, concise DTO that is easy to understand and use.  
- **Reusability**: The `resetCriteria()` method enables object pooling or reuse across requests.  
- **Extensibility**: By extending `SearchCriteria`, the class can be dropped into existing search frameworks.

### Weaknesses & Edge Cases
1. **Date handling**  
   - If `DateUtil.formatDate` does not accept `null`, calling `getStartDateString()` or `getEndDateString()` on a fresh instance will throw a `NullPointerException`.  
   - The class accepts only `Date` instances; it offers no support for time zones or `LocalDate` semantics.

2. **Sentinel values**  
   - Using `-1` for `orderId` and `0` for `customerId` is brittle; any legitimate value that could be negative or zero (unlikely for IDs, but possible in future schema changes) would be misinterpreted.  

3. **Thread safety**  
   - The object is mutable and not synchronized; shared use across threads without external locking can lead to race conditions.

4. **Validation**  
   - No checks are performed to ensure `sdate` ≤ `edate` or that IDs are positive. A consumer must enforce these invariants externally.

5. **Modernization**  
   - Replacing `java.util.Date` with `java.time.LocalDate` or `LocalDateTime` would reduce null‑safety issues and improve API expressiveness.  
   - Consider using `OptionalLong` for IDs if you wish to keep nullability semantics.

### Suggested Enhancements
- **Immutability**: Provide a builder or constructor that accepts all parameters, then make fields `final`.  
- **Validation**: Add simple checks in setters or a `validate()` method to enforce business rules (e.g., start date before end date).  
- **Null‑safe formatting**: Modify `getStartDateString()` / `getEndDateString()` to return an empty string or `Optional<String>` if the date is `null`.  
- **Use of Java 8 Date/Time API**: Transition to `java.time` types, adding conversion helpers if legacy code still uses `Date`.  
- **Unit Tests**: Add tests covering `isSet()`, `resetCriteria()`, and edge cases such as null dates or negative IDs.  

Overall, `SearchOrdersCriteria` is a lightweight, purpose‑fit DTO suitable for its current context but could benefit from modern Java practices and defensive programming to improve robustness and maintainability.

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
package com.salesmanager.core.entity.orders;

import java.util.Date;

import com.salesmanager.core.entity.common.SearchCriteria;
import com.salesmanager.core.util.DateUtil;

public class SearchOrdersCriteria extends SearchCriteria {

	private Date sdate;
	private Date edate;

	private long orderId = -1;
	private long customerId;

	private String customerName;

	public Date getEdate() {
		return edate;
	}

	public void setEdate(Date edate) {
		this.edate = edate;
	}

	public Date getSdate() {
		return sdate;
	}

	public void setSdate(Date sdate) {
		this.sdate = sdate;
	}

	public String getCustomerName() {
		return customerName;
	}

	public void setCustomerName(String customerName) {
		this.customerName = customerName;
	}

	public String getStartDateString() {
		return DateUtil.formatDate(sdate);
	}

	public String getEndDateString() {
		return DateUtil.formatDate(edate);
	}

	public boolean isSet() {
		if (customerName != null || orderId != -1 || sdate != null
				|| edate != null) {
			return true;
		} else {
			return false;
		}
	}

	public void resetCriteria() {
		this.orderId = -1;
		this.sdate = null;
		this.edate = null;
		this.customerName = null;
	}

	public long getOrderId() {
		return orderId;
	}

	public void setOrderId(long orderId) {
		this.orderId = orderId;
	}

	public long getCustomerId() {
		return customerId;
	}

	public void setCustomerId(long customerId) {
		this.customerId = customerId;
	}
}



```
