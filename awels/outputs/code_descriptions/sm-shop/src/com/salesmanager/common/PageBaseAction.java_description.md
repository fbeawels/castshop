# PageBaseAction.java

## Review

## 1. Summary

`PageBaseAction` is a helper class that provides common pagination logic for web actions (most likely Struts/Spring MVC actions).  
It maintains state for the current page, total item count, page size and the indices of the first/last item displayed on the current page. The class extends `SalesManagerBaseAction`, inheriting any shared behaviour needed by all actions in the application.

**Key components**

| Class | Purpose |
|-------|---------|
| `PageBaseAction` | Encapsulates pagination calculations and exposes getters/setters for paging parameters. |

The class does not rely on any third‑party libraries beyond whatever `SalesManagerBaseAction` provides, so it is effectively pure Java.

---

## 2. Detailed Description

### Core data members

| Field | Default | Role |
|-------|---------|------|
| `pageStartIndex` | 0 | Index of the first row of the *current* page (used for offset calculation). |
| `pageCriteriaIndex` | 0 | Calculated offset used in database queries (`pageStartIndex * size`). |
| `listingCount` | 0 | Total number of items in the list (across all pages). |
| `realCount` | 0 | Number of items actually returned for the current page (may be < `size`). |
| `size` | 20 | Page size – number of items per page. |
| `firstItem` | 1 | 1‑based index of the first item shown on the current page. |
| `lastItem` | 0 | 1‑based index of the last item shown on the current page. |

### Execution flow

1. **Initialization** – The constructor (implicit) sets defaults.  
2. **Page navigation** – When a request comes in, a subclass typically sets `pageStartIndex` (zero‑based) and `realCount` (query result size).  
3. **Offset calculation** – `setPageStartNumber()` converts `pageStartIndex` into a database offset (`pageCriteriaIndex`).  
4. **Index calculation** – `setPageElements()` computes `firstItem` and `lastItem` for display.  
5. **Rendering** – Getters expose the calculated values to the view layer (e.g., JSP/Thymeleaf).

### Design decisions & assumptions

- **Zero‑based vs one‑based** – Internally the class uses zero‑based page indices (`pageStartIndex`) but exposes one‑based item numbers (`firstItem`, `lastItem`).  
- **No validation** – The class assumes callers provide sensible values; negative indices or sizes are silently accepted.  
- **Separation of concerns** – Pagination logic is isolated from action logic, allowing reuse across different actions.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side effects |
|--------|---------|------------|---------|---------------|
| `getPageStartIndex()` | Retrieve the current page offset (zero‑based). | – | `int` | None |
| `setPageStartIndex(int)` | Set the page offset. | `pageStartIndex` | – | Updates field |
| `getListingCount()` | Total number of items available. | – | `int` | None |
| `setListingCount(int)` | Define total item count. | `listingCount` | – | Updates field |
| `getSize()` | Page size. | – | `int` | None |
| `setSize(int)` | Change page size. | `size` | – | Updates field |
| `getFirstItem()` | 1‑based index of the first item on the current page. | – | `int` | None |
| `setFirstItem(int)` | Explicitly set `firstItem`. | `firstItem` | – | Updates field |
| `getLastItem()` | 1‑based index of the last item on the current page. | – | `int` | None |
| `setLastItem(int)` | Explicitly set `lastItem`. | `lastItem` | – | Updates field |
| `setPageStartNumber()` | Computes `pageCriteriaIndex` (`pageStartIndex * size`). | – | – | Updates `pageCriteriaIndex` |
| `setPageElements()` | Calculates `firstItem` and `lastItem` based on current state. | – | – | Updates `firstItem`, `lastItem` |
| `getRealCount()` | Number of items actually retrieved for the current page. | – | `int` | None |
| `setRealCount(int)` | Set the actual count returned by the query. | `realCount` | – | Updates field |
| `getPageCriteriaIndex()` | Offset to be used in database queries. | – | `int` | None |
| `setPageCriteriaIndex(int)` | Explicitly set offset. | `pageCriteriaIndex` | – | Updates field |

**Reusable / utility methods**  
`setPageStartNumber()` and `setPageElements()` are the core utilities. They can be reused in any action that extends this base class.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `SalesManagerBaseAction` | Base class | Not part of the code snippet; likely contains common action utilities (e.g., session handling). |
| None else | Standard Java | All fields and methods use only JDK types (`int`). |

No external libraries or framework-specific APIs are referenced directly, implying the class can be dropped into any Java web framework with minimal changes.

