# Customer.java

## Review

## 1. Summary  
The `Customer` class is a plain‑old Java object (POJO) that represents a row in the `customers` table.  
It implements `Serializable` and `IMerchant`, and contains a large number of fields that map directly to the columns in the database.  The class is heavily annotated with Hibernate style comments (but no JPA annotations are actually present), which suggests that mapping is performed through an external XML mapping file.  

Key responsibilities  
* Hold all customer‑related data (personal details, contact information, billing & shipping addresses, merchant association).  
* Provide getters/setters for each column.  
* Provide helper methods that translate raw IDs into human‑readable names via a cache (`RefCache`).  
* Offer utility methods such as `getEmail()`, `getName()`, and various `get…Country/State` helpers.  

Design patterns & frameworks  
* **JavaBeans** – the class follows the standard getter/setter pattern.  
* **Data Transfer Object (DTO)** – the object is used to move data between layers (DAO ↔ service ↔ presentation).  
* **Hibernate mapping** – indicated by the class comment (`@hibernate.class`).  
* **Cache lookup** – `RefCache` is used to fetch zone/country names.  

---

## 2. Detailed Description  
### 2.1 Construction  
* `Customer()` – calls `initialize()` which sets a default state for all fields.  
* `Customer(long id)` – assigns the primary key and then calls `initialize()`.  
The `initialize()` method populates many fields with empty strings or defaults (e.g., `customerAnonymous` is set to `true`).  

### 2.2 Core Data Fields  
The class contains three logical groups:  
1. **Basic identity** – `customerId`, `customerGender`, `customerFirstname`, `customerLastname`, `customerEmailAddress`, etc.  
2. **Contact / shipping** – street, city, state, postal code, country ID/zone ID.  
3. **Billing** – separate set of fields prefixed with `customerBilling…`.  

### 2.3 Address Helpers  
Methods such as `getShippingCountry()`, `getBillingCountry()`, `getShippingSate()` (typo: *state*) and `getBillingState()` translate numeric IDs into localized names by looking up cached `Country` and `Zone` objects.  
These helpers rely on the `Locale` field (which may be `null` until explicitly set) and the static caches.  

### 2.4 Miscellaneous Utilities  
* `getEmail()` strips any anonymous prefix from the email address.  
* `getName()` returns a full name or falls back to the concatenation of first/last names.  
* `hashCode()`/`equals()` compare **all** fields (not just the primary key).  

### 2.5 Clean‑up  
The class does not manage any resources that need explicit cleanup, but it exposes a large number of mutable fields, which is typical for an ORM entity.

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `Customer()` | Default ctor – initializes all fields to defaults. | – | new `Customer` instance | – |
| `Customer(long customerId)` | Initializes with PK. | `customerId` | new `Customer` | – |
| `initialize()` | Helper to set defaults. | – | – | Sets fields |
| `setCustomerXxx()/getCustomerXxx()` | Standard JavaBean accessors for each column. | – | value | – |
| `getEmail()` | Returns the email address stripped of any anonymous prefix. | – | `String` | – |
| `toString()` | Delegates to `Object.toString()` (unhelpful). | – | `String` | – |
| `getName()` | Returns full name; falls back to first+last if `name` field unset. | – | `String` | – |
| `hashCode()` | Computes a hash over all fields. | – | `int` | – |
| `equals(Object)` | Equality based on all fields. | `Object` | `boolean` | – |
| `getShippingSate()` | (Typo) Returns shipping state name, using cache if necessary. | – | `String` | – |
| `getShippingCountry()` | Returns shipping country name via cache. | – | `String` | – |
| `getBillingState()` | Returns billing state name via cache. | – | `String` | – |
| `getBillingCountry()` | Returns billing country name via cache. | – | `String` | – |
| `getLocale()` / `setLocale(Locale)` | Accessor for the locale used for translations. | – / `Locale` | – | – |
| `getCustomerNameId()` | Combines name and id. | – | `String` | – |
| ... | **All other getters/setters** | – | – | – |

