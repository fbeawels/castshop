# Customer.java

## Review

## 1. Summary  
The file defines a plain‑old Java object (POJO) that represents a **Customer** entity used in a web‑service layer.  
Key aspects:

| Field | Purpose |
|-------|---------|
| `customerFirstname`, `customerLastname` | Basic personal info |
| `customerEmailAddress`, `customerTelephone` | Contact details |
| `customerLang` | Two‑letter locale identifier (e.g., “en”, “fr”) |
| `customerStreetAddress`, `customerPostalCode`, `customerCity` | Address details |
| `customerZoneId`, `zoneName` | Geographic zone – `customerZoneId` overrides `zoneName` when > 0 |
| `customerCountryId` | Country reference |
| `customerId` | Primary key (0 = new customer, non‑zero = existing) |

The class is essentially a data transfer object (DTO) with getters and setters for every field. No business logic is present.  

No external frameworks are referenced – it uses only core Java (`java.lang.String`). The design follows the classic JavaBean pattern, which makes it trivially serializable by many web‑service stacks (JAX‑WS, REST frameworks, etc.).

---

## 2. Detailed Description  
### Core components  
1. **Private fields** – hold raw customer data.  
2. **Public getters/setters** – expose each property.  
3. **Default values** –  
   * `customerZoneId` initialized to `0` to signal “undefined”.  
   * `customerId` defaults to `0` for new customers.  

### Execution Flow  
* **Creation** – The client constructs an instance (e.g., `new Customer()`) and populates fields via setters.  
* **Serialization** – The web‑service framework marshals the bean to XML/JSON using the public getters.  
* **Deserialization** – Incoming payloads are turned into a `Customer` by the framework via setters.  
* **Persistence** – The application layer typically converts this DTO to an entity mapped to the database.  
* **Cleanup** – No special cleanup needed; the GC handles the object once it goes out of scope.

### Assumptions & Constraints  
* **Zone logic** – When `customerZoneId > 0`, `zoneName` is ignored. This contract must be documented for callers.  
* **Locale validation** – `customerLang` is expected to be a two‑character ISO code, but no validation is performed.  
* **Mandatory fields** – The comment says `customerZoneId` and `customerCountryId` are required, but the class itself does not enforce this. Validation must be done elsewhere.  
* **Thread safety** – The bean is mutable and not thread‑safe; however, web‑service DTOs are normally used per request, so this is acceptable.

---

## 3. Functions/Methods  
| Method | Description | Parameters | Return | Side‑effects |
|--------|-------------|------------|--------|--------------|
| `getCustomerFirstname()` | Getter for first name | – | `String` | None |
| `setCustomerFirstname(String)` | Setter for first name | `customerFirstname` | void | Updates field |
| `getCustomerLastname()` | Getter for last name | – | `String` | None |
| `setCustomerLastname(String)` | Setter for last name | `customerLastname` | void | Updates field |
| `getCustomerEmailAddress()` | Getter for email | – | `String` | None |
| `setCustomerEmailAddress(String)` | Setter for email | `customerEmailAddress` | void | Updates field |
| `getCustomerTelephone()` | Getter for phone | – | `String` | None |
| `setCustomerTelephone(String)` | Setter for phone | `customerTelephone` | void | Updates field |
| `getCustomerLang()` | Getter for locale | – | `String` | None |
| `setCustomerLang(String)` | Setter for locale | `customerLang` | void | Updates field |
| `getCustomerStreetAddress()` | Getter for street | – | `String` | None |
| `setCustomerStreetAddress(String)` | Setter for street | `customerStreetAddress` | void | Updates field |
| `getCustomerPostalCode()` | Getter for postal code | – | `String` | None |
| `setCustomerPostalCode(String)` | Setter for postal code | `customerPostalCode` | void | Updates field |
| `getCustomerCity()` | Getter for city | – | `String` | None |
| `setCustomerCity(String)` | Setter for city | `customerCity` | void | Updates field |
| `getCustomerZoneId()` | Getter for zone id | – | `int` | None |
| `setCustomerZoneId(int)` | Setter for zone id | `customerZoneId` | void | Updates field |
| `getCustomerCountryId()` | Getter for country id | – | `int` | None |
| `setCustomerCountryId(int)` | Setter for country id | `customerCountryId` | void | Updates field |
| `getCustomerId()` | Getter for primary key | – | `long` | None |
| `setCustomerId(long)` | Setter for primary key | `customerId` | void | Updates field |
| `getZoneName()` | Getter for zone name | – | `String` | None |
| `setZoneName(String)` | Setter for zone name | `zoneName` | void | Updates field |

All methods are straightforward; none perform business logic or validation.

---

## 4. Dependencies  
* **Java Standard Library** – only `java.lang` types (`String`).  
* No annotations, no JPA, no validation frameworks are used.  
* Platform‑independent – works on any JVM compliant with Java 8+ (the license dates suggest Java 6+, but the code itself is compatible with later versions).

---

## 5. Additional Notes  

### Strengths  
* **Simplicity** – easy to understand and maintain.  
* **Framework‑friendly** – compliant with JavaBeans spec, making it serializable by JAXB, Jackson, etc.  
* **Clear intent** – comments describe the required fields and the zone logic.

