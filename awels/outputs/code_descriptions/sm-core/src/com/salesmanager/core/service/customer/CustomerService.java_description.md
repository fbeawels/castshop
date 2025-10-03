# CustomerService.java

## Review

## 1. Summary  

**Purpose** –  
`CustomerService` is a Spring‑managed service layer that handles all business‑logic around *customers* in the SalesManager core module.  It provides CRUD, search, authentication, basket and password‑management capabilities for customers of a merchant store.

**Key components**  

| Component | Responsibility |
|-----------|----------------|
| `CustomerDao` (`ICustomerDao`) | Low‑level persistence (JPA/Hibernate) for `Customer` entities |
| `CustomerInfoDao` (`ICustomerInfoDao`) | Persistence for `CustomerInfo` (audit/log‑on data) |
| `MerchantService` | Retrieves merchant‑specific data such as the store details |
| `PasswordGeneratorModule` | Generates random passwords |
| `EncryptionUtil` | Symmetric encryption of passwords |
| `CommonService` | Sends templated e‑mails (FreeMarker) |
| `LabelUtil` | Internationalisation of e‑mail text |
| `ReferenceUtil` | Builds URLs for the catalog / portal |

**Notable patterns / frameworks**  

* Spring dependency injection (`@Service`, `@Autowired`)  
* Spring transaction demarcation (`@Transactional`)  
* DAO pattern (service delegates persistence)  
* Template‑based e‑mail rendering (FreeMarker)  
* Internationalisation through label files  

---

## 2. Detailed Description  

### 2.1 Execution flow  

1. **Startup** – Spring creates a singleton `CustomerService`.  
2. **Dependency wiring** – `customerDao` and `customerInfoDao` are injected.  
3. **Method call** – A client (controller or another service) calls one of the public methods.  
4. **Transaction** – Each public method is wrapped in a Spring transaction.  
5. **Business logic** –  
   * For CRUD: delegate to DAO.  
   * For authentication & password handling: generate/encrypt password, check uniqueness, update entity, send e‑mail.  
   * For basket handling: persist `CustomerBasket` and its attributes.  
6. **Commit / rollback** – On normal exit the transaction commits; any uncaught exception triggers a rollback (methods annotated with `rollbackFor = Exception.class`).

### 2.2 Core responsibilities  

| Responsibility | Where it lives |
|----------------|----------------|
| Customer lookup (by id, email, username, company) | DAO + service helper methods |
| Customer search with criteria | DAO (`findCustomers`) |
| Customer create/update | `saveOrUpdateCustomer` – handles new customers, password generation, e‑mail notification, and `CustomerInfo` bookkeeping |
| Password reset / change | `resetCustomerPassword`, `changeCustomerPassword` – generate new password, encrypt, persist, e‑mail |
| Basket persistence | `addProductToSavedCart` |
| Log‑on statistics | `processLastLoggedInDate`, `findCustomerInfoById` |
| Cleanup | `deleteCustomer`, `deleteAllCustomers` |

### 2.3 Assumptions & constraints  

* **Password security** – Uses a constant key (`SecurityConstants.idConstant`) for all passwords.  
  * This is not ideal (no per‑user salt, no key rotation).  
* **Email** – Hard‑coded template names and reliance on `CommonService`.  
* **Transaction isolation** – All public methods are transactional; nested calls will join the same transaction.  
* **Locale handling** – Labels are fetched based on the customer’s locale or a default.  
* **No caching** – DAO methods appear to hit the database directly each time.  

### 2.4 Architecture  