**Reusable/Utility methods**  
The class itself has minimal utility logic; most helpers are tightly coupled to the caching infrastructure (`RefCache`, `LanguageUtil`, `LocaleUtil`).  

---

## 4. Dependencies  

| Dependency | Type | Purpose |
|------------|------|---------|
| `org.apache.commons.lang.StringUtils` | Third‑party | String utilities (e.g., `isBlank`) |
| `com.salesmanager.core.service.cache.RefCache` | In‑house | Caches `Country` and `Zone` objects |
| `com.salesmanager.core.util.LanguageUtil` | In‑house | Retrieves numeric language codes |
| `com.salesmanager.core.util.LocaleUtil` | In‑house | Provides a default `Locale` |
| `com.salesmanager.core.entity.reference.Country` / `Zone` | In‑house | Domain entities for lookup |
| `com.salesmanager.core.entity.merchant.IMerchant` | In‑house | Interface this class implements (not shown) |
| `java.io.Serializable` | Java SE | Allows the object to be serialized |
| `java.util.Date`, `java.util.Locale`, `java.util.Map`, `java.util.StringTokenizer` | Java SE | Basic types and collections |

The class relies on a cache that is populated elsewhere; there are **no** external database connections or web frameworks embedded directly in the code.

---

## 5. Additional Notes  

### 5.1 Design and Maintainability  
* **Large, flat structure** – All address fields are stored on the same object; a cleaner approach would be to extract `Address` into a separate value object and embed it twice (shipping + billing).  
* **Mutable state** – The class is fully mutable; concurrency control is the responsibility of the caller.  
* **Legacy API usage** – `StringTokenizer`, raw `Map`, `Date` instead of `java.time`.  

### 5.2 Correctness & Robustness  
* **`getEmail()`** assumes a `:` separator; if the email never contains `:`, the method still returns the full address, but if the address is `null` it will throw a `NullPointerException`.  
* **`getShippingSate()`** typo may cause confusion or runtime errors if the method is referenced via reflection or by external code.  
* **`equals()` / `hashCode()`** compare *every* field.  For an ORM entity this is risky because the entity may be persisted with a subset of its fields initialized.  Equality should normally be based on the primary key (`customerId`).  
* **`hashCode()`** does not include `serialVersionUID`, which can lead to unpredictable serialization behavior.  
* **`toString()`** delegates to `Object.toString()`, offering no useful debugging information.  

### 5.3 Performance  
* **Cache lookups** (`getShippingCountry`, `getBillingCountry`, etc.) create a new `Map` reference each call and potentially hit the cache’s `getAllCountriesMap`.  Caching strategy and performance impact are not visible here but could become significant if these helpers are called frequently.  

### 5.4 Extensibility  
* The class assumes that `customerNewsletter` is a `Character` (`'Y'/'N'` style).  Modern code would use a boolean or an enum to avoid magic characters.  
* Adding new address fields (e.g., `stateCode`) would require touching the entity and all associated helpers.  A value object would isolate these changes.  

### 5.5 Suggested Improvements  

| Issue | Suggested Fix |
|-------|---------------|
| No `serialVersionUID` | Add `private static final long serialVersionUID = 1L;` |
| `equals()/hashCode()` compare all fields | Base them solely on `customerId` (or use `Objects.equals`) |
| Raw `Map` and unchecked casts | Use generics (`Map<Integer, Zone>`) |
| Deprecated `StringTokenizer` | Use `String.split` or `Pattern` |
| `Date` for DOB | Switch to `java.time.LocalDate` (or `OffsetDateTime` if time zone matters) |
| `toString()` useless | Override to display key customer details |
| Typo in method name | Rename `getShippingSate()` → `getShippingState()` |
| Separate billing & shipping logic | Extract `Address` value object and embed it twice |
| Use of `int` for country/zone IDs | Keep as `int` but document semantics; consider `Long` if IDs can be `null` |
| `Locale` lazy init | Initialize in constructor or guard against `null` in helpers |
| Hard‑coded anonymous prefix handling | Provide a configurable mask (perhaps via configuration) |
| Mixing JPA/Hibernate XML style comments | Add real JPA annotations (`@Entity`, `@Table`, `@Column`) or keep mapping in XML but remove the misleading comments |
| No validation | Add bean‑validation annotations (`@NotNull`, `@Email`) or service‑layer checks |

