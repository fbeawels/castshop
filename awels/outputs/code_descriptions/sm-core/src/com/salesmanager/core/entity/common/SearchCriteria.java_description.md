# SearchCriteria.java

## Review

## 1. Summary
`SearchCriteria` is a small Java POJO that encapsulates paging and filtering information for a query.  
It holds the following data:

| Field | Default | Meaning |
|-------|---------|---------|
| `quantity` | 20 | Number of items to return per page |
| `startindex` | 0 | Zero‑based page number (i.e. 0 → first page) |
| `merchantId` | 1 | Identifier of the merchant to filter by |
| `languageId` | 1 | Identifier of the language to filter by |

The class also exposes two convenience methods – `getLowerLimit()` and `getUpperLimit(int count)` – that compute the numeric offset and the size of the slice to fetch from a full result set. No external libraries or frameworks are used; it is pure Java.

## 2. Detailed Description
### Core responsibilities
1. **Paging calculations** – Convert the logical page (`startindex`) and page size (`quantity`) into a SQL‑style offset (`lowerLimit`) and a row count (`upperLimit`).
2. **Filtering** – Store the `merchantId` and `languageId` values so that calling code can add them to a query’s `WHERE` clause.

### Flow of execution
- An instance is created (e.g., `SearchCriteria criteria = new SearchCriteria();`).
- The caller can override defaults by invoking the setters.
- When executing a query:
  1. `getLowerLimit()` is called to obtain the offset.
  2. `getUpperLimit(totalCount)` is called to determine how many rows to fetch, respecting the remaining rows if the last page is incomplete.
- The resulting limits are typically used in the `LIMIT`/`OFFSET` clause of a SQL statement or in a JPA `setMaxResults/setFirstResult` call.

### Assumptions & constraints
- `startindex` is assumed to be non‑negative.
- `quantity` is assumed to be > 0; a value of 0 would lead to division by zero or no rows returned.
- The class does **not** enforce any bounds on `merchantId` or `languageId`; validation must be performed elsewhere.
- The calculation in `getUpperLimit` presumes that `count` is the total number of records available.

### Architecture & design choices
- Simple value object – no business logic beyond paging calculations.
- No immutability: fields can be changed after construction, which gives flexibility but can introduce bugs if the object is shared.
- Method names use camelCase but some are somewhat non‑idiomatic (`startindex` instead of `startIndex`). Consistency with Java naming conventions would improve readability.

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `int getLowerLimit()` | Computes offset = `startindex * quantity` | none | offset | none |
| `int getUpperLimit(int count)` | Computes how many rows to fetch on the current page. If fewer than `quantity` rows remain, returns the remaining count. | `count` – total records available | number of rows to fetch | none |
| `int getLanguageId()` | Getter | none | language ID | none |
| `void setLanguageId(int)` | Setter | language ID | none | modifies state |
| `int getMerchantId()` | Getter | none | merchant ID | none |
| `void setMerchantId(int)` | Setter | merchant ID | none | modifies state |
| `int getQuantity()` | Getter | none | page size | none |
| `void setQuantity(int)` | Setter | page size | none | modifies state |
| `int getStartindex()` | Getter | none | page number | none |
| `void setStartindex(int)` | Setter | page number | none | modifies state |

### Reusable/Utility methods
`getLowerLimit()` and `getUpperLimit()` are the only domain‑specific utility methods. They can be reused across any paging logic where the data source exposes a total count.

## 4. Dependencies
- **Standard Java**: Only uses `int` primitives; no external libraries or frameworks.
- No annotations (e.g., Lombok), persistence frameworks, or serialization libraries are referenced.

## 5. Additional Notes
### Edge cases & potential bugs
- **Zero or negative quantity**: `getUpperLimit` would return `quantity` (possibly 0) or incorrect limits. A guard clause or input validation should be added.
- **Negative `startindex`**: Will produce a negative offset. Input validation is advisable.
- **Large values**: Multiplying two `int`s could overflow if both are very large. Switching to `long` for indices and quantities would future‑proof the class.
- **Inconsistent naming**: The field `startindex` does not follow camelCase; rename to `startIndex` for clarity.

### Future enhancements
- **Immutability**: Provide a builder or constructor that accepts all fields, removing setters to make the object thread‑safe.
- **Validation**: Throw `IllegalArgumentException` for invalid values instead of silently accepting them.
- **Pagination interface**: Implement an interface that standardizes paging behaviour, allowing interchangeable paging strategies (e.g., cursor‑based).
- **Documentation**: Add Javadoc comments describing the expected semantics of each method and field.
- **Unit tests**: Write tests covering normal and edge cases (zero quantity, last page, overflow).

Overall, the class serves its purpose as a lightweight paging helper, but it would benefit from stronger input validation, naming consistency, and documentation to make it robust in a production environment.

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
package com.salesmanager.core.entity.common;

public class SearchCriteria {

	private int quantity = 20;
	private int startindex = 0;
	private int merchantId = 1;
	private int languageId = 1;

	public int getLowerLimit() {
		return startindex * quantity;
	}

	public int getUpperLimit(int count) {
		int countLeft = count - startindex * quantity;
		if (countLeft < 0) {
			return quantity;
		}
		if (countLeft < quantity) {
			return countLeft;
		} else {
			return quantity;
		}
	}

	public int getLanguageId() {
		return languageId;
	}

	public void setLanguageId(int languageId) {
		this.languageId = languageId;
	}

	public int getMerchantId() {
		return merchantId;
	}

	public void setMerchantId(int merchantId) {
		this.merchantId = merchantId;
	}

	public int getQuantity() {
		return quantity;
	}

	public void setQuantity(int quantity) {
		this.quantity = quantity;
	}

	public int getStartindex() {
		return startindex;
	}

	public void setStartindex(int startindex) {
		this.startindex = startindex;
	}

}



```