A classic three‑tier architecture:  
* **Presentation** – MVC controllers (not shown) call `CustomerService`.  
* **Service** – `CustomerService` contains pure business logic, coordinates DAO, external services, and side‑effects (e‑mail).  
* **Data access** – `ICustomerDao` and `ICustomerInfoDao` abstract persistence.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects |
|--------|---------|--------|---------|--------------|
| `getCustomersByCompanyName(int merchantId, String companyName)` | Retrieve all customers of a merchant that match a company name | merchantId, companyName | `Collection<Customer>` | None |
| `getCustomersHavingCompanies(int merchantId)` | Get customers that have an associated company record | merchantId | `Collection<Customer>` | None |
| `getUniqueCustomerCompanyNameList(int merchantId)` | Fetch distinct company names for a merchant | merchantId | `Collection<String>` | None |
| `addProductToSavedCart(CustomerBasket basket)` | Persist a `CustomerBasket` and its attributes | `CustomerBasket` | void | DB writes |
| `getCustomer(long customerId)` | Load a customer by primary key | customerId | `Customer` | None |
| `getCustomerList(int merchantId)` | Load all customers for a merchant | merchantId | `Collection<Customer>` | None |
| `deleteAllCustomers(int merchantId)` | Delete every customer of a merchant | merchantId | void | Cascades delete to `CustomerInfo` |
| `deleteCustomer(Customer customer)` | Delete a single customer and its `CustomerInfo` | `Customer` | void | DB delete |
| `searchCustomers(SearchCustomerCriteria criteria)` | Advanced search with filtering/pagination | `SearchCustomerCriteria` | `SearchCustomerResponse` | None |
| `saveOrUpdateCustomer(Customer customer, SystemUrlEntryType entryType, Locale locale)` | Create or update a customer. Handles new‑customer workflow: generate password, encrypt, send email, create `CustomerInfo` | `Customer`, optional entry type, locale | void | DB write, e‑mail |
| `resetCustomerPassword(Customer customer)` | Generate new password for a non‑anonymous customer, update, send e‑mail | `Customer` | void | DB write, e‑mail |
| `changeCustomerPassword(Customer customer, String oldPassword, String newPassword)` | Verify old password, replace with new encrypted one, send notification | `Customer`, old, new | `boolean` (success) | DB write, e‑mail |
| `findCustomerbyUserNameAndPassword(String userName, String password, int merchantId)` | Load a customer by username/password (encrypted) | credentials, merchantId | `Customer` | None |
| `findCustomerByEmail(String email)` | Load customer by e‑mail address | email | `Customer` | None |
| `findCustomerByUserName(String userName, int merchantId)` | Load customer by username | userName, merchantId | `Customer` | None |
| `processLastLoggedInDate(long customerId)` | Update last‑login timestamp & login count; returns previous timestamp | customerId | `Date` (previous login) | DB update |
| `saveOrUpdateCustomerInfo(CustomerInfo customerInfo)` | Persist `CustomerInfo` | `CustomerInfo` | void | DB write |
| `findCustomerInfoById(long id)` | Load `CustomerInfo` by ID | id | `CustomerInfo` | None |

*Utility* – No explicit helper methods; most logic is embedded in the public methods.

---

## 4. Dependencies  

| Library / Module | Type | Notes |
|------------------|------|-------|
| Spring Framework | Third‑party | `@Service`, `@Autowired`, `@Transactional` |
| Hibernate / JPA (via DAO) | Third‑party | Data persistence (not shown) |
| Apache Commons Configuration | Third‑party | Reading properties |
| Apache Commons Lang | Third‑party | `StringUtils` |
| SpringUtil (custom) | Custom | Bean lookup via `SpringUtil.getBean()` |
| EncryptionUtil | Custom | Symmetric encryption, uses constant key |
| PasswordGeneratorModule | Custom | Generates random passwords |
| LabelUtil | Custom | i18n label resolution |
| ReferenceUtil | Custom | Builds URLs for catalog/portal |
| CommonService | Custom | Sends FreeMarker e‑mail templates |
| `SystemUrlEntryType` | Custom | Enum for entry type (WEB/PORTAL) |
| `SecurityConstants` | Custom | Holds constant id for encryption key |
| `MerchantService`, `MerchantStore`, `MerchantUserInformation` | Custom | Merchant‑related data |

All dependencies are standard for a Spring‑based e‑commerce application; no external frameworks (e.g., Spring Security) are used for authentication.

---

## 5. Additional Notes & Recommendations  

### 5.1 Security concerns  
* **Static encryption key** – Using a single constant key (`SecurityConstants.idConstant`) for all passwords is weak. A per‑user salt + PBKDF2/BCrypt would be far more secure.  
* **Password generation** – The loop that regenerates a password until it is unique is inefficient for high‑volume systems. A random password generator with a sufficiently large space should eliminate collisions.  
* **Storing passwords** – The encrypted value is persisted directly. There is no hashing or salt; if the key is compromised, all passwords are exposed.  