### 5.6 Licensing & Header  
The file begins with a BSD‑style license header.  That is fine, but the comment still contains Hibernate Javadoc tags (`@hibernate.*`) that are no longer used by the code – consider removing or converting to JPA annotations to avoid confusion.

---

**Verdict** – The class is functional and fits its role as an ORM entity, but it carries a number of legacy‑style pitfalls that can surface in edge cases, performance regressions, or maintenance headaches.  Refactoring toward a more modular, type‑safe, and JPA‑centric design would significantly improve readability, testability, and future‑proofing.

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

import java.io.Serializable;
import java.util.Date;
import java.util.Locale;
import java.util.Map;
import java.util.StringTokenizer;

import org.apache.commons.lang.StringUtils;

import com.salesmanager.core.entity.merchant.IMerchant;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.LocaleUtil;

/**
 * This is an object that contains data related to the customers table. Do not
 * modify this class because it will be overwritten if the configuration file
 * related to this class is modified.
 * 
 * @hibernate.class table="customers"
 */

public class Customer implements Serializable, IMerchant {

	public static String REF = "Customer";
	public static String PROP_CUSTOMER_DOB = "customerDob";
	public static String PROP_CUSTOMER_REFERRAL = "customerReferral";
	public static String PROP_CUSTOMER_NICK = "customerNick";
	public static String PROP_CUSTOMER_DEFAULT_ADDRESS_ID = "customerDefaultAddressId";
	public static String PROP_CUSTOMER_EMAIL_FORMAT = "customerEmailFormat";
	public static String PROP_CUSTOMER_GROUP_PRICING = "customerGroupPricing";
	public static String PROP_CUSTOMER_LANG = "customerLang";
	public static String PROP_CUSTOMER_PAYPAL_EC = "customerPaypalEc";
	public static String PROP_CUSTOMER_AUTHORIZATION = "customerAuthorization";
	public static String PROP_CUSTOMER_LASTNAME = "customerLastname";
	public static String PROP_CUSTOMER_GENDER = "customerGender";
	public static String PROP_CUSTOMER_FAX = "customerFax";
	public static String PROP_CUSTOMER_TELEPHONE = "customerTelephone";
	public static String PROP_CUSTOMER_FIRSTNAME = "customerFirstname";
	public static String PROP_ANONYMOUS = "customerAnonymous";
	public static String PROP_CUSTOMER_NEWSLETTER = "customerNewsletter";
	public static String PROP_CUSTOMER_EMAIL_ADDRESS = "customerEmailAddress";
	public static String PROP_CUSTOMER_PASSWORD = "customerPassword";

	// constructors
	public Customer() {
		initialize();
	}

	/**
	 * Constructor for primary key
	 */
	public Customer(long customerId) {
		this.setCustomerId(customerId);
		initialize();
	}

	protected void initialize() {

		this.setCustomerGender("");
		this.setCustomerDob(new Date(new Date().getTime()));
		this.setCustomerNick("");
		this.setCustomerDefaultAddressId(0);
		this.setCustomerPassword("");
		this.setCustomerGroupPricing(0);
		this.setCustomerEmailFormat("");
		this.setCustomerAuthorization(0);
		this.setCustomerAnonymous(true);
		this.setCustomerReferral("");
		this.setCustomerState("");
		this.setCustomerFirstname("");
		this.setCustomerLastname("");
		this.setCustomerBillingCountryName("");
		this.setCustomerStreetAddress("");
		this.setCustomerPostalCode("");
		this.setCustomerTelephone("");
		this.setCustomerCity("");
		// locale = LocaleUtil.getDefaultLocale();

	}

	// primary key
	private long customerId;

