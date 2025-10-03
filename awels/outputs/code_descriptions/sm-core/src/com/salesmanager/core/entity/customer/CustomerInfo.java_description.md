# CustomerInfo.java

## Review

## 1. Summary
The **`CustomerInfo`** class is a plain‑old Java object (POJO) that represents a customer’s meta‑information in the *SalesManager* application.  
- It stores audit data such as the last login time, the number of logins, account creation and modification dates, and a flag for global product notifications.  
- The class is **serializable** and is meant to be mapped to a database table via an ORM tool (the header comments reference *Hibernate Tools*).  
- It provides standard getters/setters and a few convenience methods that format dates into `String` values using a custom `DateUtil` helper.

### Design patterns / frameworks
- **Entity/Domain Object** pattern (Hibernate/JPA).  
- No explicit design patterns beyond the standard JavaBeans convention.

---

## 2. Detailed Description
1. **Fields**  
   ```java
   private long customerInfoId;
   private Date customerInfoDateOfLastLogon;
   private Integer customerInfoNumberOfLogon;
   private Date customerInfoDateAccountCreated;
   private Date customerInfoDateAccountLastModified;
   private Integer globalProductNotifications;
   ```

2. **Initialisation**  
   The private `init()` method sets all date fields to the current time and numeric fields to zero.  
   The default constructor calls `init()`; a minimal constructor accepts only the primary key; the full constructor attempts to initialise every field.

3. **Accessors**  
   Standard getter/setter pairs are provided for each field.  
   Two *derived* getters (`getCustomerInfoDateOfLastLogonText`, `getCustomerInfoDateAccountLastModifiedText`) format the corresponding `Date` into a readable string via `DateUtil.formatDate()`.

4. **Execution Flow**  
   - When a new `CustomerInfo` is instantiated with the default constructor, all timestamps are set to the current moment and counters to zero.  
   - When persisted, Hibernate will map the fields to the database (assumed via annotations or XML mapping).  
   - The class itself does not manage cleanup; it relies on the JVM’s garbage collection.

5. **Assumptions & Constraints**  
   - `DateUtil.formatDate(Date)` is assumed to return a non‑null string or gracefully handle `null`.  
   - The class expects to be used in a *Hibernate* context where field names correspond to column names.  
   - No validation logic is present; negative values for `customerInfoNumberOfLogon` or `globalProductNotifications` could be set without error.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Return | Notes |
|--------|---------|------------|--------|-------|
| `private void init()` | Initializes all fields to defaults. | None | void | Called by default constructor. |
| `public CustomerInfo()` | Default constructor. | None | `CustomerInfo` | Calls `init()`. |
| `public CustomerInfo(long customerInfoId)` | Minimal constructor. | `customerInfoId` | `CustomerInfo` | Does **not** initialise other fields. |
| `public CustomerInfo(int customerInfoId, Date customerInfoDateOfLastLogon, Integer customerInfoNumberOfLogons, Date customerInfoDateAccountCreated, Date customerInfoDateAccountLastModified, Integer globalProductNotifications)` | Full constructor. | *All* fields | `CustomerInfo` | **BUG** – type mismatch (`int` → `long`) and misspelled param (`customerInfoNumberOfLogons` vs `customerInfoNumberOfLogon`). |
| `public long getCustomerInfoId()` | Getter for ID. | None | `long` | |
| `public void setCustomerInfoId(long customerInfoId)` | Setter for ID. | `long` | void | |
| `public Date getCustomerInfoDateOfLastLogon()` | Getter for last logon date. | None | `Date` | |
| `public String getCustomerInfoDateOfLastLogonText()` | Formatted string of last logon date. | None | `String` | Delegates to `DateUtil.formatDate`. |
| `public void setCustomerInfoDateOfLastLogon(Date)` | Setter. | `Date` | void | |
| `public Integer getCustomerInfoNumberOfLogon()` | Getter for number of logins. | None | `Integer` | |
| `public void setCustomerInfoNumberOfLogon(Integer)` | Setter. | `Integer` | void | |
| `public Date getCustomerInfoDateAccountCreated()` | Getter. | None | `Date` | |
| `public void setCustomerInfoDateAccountCreated(Date)` | Setter. | `Date` | void | |
| `public Date getCustomerInfoDateAccountLastModified()` | Getter. | None | `Date` | |
| `public String getCustomerInfoDateAccountLastModifiedText()` | Formatted string of last modified date. | None | `String` | |
| `public void setCustomerInfoDateAccountLastModified(Date)` | Setter. | `Date` | void | |
| `public Integer getGlobalProductNotifications()` | Getter for notifications flag. | None | `Integer` | |
| `public void setGlobalProductNotifications(Integer)` | Setter. | `Integer` | void | |

**Reusable/utility methods**  
- The two `*Text()` methods are lightweight wrappers around `DateUtil.formatDate`, useful wherever a human‑readable date is needed.

---

## 4. Dependencies
| Dependency | Type | Purpose |
|------------|------|---------|
| `java.io.Serializable` | Standard | Enables the entity to be serialized (e.g., for caching or remote calls). |
| `java.util.Date` | Standard | Stores timestamps. |
| `com.salesmanager.core.util.DateUtil` | Third‑party (project‑specific) | Provides date formatting. |
| Hibernate (implied by comments) | Third‑party | ORM mapping of this entity to a relational table. |

*No external libraries beyond the Java SE platform and the project’s own utilities are referenced.*

---

## 5. Additional Notes