### 5.2 Code quality / maintainability  
* **Raw types** – Several collections (`Collection`, `List`) are used without generics (e.g., `List basketAttributes`). Use generics to avoid unchecked casts.  
* **Duplication** – `resetCustomerPassword` and `changeCustomerPassword` share large chunks of code. Extract a private method (`createNewPasswordForCustomer`) to avoid duplication.  
* **Hard‑coded URLs** – Portal / web URLs are built inline. Centralise these in configuration or a URL builder.  
* **Logging** – No logging statements. Adding SLF4J logs would aid troubleshooting.  
* **Exception handling** – Methods throw generic `Exception`. Prefer custom exceptions or at least restrict to `RuntimeException`.  
* **Magic strings** – The e‑mail templates (`"email_template_customer.ftl"`, `"email_template_password_reset_customer.ftl"`) could be constants or externalised.  
* **Transactional granularity** – All methods are transactional; consider read‑only transactions for lookup methods (`@Transactional(readOnly = true)`).  

### 5.3 Testability  
* **DAO mocking** – The service is testable with mock DAOs.  
* **Email sending** – `CommonService` is instantiated directly; inject it via `@Autowired` so it can be mocked.  

### 5.4 Future enhancements  
1. **Integrate Spring Security** – Delegate authentication & password hashing to Spring Security.  
2. **Implement caching** – Frequently accessed customers could be cached (e.g., with EHCache or Spring Cache).  
3. **Audit trail** – Store more detailed logs of password changes, email notifications.  
4. **Internationalisation** – Move label keys and templates into a proper i18n resource bundle.  
5. **Refactor** – Extract a `PasswordService` and `EmailService` to isolate responsibilities.

---

**Verdict** – The class fulfils its functional requirements but can benefit from modern security practices, stronger type safety, and cleaner separation of concerns. Addressing the above points would make the service more robust, secure, and maintainable.

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
package com.salesmanager.core.service.customer;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.salesmanager.core.constants.SecurityConstants;
import com.salesmanager.core.entity.customer.Customer;
import com.salesmanager.core.entity.customer.CustomerBasket;
import com.salesmanager.core.entity.customer.CustomerBasketAttribute;
import com.salesmanager.core.entity.customer.CustomerInfo;
import com.salesmanager.core.entity.customer.SearchCustomerCriteria;
import com.salesmanager.core.entity.customer.SearchCustomerResponse;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.reference.SystemUrlEntryType;
import com.salesmanager.core.module.model.application.PasswordGeneratorModule;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.service.customer.impl.dao.ICustomerDao;
import com.salesmanager.core.service.customer.impl.dao.ICustomerInfoDao;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.EncryptionUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.SpringUtil;

@Service
public class CustomerService {

	private static Configuration config = PropertiesUtil.getConfiguration();

	@Autowired
	private ICustomerDao customerDao;

	@Autowired
	private ICustomerInfoDao customerInfoDao;

	@Transactional
	public Collection<Customer> getCustomersByCompanyName(int merchantId,
			String companyName) throws Exception {
		return customerDao.findByCompanyName(companyName, merchantId);
	}

	@Transactional
	public Collection<Customer> getCustomersHavingCompanies(int merchantId)
			throws Exception {
		return customerDao.findCustomersHavingCompany(merchantId);
	}

	@Transactional
	public Collection<String> getUniqueCustomerCompanyNameList(int merchantId)
			throws Exception {
		return customerDao.findUniqueCompanyName(merchantId);
	}

	/**
	 * When a customer is logged in the product is added to the CUSTOMERS_BASKET
	 * table
	 * 
	 * @param productid
	 * @param quantity
	 * @param price
	 * @param merchantid
	 */
	@Transactional
	public void addProductToSavedCart(CustomerBasket basket) throws Exception {

		// @todo, use saveall
		customerDao.saveShoppingCart(basket);
		List basketAttributes = basket.getCustomerBasketAttributes();
		if (basketAttributes != null) {
			Iterator i = basketAttributes.iterator();
			while (i.hasNext()) {
				CustomerBasketAttribute basketattribute = (CustomerBasketAttribute) i
						.next();
				customerDao.saveShoppingCartAttributes(basketattribute);
			}
		}

	}