	// fields
	private java.lang.String customerGender;
	private java.lang.String customerFirstname;
	private java.lang.String customerLastname;
	private java.util.Date customerDob;
	private java.lang.String customerEmailAddress;
	private java.lang.String customerNick;
	private int customerDefaultAddressId;
	private java.lang.String customerTelephone;
	private java.lang.String customerFax;
	private java.lang.String customerPassword;
	private java.lang.Character customerNewsletter;
	private int customerGroupPricing;
	private java.lang.String customerEmailFormat;
	private int customerAuthorization;
	private java.lang.String customerReferral;
	private boolean customerAnonymous;
	private java.lang.String customerLang;
	private int merchantId;

	// shipping address
	private String customerStreetAddress;
	private String customerPostalCode;
	private String customerCity;
	private String customerCompany;
	private int customerZoneId;
	private int customerCountryId;
	private String customerState;

	// billing address
	private String customerBillingStreetAddress;
	private String customerBillingPostalCode;
	private String customerBillingCity;
	private int customerBillingZoneId;
	private int customerBillingCountryId;
	private String customerBillingState;
	private String customerBillingCountryName;
	private String customerBillingFirstName;
	private String customerBillingLastName;
	private String customerBillingCompany;

	private String name = null;

	private String stateProvinceName;
	private String countryName;

	private Locale locale;

	/**
	 * Return the unique identifier of this class
	 * 
	 * @hibernate.id generator-class="assigned" column="customers_id"
	 */
	public long getCustomerId() {
		return customerId;
	}

	/**
	 * Set the unique identifier of this class
	 * 
	 * @param customerId
	 *            the new ID
	 */
	public void setCustomerId(long customerId) {
		this.customerId = customerId;
	}

	/**
	 * Return the value associated with the column: customers_gender
	 */
	public java.lang.String getCustomerGender() {
		return customerGender;
	}

	/**
	 * Set the value related to the column: customers_gender
	 * 
	 * @param customerGender
	 *            the customers_gender value
	 */
	public void setCustomerGender(java.lang.String customerGender) {
		this.customerGender = customerGender;
	}

	/**
	 * Return the value associated with the column: customers_firstname
	 */
	public java.lang.String getCustomerFirstname() {
		return customerFirstname;
	}

	/**
	 * Set the value related to the column: customers_firstname
	 * 
	 * @param customerFirstname
	 *            the customers_firstname value
	 */
	public void setCustomerFirstname(java.lang.String customerFirstname) {
		this.customerFirstname = customerFirstname;
	}

	/**
	 * Return the value associated with the column: customers_lastname
	 */
	public java.lang.String getCustomerLastname() {
		return customerLastname;
	}

	/**
	 * Set the value related to the column: customers_lastname
	 * 
	 * @param customerLastname
	 *            the customers_lastname value
	 */
	public void setCustomerLastname(java.lang.String customerLastname) {
		this.customerLastname = customerLastname;
	}

	/**
	 * Return the value associated with the column: customers_dob
	 */
	public java.util.Date getCustomerDob() {
		return customerDob;
	}

	/**
	 * Set the value related to the column: customers_dob
	 * 
	 * @param customerDob
	 *            the customers_dob value
	 */
	public void setCustomerDob(java.util.Date customerDob) {
		this.customerDob = customerDob;
	}

	/**
	 * Return the value associated with the column: customers_email_address
	 */
	public java.lang.String getCustomerEmailAddress() {
		return customerEmailAddress;
	}

	/**
	 * Set the value related to the column: customers_email_address
	 * 
	 * @param customerEmailAddress
	 *            the customers_email_address value
	 */
	public void setCustomerEmailAddress(java.lang.String customerEmailAddress) {
		this.customerEmailAddress = customerEmailAddress;
	}

	/**
	 * Return the value associated with the column: customers_nick
	 */
	public java.lang.String getCustomerNick() {
		return customerNick;
	}

	/**
	 * Set the value related to the column: customers_nick
	 * 
	 * @param customerNick
	 *            the customers_nick value
	 */
	public void setCustomerNick(java.lang.String customerNick) {
		this.customerNick = customerNick;
	}