---

## 5. Additional Notes

### Strengths
- **Encapsulation** – Keeps pagination logic isolated, reducing duplication across actions.
- **Simplicity** – Uses only primitive types; no complex frameworks.

### Issues / Edge cases
1. **Redundant conditional**  
   ```java
   if (getListingCount() == 0) {
       if (this.getListingCount() == 0) {
           firstItem = 0;
       }
       ...
   }
   ```
   The inner `if` is unnecessary and should be removed.

2. **Lack of validation**  
   - Negative `pageStartIndex`, `size`, or `listingCount` will produce incorrect offsets or indices.  
   - `realCount` greater than `size` or negative should be guarded against.

3. **Potential overflow**  
   `pageCriteriaIndex = pageStartIndex * size;` can overflow for large values. Consider using `long` or adding a bounds check.

4. **Out‑of‑range `lastItem`**  
   The calculation `pageCriteriaIndex + realCount` might exceed `listingCount`. Typically, `lastItem` should be `min(pageCriteriaIndex + realCount, listingCount)`.

5. **Inconsistent 0/1‑based handling**  
   The class mixes 0‑based page indices with 1‑based item indices. While this is intentional, the API should document this clearly to avoid confusion.

6. **No thread safety**  
   As an action class in a web container, each request usually gets a separate instance, so this is acceptable. If shared, synchronization would be required.

### Suggested Enhancements
- **Validation helper** – Add private `validate()` to ensure all counters are non‑negative and `size > 0`.
- **Refactor `setPageElements()`** – Use clearer logic:
  ```java
  firstItem = (pageStartIndex * size) + 1;
  lastItem = Math.min(firstItem + realCount - 1, listingCount);
  ```
  and handle the case where `listingCount == 0` separately.
- **Rename fields** – `pageStartIndex` → `pageNumber` (1‑based) or `offset` for clarity.
- **Add Javadoc** – Document each method’s contract, especially the 0/1‑based semantics.
- **Unit tests** – Write tests for boundary conditions (first page, last page, empty list, oversize requests).

Overall, `PageBaseAction` provides a solid foundation for pagination but would benefit from a few clean‑ups and safety checks to make it more robust and maintainable.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-Aug 25, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.common;

public class PageBaseAction extends SalesManagerBaseAction {

	private int pageStartIndex = 0;// page number -- pagination
	private int pageCriteriaIndex = 0;// for criteria
	private int listingCount = 0;// total number of items
	private int realCount = 0;// total number in the current page
	private int size = 20;// default
	private int firstItem = 1;
	private int lastItem = 0;

	public int getPageStartIndex() {
		return pageStartIndex;
	}

	public void setPageStartIndex(int pageStartIndex) {
		this.pageStartIndex = pageStartIndex;
	}

	public int getListingCount() {
		return listingCount;
	}

	public void setListingCount(int listingCount) {
		this.listingCount = listingCount;
	}

	public int getSize() {
		return size;
	}

	public void setSize(int size) {
		this.size = size;
	}

	public int getFirstItem() {
		return firstItem;
	}

	public void setFirstItem(int firstItem) {
		this.firstItem = firstItem;
	}

	public int getLastItem() {
		return lastItem;
	}

	public void setLastItem(int lastItem) {
		this.lastItem = lastItem;
	}

	protected void setPageStartNumber() {

		int start = this.getPageStartIndex();
		if (this.getPageStartIndex() == 0) {
			start = 0;
		} else {
			start = start * this.getSize();
		}
		this.setPageCriteriaIndex(start);
	}

	protected void setPageElements() {

		if (getListingCount() == 0) {
			if (this.getListingCount() == 0) {
				firstItem = 0;
			}
			this.setFirstItem(firstItem);

			this.setLastItem(listingCount);
		} else {
			if (this.getPageStartIndex() == 0) {
				this.setFirstItem(firstItem);
			} else {

				this.setFirstItem(this.getPageCriteriaIndex() + 1);

			}
			this.setLastItem(this.getPageCriteriaIndex() + this.getRealCount());

		}
	}

	public int getRealCount() {
		return realCount;
	}

	public void setRealCount(int realCount) {
		this.realCount = realCount;
	}

	public int getPageCriteriaIndex() {
		return pageCriteriaIndex;
	}

	public void setPageCriteriaIndex(int pageCriteriaIndex) {
		this.pageCriteriaIndex = pageCriteriaIndex;
	}

}



```