	/**
	 * Retreives a Customer entity based on the id
	 * 
	 * @param customerId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Customer getCustomer(long customerId) throws Exception {
		return customerDao.findById(customerId);
	}
	


	/**
	 * Retreives a list of customer for a given merchantid
	 * 
	 * @param merchantId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Collection<Customer> getCustomerList(int merchantId)
			throws Exception {
		return customerDao.findByMerchantId(merchantId);
	}

	/**
	 * Deletes all Customer created by a given merchantId
	 * 
	 * @param merchantId
	 * @throws Exception
	 */
	@Transactional
	public void deleteAllCustomers(int merchantId) throws Exception {
		Collection customers = getCustomerList(merchantId);
		if (customers != null && customers.size() > 0) {
			Iterator i = customers.iterator();
			while (i.hasNext()) {
				Customer customer = (Customer) i.next();
				deleteCustomer(customer);
			}
		}
	}

	@Transactional
	public void deleteCustomer(Customer customer) throws Exception {
		CustomerInfo info = customerInfoDao.findById(customer.getCustomerId());
		if (info != null) {
			customerInfoDao.delete(info);
		}

		// customer basket

		// wishlist

		customerDao.delete(customer);
	}

	/**
	 * Search customers using criteria
	 * 
	 * @param criteria
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public SearchCustomerResponse searchCustomers(
			SearchCustomerCriteria criteria) throws Exception {
		return customerDao.findCustomers(criteria);
	}

	@Transactional(rollbackFor = { Exception.class })
	public void saveOrUpdateCustomer(Customer customer,
			SystemUrlEntryType entryType, Locale locale) throws Exception {

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		MerchantStore store = mservice.getMerchantStore(customer
				.getMerchantId());
		//MerchantUserInformation minfo = mservice.getMerchantUserInfo(customer
		//		.getMerchantId());

		if (entryType == null) {
			entryType = SystemUrlEntryType.WEB;
		}

		// check if email aleady exist

		boolean isNew = false;
		if (customer.getCustomerId() == 0) {
			isNew = true;

		}

		if (isNew && !customer.isCustomerAnonymous()) {

			// generate password
			PasswordGeneratorModule passwordGenerator = (PasswordGeneratorModule) SpringUtil
					.getBean("passwordgenerator");

			// encrypt
			String key = EncryptionUtil.generatekey(String
					.valueOf(SecurityConstants.idConstant));
			boolean found = true;

			String password = null;
			String encrypted = null;
			// validate if already exist
			while (found) {

				password = passwordGenerator.generatePassword();
				encrypted = EncryptionUtil.encrypt(key, password);
				Customer cfound = customerDao.findByUserNameAndPassword(
						customer.getCustomerNick(), encrypted);
				if (cfound == null) {
					found = false;
				}
			}

			// store in customer
			customer.setCustomerNick(customer.getCustomerEmailAddress());
			customer.setCustomerPassword(encrypted);

			// send email
			String l = config.getString("core.system.defaultlanguage", "en");
			if (!StringUtils.isBlank(customer.getCustomerLang())) {
				l = customer.getCustomerLang();
			}

			LabelUtil lhelper = LabelUtil.getInstance();
			String subject = lhelper.getText(l, "label.profile.information");
			List params = new ArrayList();
			params.add(store.getStorename());
			String greeting = lhelper.getText(locale,
					"label.email.customer.greeting", params);

			String username = lhelper.getText(l,
					"label.generic.customer.username")
					+ " " + customer.getCustomerNick();
			String pass = lhelper.getText(l, "label.generic.customer.password")
					+ " " + password;

			String info = "";
			String portalurl = "";

			if (entryType == SystemUrlEntryType.PORTAL) {
				info = lhelper.getText(l, "label.email.customer.portalinfo");
				String url = "<a href=\""
						+ config
								.getProperty("core.accountmanagement.portal.url")
						+ "/"
						+ customer.getMerchantId()
						+ "\">"
						+ config
								.getProperty("core.accountmanagement.portal.url")
						+ "/" + customer.getMerchantId() + "</a>";
				portalurl = lhelper
						.getText(l, "label.email.customer.portalurl")
						+ " " + url;
			} else {
				info = lhelper.getText(l, "label.email.customer.webinfo");
				String url = "<a href=\""
						+ ReferenceUtil.buildCatalogUri(store) + "/\">"
						+ ReferenceUtil.buildCatalogUri(store)
						+ "/landing.action?merchantId=" + store.getMerchantId()
						+ "</a>";
				portalurl = lhelper.getText(l, "label.email.customer.weburl")
						+ " " + url;
			}

			Map emailctx = new HashMap();
			emailctx.put("EMAIL_STORE_NAME", store.getStorename());
			emailctx.put("EMAIL_CUSTOMER_FIRSTNAME", customer
					.getCustomerFirstname());
			emailctx.put("EMAIL_CUSTOMER_LAST", customer.getCustomerLastname());
			emailctx.put("EMAIL_CUSTOMER_USERNAME", username);
			emailctx.put("EMAIL_CUSTOMER_PASSWORD", pass);
			emailctx.put("EMAIL_GREETING", greeting);
			emailctx.put("EMAIL_CUSTOMER_PORTAL_INFO", info);
			emailctx.put("EMAIL_CUSTOMER_PORTAL_ENTRY", portalurl);
			emailctx.put("EMAIL_CONTACT_OWNER", store.getStoreemailaddress());

			CommonService cservice = new CommonService();
			cservice.sendHtmlEmail(customer.getCustomerEmailAddress(), subject,
					store, emailctx, "email_template_customer.ftl",
					customer.getCustomerLang());

		}

		customerDao.saveOrUptade(customer);

		// set CustomerInfo

		CustomerInfo customerInfo = new CustomerInfo();
		customerInfo.setCustomerInfoId(customer.getCustomerId());

		int login = customerInfo.getCustomerInfoNumberOfLogon();
		customerInfo.setCustomerInfoNumberOfLogon(login++);
		customerInfo.setCustomerInfoDateOfLastLogon(new Date());
		customerInfoDao.saveOrUpdate(customerInfo);

	}

	/**
	 * Reset a Customer password. Will also send an email the the customer with
	 * the new password
	 * 
	 * @param customer
	 * @throws Exception
	 */
	@Transactional(rollbackFor = { Exception.class })
	public void resetCustomerPassword(Customer customer) throws Exception {

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(customer
				.getMerchantId());
		//MerchantUserInformation minfo = mservice.getMerchantUserInfo(customer
		//		.getMerchantId());

		if (!customer.isCustomerAnonymous()) {

			// generate password
			PasswordGeneratorModule passwordGenerator = (PasswordGeneratorModule) SpringUtil
					.getBean("passwordgenerator");

			// encrypt
			String key = EncryptionUtil.generatekey(String
					.valueOf(SecurityConstants.idConstant));
			boolean found = true;

			String password = null;
			String encrypted = null;
			// validate if already exist
			while (found) {

				password = passwordGenerator.generatePassword();
				encrypted = EncryptionUtil.encrypt(key, password);
				Customer cfound = customerDao.findByUserNameAndPassword(
						customer.getCustomerNick(), encrypted);
				if (cfound == null) {
					found = false;
				}
			}

			// store in customer
			customer.setCustomerNick(customer.getCustomerEmailAddress());
			customer.setCustomerPassword(encrypted);
			customerDao.saveOrUptade(customer);

			// send email
			String l = config.getString("core.system.defaultlanguage", "en");
			if (!StringUtils.isBlank(customer.getCustomerLang())) {
				l = customer.getCustomerLang();
			}

			LabelUtil lhelper = LabelUtil.getInstance();
			String subject = lhelper.getText(l, "label.profile.information");
			String info = lhelper.getText(l, "label.email.customer.portalinfo");
			String pass = lhelper.getText(l,
					"label.email.customer.passwordreset.text")
					+ " " + password;

			// @TODO replace suffix
			String url = "<a href=\""
					+ config.getString("core.accountmanagement.portal.url")
					+ "\">"
					+ config.getString("core.accountmanagement.portal.url")
					+ "</a>";
			String portalurl = lhelper.getText(l,
					"label.email.customer.portalurl")
					+ " " + url;

			Map emailctx = new HashMap();
			emailctx.put("EMAIL_STORE_NAME", store.getStorename());
			emailctx.put("EMAIL_CUSTOMER_PASSWORD", pass);
			emailctx.put("EMAIL_CUSTOMER_PORTAL_INFO", info);
			emailctx.put("EMAIL_CONTACT_OWNER", store.getStoreemailaddress());

			CommonService cservice = new CommonService();
			cservice.sendHtmlEmail(customer.getCustomerEmailAddress(), subject,
					store, emailctx,
					"email_template_password_reset_customer.ftl", customer
							.getCustomerLang());

		}

	}