	/**
	 * Return the value associated with the column: customers_default_address_id
	 */
	public int getCustomerDefaultAddressId() {
		return customerDefaultAddressId;
	}

	/**
	 * Set the value related to the column: customers_default_address_id
	 * 
	 * @param customerDefaultAddressId
	 *            the customers_default_address_id value
	 */
	public void setCustomerDefaultAddressId(int customerDefaultAddressId) {
		this.customerDefaultAddressId = customerDefaultAddressId;
	}

	/**
	 * Return the value associated with the column: customers_telephone
	 */
	public java.lang.String getCustomerTelephone() {
		return customerTelephone;
	}

	/**
	 * Set the value related to the column: customers_telephone
	 * 
	 * @param customerTelephone
	 *            the customers_telephone value
	 */
	public void setCustomerTelephone(java.lang.String customerTelephone) {
		this.customerTelephone = customerTelephone;
	}

	/**
	 * Return the value associated with the column: customers_fax
	 */
	public java.lang.String getCustomerFax() {
		return customerFax;
	}

	/**
	 * Set the value related to the column: customers_fax
	 * 
	 * @param customerFax
	 *            the customers_fax value
	 */
	public void setCustomerFax(java.lang.String customerFax) {
		this.customerFax = customerFax;
	}

	/**
	 * Return the value associated with the column: customers_password
	 */
	public java.lang.String getCustomerPassword() {
		return customerPassword;
	}

	/**
	 * Set the value related to the column: customers_password
	 * 
	 * @param customerPassword
	 *            the customers_password value
	 */
	public void setCustomerPassword(java.lang.String customerPassword) {
		this.customerPassword = customerPassword;
	}

	/**
	 * Return the value associated with the column: customers_newsletter
	 */
	public java.lang.Character getCustomerNewsletter() {
		return customerNewsletter;
	}

	/**
	 * Set the value related to the column: customers_newsletter
	 * 
	 * @param customerNewsletter
	 *            the customers_newsletter value
	 */
	public void setCustomerNewsletter(java.lang.Character customerNewsletter) {
		this.customerNewsletter = customerNewsletter;
	}

	/**
	 * Return the value associated with the column: customers_group_pricing
	 */
	public int getCustomerGroupPricing() {
		return customerGroupPricing;
	}

	/**
	 * Set the value related to the column: customers_group_pricing
	 * 
	 * @param customerGroupPricing
	 *            the customers_group_pricing value
	 */
	public void setCustomerGroupPricing(int customerGroupPricing) {
		this.customerGroupPricing = customerGroupPricing;
	}

	/**
	 * Return the value associated with the column: customers_email_format
	 */
	public java.lang.String getCustomerEmailFormat() {
		return customerEmailFormat;
	}

	/**
	 * Set the value related to the column: customers_email_format
	 * 
	 * @param customerEmailFormat
	 *            the customers_email_format value
	 */
	public void setCustomerEmailFormat(java.lang.String customerEmailFormat) {
		this.customerEmailFormat = customerEmailFormat;
	}

	/**
	 * Return the value associated with the column: customers_authorization
	 */
	public int getCustomerAuthorization() {
		return customerAuthorization;
	}

	/**
	 * Set the value related to the column: customers_authorization
	 * 
	 * @param customerAuthorization
	 *            the customers_authorization value
	 */
	public void setCustomerAuthorization(int customerAuthorization) {
		this.customerAuthorization = customerAuthorization;
	}

	/**
	 * Return the value associated with the column: customers_referral
	 */
	public java.lang.String getCustomerReferral() {
		return customerReferral;
	}

	/**
	 * Set the value related to the column: customers_referral
	 * 
	 * @param customerReferral
	 *            the customers_referral value
	 */
	public void setCustomerReferral(java.lang.String customerReferral) {
		this.customerReferral = customerReferral;
	}

	/**
	 * Return the value associated with the column: customers_paypal_ec
	 */
	public boolean isCustomerAnonymous() {
		return customerAnonymous;
	}