### Potential Issues & Edge Cases  
1. **Missing Validation** – No checks for null/empty strings, email format, phone format, or locale codes. If the service accepts arbitrary data, downstream processes may fail.  
2. **Zone Logic Confusion** – The contract that `zoneName` is ignored when `customerZoneId > 0` is only in comments. A more robust design could use a dedicated `Zone` object or an enum.  
3. **Immutability** – The bean is fully mutable; accidental modifications can occur if the object is shared. Consider returning copies or making the class immutable if thread‑safety or security is a concern.  
4. **Serialization Anomalies** – Since `zoneName` is not used when `customerZoneId > 0`, it may still be serialized by frameworks, potentially confusing consumers. Mark it `transient` or use annotations to control serialization.  
5. **Documentation** – The class comment lists “requires a valid customerZoneId / customerCountryId”, but the implementation does not enforce this. A separate validation layer is needed, or you could add a constructor that accepts required fields.  

### Future Enhancements  
* **Validation Annotations** – Integrate Bean Validation (`javax.validation.constraints`) to enforce non‑null, size, email, and custom locale constraints.  
* **Builder Pattern** – Provide a fluent builder for constructing instances more safely.  
* **Immutability** – Replace setters with constructor parameters or use Lombok’s `@Value`.  
* **Encapsulation of Zone** – Replace `customerZoneId` and `zoneName` with a dedicated `Zone` value object that enforces the invariant.  
* **Unit Tests** – Add tests covering default values, setter behavior, and serialization.  

Overall, the class is a clean, minimal DTO suitable for a web service, but adding validation and considering immutability would improve robustness and reduce the risk of misuse.

## Code Critique



## Code Preview

```java
/*
 * Licensed to csti consulting 
 * You may obtain a copy of the License at
 *
 * http://www.csticonsulting.com
 * Copyright (c) 2006-3 Sep, 2010 Consultation CS-TI inc. 
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */
package com.salesmanager.core.entity.customer.ws;



/**
 * Customer entity used in the web service to create a Customer
 * requires a valid customerZoneId
 * requires a valid customerCountryId
 * localeStr is 'en', 'fr' ... 2 char language char
 * @author Carl Samson
 *
 */
public class Customer {


	private java.lang.String customerFirstname;
	private java.lang.String customerLastname;
	private java.lang.String customerEmailAddress;
	private java.lang.String customerTelephone;
	private java.lang.String customerLang;

	private String customerStreetAddress;
	private String customerPostalCode;
	private String customerCity;
	private int customerZoneId = 0;//0 means the zone is undefined, then will require zoneName
	private int customerCountryId;
	private String zoneName;//if customerZoneId > 0, it will ignore this field
	

	private long customerId;//leave it to 0 for creation or set it to customerId for edition

	

	public java.lang.String getCustomerFirstname() {
		return customerFirstname;
	}
	public void setCustomerFirstname(java.lang.String customerFirstname) {
		this.customerFirstname = customerFirstname;
	}
	public java.lang.String getCustomerLastname() {
		return customerLastname;
	}
	public void setCustomerLastname(java.lang.String customerLastname) {
		this.customerLastname = customerLastname;
	}
	public java.lang.String getCustomerEmailAddress() {
		return customerEmailAddress;
	}
	public void setCustomerEmailAddress(java.lang.String customerEmailAddress) {
		this.customerEmailAddress = customerEmailAddress;
	}
	public java.lang.String getCustomerTelephone() {
		return customerTelephone;
	}
	public void setCustomerTelephone(java.lang.String customerTelephone) {
		this.customerTelephone = customerTelephone;
	}
	public java.lang.String getCustomerLang() {
		return customerLang;
	}
	public void setCustomerLang(java.lang.String customerLang) {
		this.customerLang = customerLang;
	}
	public String getCustomerStreetAddress() {
		return customerStreetAddress;
	}
	public void setCustomerStreetAddress(String customerStreetAddress) {
		this.customerStreetAddress = customerStreetAddress;
	}
	public String getCustomerPostalCode() {
		return customerPostalCode;
	}
	public void setCustomerPostalCode(String customerPostalCode) {
		this.customerPostalCode = customerPostalCode;
	}
	public String getCustomerCity() {
		return customerCity;
	}
	public void setCustomerCity(String customerCity) {
		this.customerCity = customerCity;
	}
	public int getCustomerZoneId() {
		return customerZoneId;
	}
	public void setCustomerZoneId(int customerZoneId) {
		this.customerZoneId = customerZoneId;
	}
	public int getCustomerCountryId() {
		return customerCountryId;
	}
	public void setCustomerCountryId(int customerCountryId) {
		this.customerCountryId = customerCountryId;
	}

	public long getCustomerId() {
		return customerId;
	}
	public void setCustomerId(long customerId) {
		this.customerId = customerId;
	}
	public String getZoneName() {
		return zoneName;
	}
	public void setZoneName(String zoneName) {
		this.zoneName = zoneName;
	}

	
	
}



```