	@Transactional
	public boolean changeCustomerPassword(Customer customer,
			String oldPassword, String newPassword) throws Exception {
		String key = EncryptionUtil.generatekey(String
				.valueOf(SecurityConstants.idConstant));
		String encrypted = EncryptionUtil.encrypt(key, newPassword);

		String old = EncryptionUtil.encrypt(key, oldPassword);

		if (!customer.getCustomerPassword().equals(old)) {
			return false;
		}

		customer.setCustomerPassword(encrypted);

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);

		//MerchantUserInformation minfo = mservice.getMerchantUserInfo(customer
		//		.getMerchantId());

		MerchantStore store = mservice.getMerchantStore(customer
				.getMerchantId());

		customerDao.saveOrUptade(customer);

		// send email
		String l = config.getString("core.system.defaultlanguage", "en");
		if (!StringUtils.isBlank(customer.getCustomerLang())) {
			l = customer.getCustomerLang();
		}

		LabelUtil lhelper = LabelUtil.getInstance();
		String subject = lhelper.getText(l, "label.profile.information");
		String info = lhelper.getText(l, "label.email.customer.portalinfo");
		String pass = lhelper.getText(l,
				"label.email.customer.passwordreset.text")
				+ " " + newPassword;

		// @TODO replace suffix
		String url = "<a href=\""
				+ config.getString("core.accountmanagement.portal.url") + "\">"
				+ config.getProperty("core.accountmanagement.portal.url")
				+ "</a>";
		String portalurl = lhelper.getText(l, "label.email.customer.portalurl")
				+ " " + url;