### 5.1 Compile‑time Issues
- **Full constructor**:  
  - The `customerInfoId` parameter is declared as `int` while the field is `long`. This will compile but may truncate values.  
  - The `customerInfoNumberOfLogons` parameter is unused; the code assigns `customerInfoNumberOfLogon` (without the trailing `s`). This results in a *compiler error* (undefined variable).  
  - Suggested fix: change the constructor signature to use `long` and `Integer` for all fields, and assign each field correctly.

### 5.2 API Design Improvements
- **Use `java.time`** (e.g., `Instant`, `LocalDateTime`) instead of `java.util.Date`. The newer API is immutable, thread‑safe, and offers better formatting utilities.
- **Avoid primitive wrappers** for counters unless `null` semantics are required. Prefer `int` and `boolean` where appropriate.
- **Add validation** in setters (e.g., non‑negative logon counts).
- **Document** the meaning of `globalProductNotifications` (true/false, count, etc.).
- **Consider immutability** for audit fields, updating them via dedicated service methods rather than public setters.

### 5.3 Edge Cases
- **Null dates**: If a `Date` field is `null`, `DateUtil.formatDate` might throw a `NullPointerException`. Guard against this or document that the field must never be null.
- **Thread‑safety**: As a mutable POJO, concurrent access to a single instance can lead to race conditions. In a typical JPA/Hibernate context, instances are not shared across threads, but this assumption should be documented.

### 5.4 Future Enhancements
- **JPA annotations** (`@Entity`, `@Table`, `@Column`, `@Id`) for clarity and to remove reliance on external XML mapping.  
- **Audit listeners** to automatically update `customerInfoDateAccountLastModified` on every persistence event.  
- **Domain‑specific methods** such as `incrementLoginCount()` or `hasEnabledGlobalNotifications()` to encapsulate business logic.  
- **Unit tests** covering serialization, formatting, and constructor behaviour.

--- 

**Overall assessment**: The class provides the minimal skeleton for a customer metadata entity but suffers from several compilation errors and design shortcomings. Addressing the constructor bug and modernizing the date handling would greatly improve reliability and maintainability.

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
package com.salesmanager.core.entity.customer;

// Generated Mar 8, 2009 10:16:39 PM by Hibernate Tools 3.2.0.beta8

import java.util.Date;

import com.salesmanager.core.util.DateUtil;

/**
 * CustomersInfo generated by hbm2java
 */
public class CustomerInfo implements java.io.Serializable {

	// Fields

	private long customerInfoId;

	private Date customerInfoDateOfLastLogon;

	private Integer customerInfoNumberOfLogon;

	private Date customerInfoDateAccountCreated;

	private Date customerInfoDateAccountLastModified;

	private Integer globalProductNotifications;

	private void init() {
		customerInfoDateOfLastLogon = new Date();
		customerInfoDateAccountCreated = new Date();
		customerInfoDateAccountLastModified = new Date();
		globalProductNotifications = new Integer(0);
		customerInfoNumberOfLogon = new Integer(0);
	}

	// Constructors

	/** default constructor */
	public CustomerInfo() {
		init();
	}

	/** minimal constructor */
	public CustomerInfo(long customerInfoId) {
		this.customerInfoId = customerInfoId;
	}

	/** full constructor */
	public CustomerInfo(int customerInfoId, Date customerInfoDateOfLastLogon,
			Integer customerInfoNumberOfLogons,
			Date customerInfoDateAccountCreated,
			Date customerInfoDateAccountLastModified,
			Integer globalProductNotifications) {
		this.customerInfoId = customerInfoId;
		this.customerInfoDateOfLastLogon = customerInfoDateOfLastLogon;
		this.customerInfoNumberOfLogon = customerInfoNumberOfLogon;
		this.customerInfoDateAccountCreated = customerInfoDateAccountCreated;
		this.customerInfoDateAccountLastModified = customerInfoDateAccountLastModified;
		this.globalProductNotifications = globalProductNotifications;
	}

	// Property accessors
	public long getCustomerInfoId() {
		return this.customerInfoId;
	}

	public void setCustomerInfoId(long customerInfoId) {
		this.customerInfoId = customerInfoId;
	}

	public Date getCustomerInfoDateOfLastLogon() {
		return this.customerInfoDateOfLastLogon;
	}

	public String getCustomerInfoDateOfLastLogonText() {

		return DateUtil.formatDate(this.customerInfoDateOfLastLogon);

	}

	public void setCustomerInfoDateOfLastLogon(Date customerInfoDateOfLastLogon) {
		this.customerInfoDateOfLastLogon = customerInfoDateOfLastLogon;
	}

	public Integer getCustomerInfoNumberOfLogon() {
		return this.customerInfoNumberOfLogon;
	}

	public void setCustomerInfoNumberOfLogon(Integer customerInfoNumberOfLogon) {
		this.customerInfoNumberOfLogon = customerInfoNumberOfLogon;
	}

	public Date getCustomerInfoDateAccountCreated() {
		return this.customerInfoDateAccountCreated;
	}

	public void setCustomerInfoDateAccountCreated(
			Date customerInfoDateAccountCreated) {
		this.customerInfoDateAccountCreated = customerInfoDateAccountCreated;
	}

	public Date getCustomerInfoDateAccountLastModified() {
		return this.customerInfoDateAccountLastModified;
	}

	public String getCustomerInfoDateAccountLastModifiedText() {

		return DateUtil.formatDate(this.customerInfoDateAccountLastModified);

	}

	public void setCustomerInfoDateAccountLastModified(
			Date customerInfoDateAccountLastModified) {
		this.customerInfoDateAccountLastModified = customerInfoDateAccountLastModified;
	}

	public Integer getGlobalProductNotifications() {
		return this.globalProductNotifications;
	}

	public void setGlobalProductNotifications(Integer globalProductNotifications) {
		this.globalProductNotifications = globalProductNotifications;
	}

}



```