	/**
	 * Set the value related to the column: customers_paypal_ec
	 * 
	 * @param customerPaypalEc
	 *            the customers_paypal_ec value
	 */
	public void setCustomerAnonymous(boolean customerAnonymous) {
		this.customerAnonymous = customerAnonymous;
	}

	/**
	 * Return the value associated with the column: customer_lang
	 */
	public java.lang.String getCustomerLang() {
		return customerLang;
	}

	/**
	 * Set the value related to the column: customer_lang
	 * 
	 * @param customerLang
	 *            the customer_lang value
	 */
	public void setCustomerLang(java.lang.String customerLang) {
		this.customerLang = customerLang;
	}

	/**
	 * Strip anonymous characters
	 * 
	 * @return
	 */
	public String getEmail() {
		StringTokenizer st = new StringTokenizer(
				this.getCustomerEmailAddress(), ":");
		int i = 0;
		String email = null;
		while (st.hasMoreTokens()) {
			String t = st.nextToken();
			if (i == 0) {
				email = t;
			}
			i++;
		}
		return email;
	}

	public String toString() {
		return super.toString();
	}

	public String getName() {
		if (!StringUtils.isBlank(name)) {
			return name;
		} else {
			return this.getCustomerFirstname() + " "
					+ this.getCustomerLastname();
		}
	}

	public void setName(String name) {
		this.name = name;
	}

	public int getMerchantId() {
		return merchantId;
	}

	public void setMerchantId(int merchantId) {
		this.merchantId = merchantId;
	}

	@Override
	public int hashCode() {
		final int PRIME = 31;
		int result = 1;
		result = PRIME * result + (customerAnonymous ? 1231 : 1237);
		result = PRIME * result + customerAuthorization;
		result = PRIME * result + customerDefaultAddressId;
		result = PRIME * result
				+ ((customerDob == null) ? 0 : customerDob.hashCode());
		result = PRIME
				* result
				+ ((customerEmailAddress == null) ? 0 : customerEmailAddress
						.hashCode());
		result = PRIME
				* result
				+ ((customerEmailFormat == null) ? 0 : customerEmailFormat
						.hashCode());
		result = PRIME * result
				+ ((customerFax == null) ? 0 : customerFax.hashCode());
		result = PRIME
				* result
				+ ((customerFirstname == null) ? 0 : customerFirstname
						.hashCode());
		result = PRIME * result
				+ ((customerGender == null) ? 0 : customerGender.hashCode());
		result = PRIME * result + customerGroupPricing;
		result = PRIME * result + (int) (customerId ^ (customerId >>> 32));
		result = PRIME * result
				+ ((customerLang == null) ? 0 : customerLang.hashCode());
		result = PRIME
				* result
				+ ((customerLastname == null) ? 0 : customerLastname.hashCode());
		result = PRIME
				* result
				+ ((customerNewsletter == null) ? 0 : customerNewsletter
						.hashCode());
		result = PRIME * result
				+ ((customerNick == null) ? 0 : customerNick.hashCode());
		result = PRIME
				* result
				+ ((customerPassword == null) ? 0 : customerPassword.hashCode());
		result = PRIME
				* result
				+ ((customerReferral == null) ? 0 : customerReferral.hashCode());
		result = PRIME
				* result
				+ ((customerTelephone == null) ? 0 : customerTelephone
						.hashCode());
		result = PRIME * result + merchantId;
		return result;
	}