		Map emailctx = new HashMap();
		emailctx.put("EMAIL_STORE_NAME", store.getStorename());
		emailctx.put("EMAIL_CUSTOMER_PASSWORD", pass);
		emailctx.put("EMAIL_CUSTOMER_PORTAL_INFO", info);
		emailctx.put("EMAIL_CUSTOMER_PORTAL_ENTRY", portalurl);
		emailctx.put("EMAIL_CONTACT_OWNER", store.getStoreemailaddress());

		CommonService cservice = new CommonService();

		cservice.sendHtmlEmail(customer.getCustomerEmailAddress(), subject,
				store, emailctx,
				"email_template_password_reset_customer.ftl", customer
						.getCustomerLang());

		return true;

	}

	@Transactional
	public Customer findCustomerbyUserNameAndPassword(String userName,
			String password, int merchantId) throws Exception {
		return customerDao.findByUserNameAndPasswordByMerchantId(userName,
				password, merchantId);
	}

	@Transactional
	public Customer findCustomerByEmail(String email) {
		return customerDao.findCustomerbyEmail(email);
	}

	@Transactional
	public Customer findCustomerByUserName(String userName, int merchantId) {
		return customerDao.findCustomerbyUserName(userName, merchantId);
	}

	@Transactional
	public Date processLastLoggedInDate(long customerId) {
		Date lastLoggedInDate = null;
		if (customerId != 0) {
			CustomerInfo info = customerInfoDao.findById(customerId);
			if (info != null) {
				lastLoggedInDate = info.getCustomerInfoDateOfLastLogon();
			} else {
				// SET it as current date if it is customer's first login.
				lastLoggedInDate = new Date();
				info = new CustomerInfo();
				info.setCustomerInfoId(customerId);
				info.setCustomerInfoDateAccountCreated(new Date());
			}
			info.setCustomerInfoDateOfLastLogon(new Date());
			info.setCustomerInfoNumberOfLogon(new Integer(((info
					.getCustomerInfoNumberOfLogon() != null) ? info
					.getCustomerInfoNumberOfLogon() : 0) + 1));
			customerInfoDao.saveOrUpdate(info);
		}
		return lastLoggedInDate;
	}

	@Transactional
	public void saveOrUpdateCustomerInfo(CustomerInfo customerInfo) {
		customerInfoDao.saveOrUpdate(customerInfo);
	}

	@Transactional
	public CustomerInfo findCustomerInfoById(long id) {
		return customerInfoDao.findById(id);
	}

}



```
