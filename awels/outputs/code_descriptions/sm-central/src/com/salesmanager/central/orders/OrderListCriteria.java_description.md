# OrderListCriteria.java

## Review

## 1. Summary  
`OrderListCriteria` is a small, plain‑old Java object (POJO) used to hold filtering parameters for an order‑search operation.  
It exposes four optional criteria – start date, end date, order ID and customer name – and provides a helper method (`isSet`) to determine if any of those criteria have been supplied, as well as a convenience `resetCriteria` to clear all values.

The class is deliberately lightweight, relying only on the JDK (`java.util.Date`) and containing no external dependencies or framework annotations.  

## 2. Detailed Description  
The object follows a straightforward bean pattern:

1. **State**  
   - `sdate` / `edate` – `java.util.Date` instances representing the lower/upper bounds of the search window.  
   - `orderid` – a `long` defaulting to `-1` to signal “unset.”  
   - `customerName` – a nullable `String` filter.

2. **Interaction**  
   - Consumers create an instance, set the desired fields via the public setters, and then pass it to whatever service or DAO performs the query.  
   - The service may call `isSet()` to guard against executing a query with no filters (i.e., to avoid returning all orders).  
   - After a query is performed, a consumer may call `resetCriteria()` to reuse the same object for a fresh search.

3. **Assumptions & Constraints**  
   - An order ID of `-1` is treated as “not specified.”  
   - The class does not enforce any ordering relationship between `sdate` and `edate`.  
   - Dates are mutable; the class does not defensively copy them, so callers must avoid mutating the dates after passing them in.

4. **Architecture & Design Choices**  
   - Simplicity was favored over robustness: no validation, no immutability, no builder pattern.  
   - The sentinel value for `orderid` keeps the field primitive and avoids boxing, at the cost of a magic number.

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `Date getSdate()` | Retrieve start date filter | None | `Date` | None |
| `void setSdate(Date)` | Assign start date filter | `Date` | None | Sets `sdate` |
| `Date getEdate()` | Retrieve end date filter | None | `Date` | None |
| `void setEdate(Date)` | Assign end date filter | `Date` | None | Sets `edate` |
| `long getOrderid()` | Retrieve order ID filter | None | `long` | None |
| `void setOrderid(long)` | Assign order ID filter | `long` | None | Sets `orderid` |
| `String getCustomerName()` | Retrieve customer name filter | None | `String` | None |
| `void setCustomerName(String)` | Assign customer name filter | `String` | None | Sets `customerName` |
| `boolean isSet()` | Determine if any filter has been specified | None | `boolean` (`true` if any field is non‑null/`-1`) | None |
| `void resetCriteria()` | Clear all filters to defaults | None | None | Resets all fields to initial state |

### Utility / Reusable Methods  
- None beyond the simple getters/setters. The class is intentionally minimal.

## 4. Dependencies  
| Dependency | Type | Notes |
|------------|------|-------|
| `java.util.Date` | JDK standard | Mutable; no defensive copying performed. |
| No external libraries or frameworks. | | |

The class is fully portable across any Java SE runtime and has no platform‑specific assumptions.

## 5. Additional Notes  

### Edge Cases & Potential Issues  
1. **Date Mutability** – Because `Date` is mutable, callers could inadvertently alter `sdate`/`edate` after setting them, leading to subtle bugs. A defensive copy in getters/setters would mitigate this.  
2. **Order ID Sentinel** – Using `-1` as a magic number can be fragile. If a legitimate order ID could ever be negative, the logic breaks. Switching to `Long` and using `null` would be clearer.  
3. **No Validation** – The class accepts any combination of fields, even nonsensical ones (e.g., start date after end date). If the surrounding service does not validate, this may result in empty or incorrect results.  
4. **Thread Safety** – The object is not thread‑safe. Concurrent modifications could lead to race conditions. If shared across threads, synchronization or immutable copies are required.  

### Suggested Enhancements  
- **Use `java.time` API** – Replace `Date` with `LocalDateTime` or `Instant` for better clarity and immutability.  
- **Builder Pattern** – Provide a fluent builder to construct instances in a readable way.  
- **Validation Logic** – Add a `validate()` method (or incorporate validation in setters) to enforce that `sdate <= edate`.  
- **Equals / HashCode / toString** – Implement these for easier debugging and collection usage.  
- **Immutability** – Consider making the class immutable once constructed, preventing accidental changes after use.  
- **Unit Tests** – Add tests to cover typical usage scenarios, including the reset logic and `isSet` behavior.  

Overall, the class serves its purpose as a simple criteria holder but could benefit from minor defensive programming improvements and modern Java practices.

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
package com.salesmanager.central.orders;

import java.util.Date;

public class OrderListCriteria {

	private Date sdate;
	private Date edate;

	private long orderid = -1;

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

	public long getOrderid() {
		return orderid;
	}

	public void setOrderid(long orderid) {
		this.orderid = orderid;
	}

	public String getCustomerName() {
		return customerName;
	}

	public void setCustomerName(String customerName) {
		this.customerName = customerName;
	}

	public boolean isSet() {
		if (customerName != null || orderid != -1 || sdate != null
				|| edate != null) {
			return true;
		} else {
			return false;
		}
	}

	public void resetCriteria() {
		this.orderid = -1;
		this.sdate = null;
		this.edate = null;
		this.customerName = null;
	}

}



```