	@Override
	public boolean equals(Object obj) {
		if (this == obj)
			return true;
		if (obj == null)
			return false;
		if (getClass() != obj.getClass())
			return false;
		final Customer other = (Customer) obj;
		if (customerAnonymous != other.customerAnonymous)
			return false;
		if (customerAuthorization != other.customerAuthorization)
			return false;
		if (customerDefaultAddressId != other.customerDefaultAddressId)
			return false;
		if (customerDob == null) {
			if (other.customerDob != null)
				return false;
		} else if (!customerDob.equals(other.customerDob))
			return false;
		if (customerEmailAddress == null) {
			if (other.customerEmailAddress != null)
				return false;
		} else if (!customerEmailAddress.equals(other.customerEmailAddress))
			return false;
		if (customerEmailFormat == null) {
			if (other.customerEmailFormat != null)
				return false;
		} else if (!customerEmailFormat.equals(other.customerEmailFormat))
			return false;
		if (customerFax == null) {
			if (other.customerFax != null)
				return false;
		} else if (!customerFax.equals(other.customerFax))
			return false;
		if (customerFirstname == null) {
			if (other.customerFirstname != null)
				return false;
		} else if (!customerFirstname.equals(other.customerFirstname))
			return false;
		if (customerGender == null) {
			if (other.customerGender != null)
				return false;
		} else if (!customerGender.equals(other.customerGender))
			return false;
		if (customerGroupPricing != other.customerGroupPricing)
			return false;
		if (customerId != other.customerId)
			return false;
		if (customerLang == null) {
			if (other.customerLang != null)
				return false;
		} else if (!customerLang.equals(other.customerLang))
			return false;
		if (customerLastname == null) {
			if (other.customerLastname != null)
				return false;
		} else if (!customerLastname.equals(other.customerLastname))
			return false;
		if (customerNewsletter == null) {
			if (other.customerNewsletter != null)
				return false;
		} else if (!customerNewsletter.equals(other.customerNewsletter))
			return false;
		if (customerNick == null) {
			if (other.customerNick != null)
				return false;
		} else if (!customerNick.equals(other.customerNick))
			return false;
		if (customerPassword == null) {
			if (other.customerPassword != null)
				return false;
		} else if (!customerPassword.equals(other.customerPassword))
			return false;
		if (customerReferral == null) {
			if (other.customerReferral != null)
				return false;
		} else if (!customerReferral.equals(other.customerReferral))
			return false;
		if (customerTelephone == null) {
			if (other.customerTelephone != null)
				return false;
		} else if (!customerTelephone.equals(other.customerTelephone))
			return false;
		if (merchantId != other.merchantId)
			return false;
		return true;
	}

	public String getCustomerCity() {
		return customerCity;
	}

	public void setCustomerCity(String customerCity) {
		this.customerCity = customerCity;
	}

	public String getCustomerCompany() {
		return customerCompany;
	}

	public void setCustomerCompany(String customerCompany) {
		this.customerCompany = customerCompany;
	}

	public int getCustomerCountryId() {
		return customerCountryId;
	}

	public void setCustomerCountryId(int customerCountryId) {
		this.customerCountryId = customerCountryId;
	}

	public String getCustomerPostalCode() {
		return customerPostalCode;
	}

	public void setCustomerPostalCode(String customerPostalCode) {
		this.customerPostalCode = customerPostalCode;
	}

	public String getCustomerStreetAddress() {
		return customerStreetAddress;
	}

	public void setCustomerStreetAddress(String customerStreetAddress) {
		this.customerStreetAddress = customerStreetAddress;
	}

	public int getCustomerZoneId() {
		return customerZoneId;
	}

	public void setCustomerZoneId(int customerZoneId) {
		this.customerZoneId = customerZoneId;
	}

	public String getCustomerState() {
		return customerState;
	}

	public void setCustomerState(String customerState) {
		this.customerState = customerState;
	}

	public String getCountryName() {
		return countryName;
	}

	public void setCountryName(String countryName) {
		this.countryName = countryName;
	}

	public String getStateProvinceName() {

		if (!StringUtils.isBlank(this.getCustomerState())) {
			return this.getCustomerState();
		}
		return stateProvinceName;
	}

	public void setStateProvinceName(String stateProvinceName) {
		this.stateProvinceName = stateProvinceName;
	}

	public String getCustomerBillingCity() {
		return customerBillingCity;
	}

	public void setCustomerBillingCity(String customerBillingCity) {
		this.customerBillingCity = customerBillingCity;
	}

	public int getCustomerBillingCountryId() {
		return customerBillingCountryId;
	}

	public void setCustomerBillingCountryId(int customerBillingCountryId) {
		this.customerBillingCountryId = customerBillingCountryId;
	}

	public String getCustomerBillingPostalCode() {
		return customerBillingPostalCode;
	}

	public void setCustomerBillingPostalCode(String customerBillingPostalCode) {
		this.customerBillingPostalCode = customerBillingPostalCode;
	}

	public String getCustomerBillingState() {
		return customerBillingState;
	}

	public void setCustomerBillingState(String customerBillingState) {
		this.customerBillingState = customerBillingState;
	}

	public String getCustomerBillingStreetAddress() {
		return customerBillingStreetAddress;
	}

	public void setCustomerBillingStreetAddress(
			String customerBillingStreetAddress) {
		this.customerBillingStreetAddress = customerBillingStreetAddress;
	}

	public int getCustomerBillingZoneId() {
		return customerBillingZoneId;
	}

	public void setCustomerBillingZoneId(int customerBillingZoneId) {
		this.customerBillingZoneId = customerBillingZoneId;
	}

	public String getShippingSate() {
		if (!StringUtils.isBlank(this.getStateProvinceName())) {
			return this.getStateProvinceName();
		} else {
			if (locale == null) {
				locale = LocaleUtil.getDefaultLocale();
			}
			Map zones = RefCache.getAllZonesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));
			Zone zone = (Zone) zones.get(this.getCustomerZoneId());
			if (zone != null) {
				return zone.getZoneName();
			} else {
				return "";
			}
		}
	}

	public String getShippingCountry() {

		if (locale == null) {
			locale = LocaleUtil.getDefaultLocale();
		}
		Map c = RefCache.getAllcountriesmap(LanguageUtil
				.getLanguageNumberCode(locale.getLanguage()));
		Country country = (Country) c.get(this.getCustomerCountryId());
		if (country != null) {
			return country.getCountryName();
		} else {
			return "";
		}

	}

	public String getBillingState() {
		if (!StringUtils.isBlank(this.getCustomerBillingState())) {
			return this.getCustomerBillingState();
		} else {
			if (locale == null) {
				locale = LocaleUtil.getDefaultLocale();
			}
			Map zones = RefCache.getAllZonesmap(LanguageUtil
					.getLanguageNumberCode(locale.getLanguage()));
			Zone zone = (Zone) zones.get(this.getCustomerBillingZoneId());
			if (zone != null) {
				return zone.getZoneName();
			} else {
				return "";
			}
		}
	}

	public String getBillingCountry() {

		if (locale == null) {
			locale = LocaleUtil.getDefaultLocale();
		}
		Map c = RefCache.getAllcountriesmap(LanguageUtil
				.getLanguageNumberCode(locale.getLanguage()));
		Country country = (Country) c.get(this.getCustomerBillingCountryId());
		if (country != null) {
			return country.getCountryName();
		} else {
			return "";
		}

	}

	public Locale getLocale() {
		return locale;
	}

	public void setLocale(Locale locale) {
		this.locale = locale;

	}

	public String getCustomerNameId() {
		return this.getName() + " (" + this.getCustomerId() + ")";
	}

	public String getCustomerBillingCountryName() {
		return customerBillingCountryName;
	}

	public void setCustomerBillingCountryName(String customerBillingCountryName) {
		this.customerBillingCountryName = customerBillingCountryName;
	}

	public String getCustomerBillingCompany() {
		return customerBillingCompany;
	}

	public void setCustomerBillingCompany(String customerBillingCompany) {
		this.customerBillingCompany = customerBillingCompany;
	}

	public String getCustomerBillingFirstName() {
		return customerBillingFirstName;
	}

	public void setCustomerBillingFirstName(String customerBillingFirstName) {
		this.customerBillingFirstName = customerBillingFirstName;
	}

	public String getCustomerBillingLastName() {
		return customerBillingLastName;
	}

	public void setCustomerBillingLastName(String customerBillingLastName) {
		this.customerBillingLastName = customerBillingLastName;
	}

}


```
