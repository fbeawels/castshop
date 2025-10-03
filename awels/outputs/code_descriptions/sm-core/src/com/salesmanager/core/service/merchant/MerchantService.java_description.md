# MerchantService.java

## Review

## 1. Summary  

`MerchantService` is a Spring‐managed service that orchestrates all core merchant‑related business logic for the SalesManager platform.  
* **Purpose** – Handle CRUD operations for merchants, users, stores, and configuration settings; provide authentication and role management; expose helper methods for integration modules; and orchestrate the deletion cascade of a merchant’s entire ecosystem (products, orders, customers, etc.).  
* **Key components**  
  * **DAOs** – `IMerchantConfigurationDao`, `IMerchantRegistrationDao`, `IMerchantUserInformationDao`, `IMerchantStoreDao`, `IMerchantIdDao`, `IMerchantUserRoleDao`, `IMerchantUserRoleDefDao` – all injected via `@Autowired`.  
  * **Utility beans** – `AdministrationLogonModule`, `PasswordGeneratorModule`, `CommonService`, `ReferenceService`, `CatalogService`, `OrderService`, `CustomerService`, `TaxService`, etc.  These are obtained either through Spring’s `@Autowired` or via the legacy `SpringUtil.getBean()` and `ServiceFactory.getService()` patterns.  
  * **Constants** – integration type flags and `Configuration` object loaded once at class initialization.  
  * **Transactional boundaries** – Most public methods are annotated with `@Transactional`, ensuring atomic persistence operations.  
* **Design patterns / frameworks** – Spring DI + Transaction management, DAO pattern, Service layer, Utility/Facade modules (`PasswordGeneratorModule`, `AdministrationLogonModule`), and a legacy `ServiceFactory` for accessing other services.  

## 2. Detailed Description  

### Flow of execution  
1. **Startup** – The service is instantiated by Spring. All DAOs are injected, while other utilities are retrieved lazily through `SpringUtil` or `ServiceFactory`.  
2. **Merchant lifecycle** –  
   * **Creation** – `createNewOrSaveMerchant()` handles both new merchant registrations and updates. It creates `MerchantUserInformation`, `MerchantRegistration`, `MerchantStore`, and assigns a default admin role.  
   * **Deletion** – `deleteMerchant()` removes the merchant and cascades deletions across configuration, registration, user accounts, roles, store, geo‑zones, products, categories, orders, customers, tax data, and dynamic labels.  
3. **Authentication** – `adminLogon()` delegates to `AdministrationLogonModule` for authentication.  
4. **Configuration** – Several `getConfiguration*` methods obtain configuration via `MerchantConfigurationImpl`.  
5. **Role & user management** – Methods such as `getUserRoles`, `deleteUserRoles`, `saveOrUpdateRoles`, and `createMerchantUserInformation` handle user roles and info.  
6. **Integration services** – `getModuleServices()` provides a list of shipping or payment module services for a country.  

### Assumptions & constraints  
* The service presumes that all dependent DAOs are correctly configured with a persistence context (JPA/Hibernate).  
* It expects that the underlying database schema supports the cascade deletes (or that explicit deletions in `deleteMerchant()` are sufficient).  
* Email templates and label bundles (`LabelUtil`) are pre‑configured and available.  
* The code assumes that a `MerchantUserInformation` can be identified uniquely by `adminEmail` or `adminName`.  

### Architecture & design choices  
* **Service layer as orchestration hub** – The class mixes pure business logic with data access logic. DAO usage is direct and not abstracted behind a repository interface.  
* **Legacy patterns** – `SpringUtil.getBean()` and `ServiceFactory.getService()` are still used for some dependencies, breaking the consistency of Spring DI.  
* **Manual password handling** – Password generation, encryption, and emailing are performed manually rather than delegating to a dedicated security component.  
* **Hard‑coded strings** – Email keys and text references are hard‑coded, making localisation harder to extend.  

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side‑effects | Notes |
|--------|---------|--------|---------|--------------|-------|
| `getUser(HttpServletRequest)` | Retrieve the current user via logon module | `request` | `String` username | None | Delegates to `AdministrationLogonModule` |
| `getMerchantUserRoleDef()` | Retrieve all role definitions | None | `Collection<MerchantUserRoleDef>` | None | DAO call |
| `createMerchantUserInformation(...)` | Create/initialize a `MerchantUserInformation` record, send intro email, create roles | `merchantId`, `info`, `roles`, `locale` | None | Persists user info, roles, sends email | Duplicated logic in `createNewOrSaveMerchant` |
| `deleteMerchantUserInformation(MerchantUserInformation)` | Delete a user and its roles | `user` | None | Deletes from DB | May throw NPE if `user` null |
| `getMerchantUserInformationByAdminEmail(String)` | Fetch user by admin email | `adminEmail` | `MerchantUserInformation` | None | DAO query |
| `createNewOrSaveMerchant(...)` | Main merchant registration / update routine | `creatorStore`, `merchantUserInfo`, `merchantRegistration` | None | Persists user, registration, store, roles, sends email | Handles both new & existing merchants |
| `deleteMerchant(int)` | Cascade delete of all data for a merchant | `merchantId` | None | Deletes config, registration, users, roles, store, ID, geo‑zones, products, categories, orders, customers, tax | Long method; risk of partial failure |
| `getAllMerchantStores()` | Return a list of store headers for all merchants | None | `List<MerchantStoreHeader>` | None | Uses `merchantUserInformationDao.findByMerchantId` to fill adminName |
| `getUserRoles(String)` | Fetch roles for a user | `userName` | `Collection<MerchantUserRole>` | None | DAO call |
| `deleteUserRoles(String)` | Delete all roles for a user | `userName` | None | DAO delete | |
| `saveOrUpdateRoles(Collection)` | Persist or update roles | `roles` | None | DAO save | |
| `adminLogon(HttpServletRequest, String, String)` | Authenticate a merchant admin | `request`, `user`, `password` | `MerchantUserInformation` | Delegates to logon module | |
| `getConfiguration(ConfigurationRequest)` | Retrieve config for a merchant/module key | `request` | `ConfigurationResponse` | Delegates to `MerchantConfigurationImpl` | |
| `getConfigurationByModule(ConfigurationRequest, String)` | Retrieve all config entries for a merchant/module | `request`, `moduleName` | `ConfigurationResponse` | Delegates to `MerchantConfigurationImpl` | |
| `getMerchantRegistration(int)` | Get registration record | `merchantid` | `MerchantRegistration` | DAO | |
| `getMerchantStore(int)` | Get store record | `merchantid` | `MerchantStore` | DAO | |
| `getMerchantUserInformation(String)` | Get user by admin name | `adminName` | `MerchantUserInformation` | DAO | |
| `getMerchantUserInformation(long)` | Get user by ID | `id` | `MerchantUserInformation` | DAO | |
| `getMerchantInformationByUserNameAndPassword(String, String)` | Authenticate using username/password | `username`, `password` | `MerchantUserInformation` | Encrypted lookup | |
| `getModuleServices(int, String)` | List real‑time shipping or payment services for a country | `integrationtype`, `countryIsoCode` | `List<CoreModuleService>` | Utility call | |
| `getMerchantUserInfo(int)` | Get all user records for a merchant | `merchantId` | `Collection<MerchantUserInformation>` | DAO | |
| `deleteMerchantConfiguration(MerchantConfiguration)` | Delete a single config | `config` | None | DAO | |
| `deleteMerchantConfigurations(Collection)` | Delete multiple configs | `configs` | None | DAO | |
| `cleanConfigurationKeys(String, int)`, `cleanConfigurationKey(String, int)`, `cleanConfigurationLikeKey(String, int)`, `cleanConfigurationLikeKeyModule(String, String, int)` | Delete config(s) matching a key or a LIKE pattern | `key/likeKey`, `merchantid`, optional `moduleid` | None | DAO | |
| `getConfigurationByModule(String, int)` | Retrieve config for a module & merchant ID (overloaded) | `moduleName`, `merchantId` | `ConfigurationResponse` | DAO | |
| `saveOrUpdateMerchantStore(MerchantStore)` | Persist/merge store | `store` | None | DAO | |
| `saveOrUpdateMerchantUserInformation(MerchantUserInformation)` | Persist/merge user info | `info` | None | DAO | |
| `saveOrUpdateMerchantConfiguration(MerchantConfiguration)` | Persist/merge a single config | `configuration` | None | DAO | |
| `saveOrUpdateMerchantConfigurations(List)` | Persist/merge multiple configs | `configurations` | None | DAO | |

### Helper / internal  
* `MerchantUserInformation adminLogon` – delegates to `AdministrationLogonModule`.  
* `ConfigurationResponse getConfigurationByModule(String, int)` – overloaded version that creates `MerchantConfigurationImpl` via `SpringUtil`.  

## 4. Dependencies  

| External libs | Role |
|---------------|------|
| **Spring Framework** (`@Service`, `@Autowired`, `@Transactional`) | DI & transaction mgmt |
| **Apache log4j** (`Logger`) | Logging (only `debug/trace` missing) |
| **Hibernate/JPA** | Persistence (implied by DAOs) |
| **Apache Commons** (`SpringUtil`) | Legacy bean lookup |
| **SalesManager core modules** (`AdministrationLogonModule`, `PasswordGeneratorModule`, `CommonService`, `ReferenceService`, `CatalogService`, `OrderService`, `CustomerService`, `TaxService`) | Orchestrating domain operations |
| **SalesManager configuration** (`Configuration`, `ConfigurationResponse`, `ConfigurationRequest`) | Config management |
| **CoreModuleService** | Shipping/payment integration services |
| **LabelUtil** | Internationalised label lookup |
| **ServicesUtil** | Provides lists of real‑time integration services |

The class mixes `@Autowired` injection with manual bean lookup (`SpringUtil.getBean()` / `ServiceFactory.getService()`), which is a point of architectural inconsistency.

## 5. Additional Notes  

### Strengths  
* **Centralised orchestration** – All merchant‑related operations are in a single place, making it straightforward to trace a flow from registration to configuration.  
* **Transactional safety** – Most data‑changing methods are wrapped in a single transaction.  
* **Extensible configuration API** – The service exposes a thin façade over `MerchantConfigurationImpl`, allowing callers to request configs by key, pattern, or module.  

### Weaknesses / Technical Debt  

| Category | Issue | Impact | Suggested Fix |
|----------|-------|--------|---------------|
| **Coupling & DI** | Mixing `@Autowired` with `SpringUtil.getBean()` and `ServiceFactory.getService()` breaks the Spring‑DI contract. | Harder to test, brittle if the context changes. | Replace all manual lookups with constructor or field injection. |
| **Duplication** | `createMerchantUserInformation` and `createNewOrSaveMerchant` contain identical blocks for password generation, encryption, and email sending. | Code maintenance nightmare, potential for divergence. | Extract a private helper (`persistUserAndRoles(info, roles, locale)`) and reuse it. |
| **Error handling** | Methods wrap generic `Exception` in `MerchantException` or `ServiceException` but do not propagate specific causes (e.g., duplicate email, DAO failures). | Caller loses context; debugging harder. | Provide meaningful messages, use custom exception types, and keep the original cause. |
| **Null‑pointer risk** | Several methods assume non‑null parameters (`merchantUserInformation.getAdminEmail()`, `roles` collection). | Potential NPE at runtime. | Add defensive checks or validate in callers. |
| **Email failures** | `CommonService.sendMail()` is invoked without a try/catch, so an email‑delivery exception would abort the whole transaction. | Partial data persistence. | Isolate email sending in a separate transaction or catch & log the failure. |
| **Static configuration** | `Configuration conf` is loaded once at class init. If the underlying properties change, the service will never see the update. | Stale configuration. | Inject `@ConfigurationProperties` or use `Environment` to reload on demand. |
| **Legacy `ServiceFactory`** | Calls like `ServiceFactory.getService("referenceService")` bypass Spring’s transaction demarcation. | Inconsistent transaction boundaries, harder to unit‑test. | Autowire the referenced services directly. |
| **Transactional boundaries for reads** | Most read methods (`getAllMerchantStores`, `getModuleServices`, etc.) are transactional but not marked as `readOnly = true`. | Unnecessary write locks, performance overhead. | Add `@Transactional(readOnly = true)` where appropriate. |
| **Type safety** | Use of raw types (`Collection`, `List`) and casting (`(List)userInfo`). | Compile‑time warnings, possible `ClassCastException`. | Use generics (`List<MerchantUserInformation>`) everywhere. |
| **Hard‑coded keys** | Email and label keys (`"merchant.intro.email.title"`, `"password_reset"`, etc.) are string literals. | Localisation harder to manage. | Expose these as constants or use a dedicated i18n component. |
| **Password encryption** | Manual key generation with `SecurityConstants.idConstant` every time. | Security risk if key changes or is shared. | Centralise encryption in a dedicated security service. |
| **Long delete method** | `deleteMerchant` performs 20+ DAO deletes. | Risk of partial failure, transaction rollback may be expensive. | Break into smaller transactional helper methods or use database cascade rules. |
| **Logging** | Only a single logger instance, no entry/exit logs or detailed error messages. | Hard to trace issues in production. | Add structured logging (method name, params) and log exceptions with stack traces. |

### Suggested Improvements  

1. **Modernize DI** – Replace `SpringUtil.getBean()` and `ServiceFactory.getService()` with constructor injection.  
2. **Extract services** – Move email sending and password handling to dedicated services (`MailService`, `PasswordService`) so `MerchantService` only orchestrates.  
3. **Reduce duplication** – Merge user‑creation logic into a shared private method.  
4. **Enhance transaction semantics** – Use `@Transactional(readOnly = true)` for fetch operations; isolate write‑heavy operations into smaller transactional units.  
5. **Improve error handling** – Throw specific custom exceptions (`DuplicateMerchantException`, `EmailSendFailedException`, etc.) and surface meaningful error messages.  
6. **Strengthen type safety** – Replace raw collections and casts with generics.  
7. **Add defensive programming** – Validate inputs, guard against nulls, and return informative messages.  
8. **Logging** – Add entry/exit logs and error logs with contextual data (merchantId, userName, etc.).  

### Edge Cases & Risks  

| Scenario | Risk | Mitigation |
|----------|------|------------|
| Registering a merchant with an existing `adminEmail` | Duplicate entry error | Check via `getMerchantUserInformationByAdminEmail` before persisting. |
| Deleting a merchant while other processes are creating or updating its entities | Inconsistent state | Ensure transaction isolation or use DB cascade deletes. |
| Email templates missing or label bundles not loaded | `NullPointerException` at email send | Validate template existence; fallback to default message. |
| Role collection `roles` is null or empty | No roles assigned, but later role checks fail | Default to empty list; guard against `NullPointerException`. |
| Concurrent updates to same merchant | Lost update / dirty data | Use optimistic locking (`@Version`) or synchronized blocks. |

---

### Bottom line  

`MerchantService` is functional and covers the breadth of merchant management, but it suffers from a mix of legacy and modern patterns, duplicated logic, and brittle error handling. A refactor toward a cleaner service‑repository separation, full use of Spring DI, and stricter transaction semantics would reduce complexity, improve testability, and increase robustness.

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
package com.salesmanager.core.service.merchant;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.HashMap;
import java.util.List;
import java.util.Locale;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;

import org.apache.commons.configuration.Configuration;
import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.salesmanager.core.constants.ErrorConstants;
import com.salesmanager.core.constants.SecurityConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantRegistration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantStoreHeader;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.merchant.MerchantUserRole;
import com.salesmanager.core.entity.merchant.MerchantUserRoleDef;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.MerchantId;
import com.salesmanager.core.module.model.application.AdministrationLogonModule;
import com.salesmanager.core.module.model.application.PasswordGeneratorModule;
import com.salesmanager.core.service.ServiceException;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.catalog.CatalogService;
import com.salesmanager.core.service.common.CommonService;
import com.salesmanager.core.service.common.impl.ServicesUtil;
import com.salesmanager.core.service.customer.CustomerService;
import com.salesmanager.core.service.merchant.impl.MerchantConfigurationImpl;
import com.salesmanager.core.service.merchant.impl.dao.IMerchantConfigurationDao;
import com.salesmanager.core.service.merchant.impl.dao.IMerchantIdDao;
import com.salesmanager.core.service.merchant.impl.dao.IMerchantRegistrationDao;
import com.salesmanager.core.service.merchant.impl.dao.IMerchantStoreDao;
import com.salesmanager.core.service.merchant.impl.dao.IMerchantUserInformationDao;
import com.salesmanager.core.service.merchant.impl.dao.IMerchantUserRoleDao;
import com.salesmanager.core.service.merchant.impl.dao.IMerchantUserRoleDefDao;
import com.salesmanager.core.service.merchant.impl.dao.MerchantUserRoleDefDao;
import com.salesmanager.core.service.order.OrderService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.service.tax.TaxService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.EncryptionUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LocaleUtil;
import com.salesmanager.core.util.PropertiesUtil;
import com.salesmanager.core.util.ReferenceUtil;
import com.salesmanager.core.util.SpringUtil;

@Service
public class MerchantService {

	public final static int INTEGRATION_TYPE_SHIPPING = 1;
	public final static int INTEGRATION_TYPE_PAYMENT = 2;

	private Logger log = Logger.getLogger(MerchantService.class);
	private static Configuration conf = PropertiesUtil.getConfiguration();

	@Autowired
	private IMerchantConfigurationDao merchantConfigurationDao;

	@Autowired
	private IMerchantRegistrationDao merchantRegistrationDao;

	@Autowired
	private IMerchantUserInformationDao merchantUserInformationDao;

	@Autowired
	private IMerchantStoreDao merchantStoreDao;

	@Autowired
	private IMerchantIdDao merchantIdDao;

	@Autowired
	private IMerchantUserRoleDao merchantUserRoleDao;
	
	@Autowired
	private IMerchantUserRoleDefDao merchantUserRoleDefDao;

	public String getUser(HttpServletRequest request) throws MerchantException {


		try {


			AdministrationLogonModule module = (AdministrationLogonModule) SpringUtil
					.getBean("merchantLogon");

			return module.getUser(request);
		} catch (Exception e) {
			throw new MerchantException(e);
		}
	}
	
	
	@Transactional
	public Collection<MerchantUserRoleDef> getMerchantUserRoleDef() throws Exception {
		return merchantUserRoleDefDao.findAll();
	}
	
	/**
	 * Creates a basic merchant id
	 * first name
	 * last name
	 * email
	 * admin name
	 * password
	 * Sends an email
	 * @param merchantId
	 * @param merchantUserInformation
	 * @param locale
	 */
	@Transactional
	public void createMerchantUserInformation(int merchantId, MerchantUserInformation merchantUserInformation, Collection<MerchantUserRole> roles, Locale locale) throws Exception {
		
		
		if(merchantUserInformation==null) {
			merchantUserInformation = new MerchantUserInformation();
		}
		merchantUserInformation.setMerchantId(merchantId);
		merchantUserInformation.setLastModified(new Date());
		merchantUserInformation.setUserlang(locale.getLanguage());

		PasswordGeneratorModule passwordGenerator = (PasswordGeneratorModule) SpringUtil
				.getBean("passwordgenerator");
		String key = EncryptionUtil.generatekey(String
				.valueOf(SecurityConstants.idConstant));
		String encrypted;
		String password = "";
		try {
			password = passwordGenerator.generatePassword();
			encrypted = EncryptionUtil.encrypt(key, password);
		} catch (Exception e) {
			log.error(e);
			throw new ServiceException(e);

		}
		merchantUserInformation.setAdminPass(encrypted);
		

		
		MerchantStore store = this.getMerchantStore(merchantId);
		
		merchantUserInformation.setUseraddress(store.getStoreaddress());
		merchantUserInformation.setUsercity(store.getStorecity());
		merchantUserInformation.setUsercountrycode(store.getCountry());
		merchantUserInformation.setUserphone(store.getStorephone());
		merchantUserInformation.setUserpostalcode(store.getStorepostalcode());
		merchantUserInformation.setUserstate(store.getZone());
		
		this.saveOrUpdateMerchantUserInformation(merchantUserInformation);
		
		// send an introduction email
		LabelUtil lhelper = LabelUtil.getInstance();
		String title = lhelper.getText(merchantUserInformation.getUserlang(),
				"label.profile.newmerchantemailtitle");
		String adminInfo = lhelper.getText(merchantUserInformation.getUserlang(),
				"label.profile.userinformation");
		String username = lhelper.getText(merchantUserInformation.getUserlang(),
				"username");
		String pwd = lhelper.getText(merchantUserInformation.getUserlang(),
				"password");
		String url = lhelper.getText(merchantUserInformation.getUserlang(),
				"label.profile.adminurl");
		String mailTitle = lhelper.getText(merchantUserInformation.getUserlang(),
		"label.profile.newusertitle");


		Map context = new HashMap();
		context.put("EMAIL_NEW_USER_TEXT", title);
		context.put("EMAIL_STORE_NAME", store.getStorename());
		context.put("EMAIL_ADMIN_LABEL", adminInfo);
		context.put("EMAIL_CUSTOMER_FIRSTNAME", merchantUserInformation
				.getUserfname());
		context.put("EMAIL_CUSTOMER_LAST", merchantUserInformation.getUserlname());
		context.put("EMAIL_ADMIN_NAME", merchantUserInformation.getAdminName());
		context.put("EMAIL_ADMIN_PASSWORD", password);
		context.put("EMAIL_ADMIN_USERNAME_LABEL", username);
		context.put("EMAIL_ADMIN_PASSWORD_LABEL", pwd);
		context.put("EMAIL_ADMIN_URL_LABEL", url);
		context.put("EMAIL_ADMIN_URL", ReferenceUtil
				.buildCentralUri(store));

		this.saveOrUpdateRoles(roles);

		String email = merchantUserInformation.getAdminEmail();

		CommonService cservice = new CommonService();
		cservice.sendHtmlEmail(email, mailTitle,
				store, context, "email_template_new_user.ftl",
				merchantUserInformation.getUserlang());
		
	}
	
	
	/**
	 * Deletes a MerchantUserInformation entity and roles attached
	 * @param merchantUserId
	 */
	@Transactional
	public void deleteMerchantUserInformation(MerchantUserInformation user) {
	
		merchantUserRoleDao.deleteByUserName(user.getAdminName());
		
		merchantUserInformationDao.delete(user);
		
		
		
	}
	
	/**
	 * Returns a MerchantUserInformation based on the administration email
	 * @param adminEmail
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public MerchantUserInformation getMerchantUserInformationByAdminEmail(String adminEmail) throws Exception {
		return merchantUserInformationDao.findByAdminEmail(adminEmail);
	}

	@Transactional
	public void createNewOrSaveMerchant(MerchantStore creatorStore,
			MerchantUserInformation merchantUserInfo,
			MerchantRegistration merchantRegistration) throws Exception {
		if (merchantUserInfo.getMerchantId() == 0) {
			// New Merchant User

			if (merchantUserInformationDao.findByAdminEmail(merchantUserInfo
					.getAdminEmail()) != null) {
				throw new ServiceException("Admin Email Already Exists",
						ErrorConstants.EMAIL_ALREADY_EXISTS);
			}
			
			
			String language = conf.getString("core.system.defaultlanguage");
			if(!StringUtils.isBlank(merchantUserInfo.getUserlang())) {
				language = merchantUserInfo.getUserlang();
			} else {
				merchantUserInfo.setUserlang(language);
			}

			
			Locale locale = LocaleUtil.getLocale(language);

			MerchantId merchantId = new MerchantId(0, new Date());
			Integer newMerchantId = merchantIdDao.saveMerchantId(merchantId);

			merchantUserInfo.setMerchantId(newMerchantId);
			merchantUserInfo.setLastModified(new Date());


			PasswordGeneratorModule passwordGenerator = (PasswordGeneratorModule) SpringUtil
					.getBean("passwordgenerator");
			String key = EncryptionUtil.generatekey(String
					.valueOf(SecurityConstants.idConstant));
			String encrypted;
			String password = "";
			try {
				password = passwordGenerator.generatePassword();
				encrypted = EncryptionUtil.encrypt(key, password);
			} catch (Exception e) {
				log.error(e);
				throw new ServiceException(e);

			}
			merchantUserInfo.setAdminPass(encrypted);
			// merchantUserInfo.setAdminPass(password);

			merchantUserInformationDao.persist(merchantUserInfo);

			merchantRegistration.setMerchantId(newMerchantId);
			merchantRegistration.setDateAdded(new Date());
			merchantRegistration.setLastModified(new Date());
			merchantRegistration.setPromoCode(0);
			merchantRegistration.setPromoCodeExpiry(new Date());
			merchantRegistrationDao.persist(merchantRegistration);

			// create a MerchantStore
			MerchantStore mStore = new MerchantStore();
			mStore.setMerchantId(newMerchantId);
			mStore.setStorename(merchantUserInfo.getUserfname() + " "
					+ merchantUserInfo.getUserlname());
			mStore.setCountry(merchantUserInfo.getUsercountrycode());
			mStore.setZone(merchantUserInfo.getUserstate());
			mStore.setCurrency(CurrencyUtil.getDefaultCurrency());
			mStore.setTemplateModule("decotemplate");
			merchantStoreDao.saveOrUpdate(mStore);

			// send an introduction email
			LabelUtil lhelper = LabelUtil.getInstance();
			String title = lhelper.getText(merchantUserInfo.getUserlang(),
					"label.profile.newmerchantemailtitle");
			String adminInfo = lhelper.getText(merchantUserInfo.getUserlang(),
					"label.profile.userinformation");
			String username = lhelper.getText(merchantUserInfo.getUserlang(),
					"username");
			String pwd = lhelper.getText(merchantUserInfo.getUserlang(),
					"password");
			String url = lhelper.getText(merchantUserInfo.getUserlang(),
					"label.profile.adminurl");
			String mailTitle = lhelper.getText(merchantUserInfo.getUserlang(),
					"label.profile.newstoretitle");

			Map context = new HashMap();
			context.put("EMAIL_NEW_STORE_TEXT", title);
			context.put("EMAIL_STORE_NAME", mStore.getStorename());
			context.put("EMAIL_ADMIN_LABEL", adminInfo);
			context.put("EMAIL_CUSTOMER_FIRSTNAME", merchantUserInfo
					.getUserfname());
			context.put("EMAIL_CUSTOMER_LAST", merchantUserInfo.getUserlname());
			context.put("EMAIL_ADMIN_NAME", merchantUserInfo.getAdminName());
			context.put("EMAIL_ADMIN_PASSWORD", password);
			context.put("EMAIL_ADMIN_USERNAME_LABEL", username);
			context.put("EMAIL_ADMIN_PASSWORD_LABEL", pwd);
			context.put("EMAIL_ADMIN_URL_LABEL", url);
			context.put("EMAIL_ADMIN_URL", ReferenceUtil
					.buildCentralUri(mStore));

			// create role
			MerchantUserRole role = new MerchantUserRole();
			role.setAdminName(merchantUserInfo.getAdminName());
			role.setRoleCode(SecurityConstants.ADMINISTRATOR);
			merchantUserRoleDao.save(role);

			String email = merchantUserInfo.getAdminEmail();

			CommonService cservice = new CommonService();
			cservice.sendHtmlEmail(email, mailTitle,
					creatorStore, context, "email_template_new_store.ftl",
					merchantUserInfo.getUserlang());

		} else {
			MerchantUserInformation existingUserInfo = merchantUserInformationDao
					.findById(merchantUserInfo.getMerchantUserId().intValue());
			existingUserInfo.setLastModified(new Date());
			existingUserInfo.setAdminEmail(merchantUserInfo.getAdminEmail());
			existingUserInfo.setAdminName(merchantUserInfo.getAdminName());
			existingUserInfo.setUserfname(merchantUserInfo.getUserfname());
			existingUserInfo.setUserlname(merchantUserInfo.getUserlname());
			existingUserInfo.setUseraddress(merchantUserInfo.getUseraddress());
			existingUserInfo.setUserphone(merchantUserInfo.getUserphone());
			existingUserInfo.setUsercity(merchantUserInfo.getUsercity());
			existingUserInfo.setUserpostalcode(merchantUserInfo
					.getUserpostalcode());
			existingUserInfo.setUserstate(merchantUserInfo.getUserstate());
			existingUserInfo.setUsercountrycode(merchantUserInfo
					.getUsercountrycode());
			existingUserInfo.setUserlang(merchantUserInfo.getUserlang());
			merchantUserInformationDao.persist(existingUserInfo);

			MerchantRegistration existingMerchantReg = merchantRegistrationDao
					.findByMerchantId(merchantUserInfo.getMerchantId());
			existingMerchantReg.setLastModified(new Date());
			existingMerchantReg
					.setMerchantRegistrationDefCode(merchantRegistration
							.getMerchantRegistrationDefCode());
			merchantRegistrationDao.persist(existingMerchantReg);
		}
	}

	@Transactional
	public void deleteMerchant(int merchantId) throws Exception {
		MerchantConfiguration config = new MerchantConfiguration();
		config.setMerchantId(merchantId);
		merchantConfigurationDao.delete(config);

		MerchantRegistration merchantReg = merchantRegistrationDao
				.findByMerchantId(merchantId);
		if (merchantReg != null) {
			merchantRegistrationDao.delete(merchantReg);
		}

		Collection<MerchantUserInformation> merchantUsers = merchantUserInformationDao
				.findByMerchantId(merchantId);
		if (merchantUsers != null) {
			merchantUserInformationDao.deleteAll(merchantUsers);
		}

		MerchantStore store = merchantStoreDao.findByMerchantId(merchantId);
		if (store != null) {
			merchantStoreDao.delete(store);
		}

		MerchantId merchant = merchantIdDao.findById(merchantId);
		if (merchant != null) {
			merchantIdDao.delete(merchant);
		}

		// delete roles
		
		for(Object o : merchantUsers) {
			
			MerchantUserInformation userInfo  = (MerchantUserInformation)o;
			merchantUserRoleDao.deleteByUserName(userInfo.getAdminName());
		}
		
		

		// delete merchant configuration
		Collection configs = merchantConfigurationDao
				.findListMerchantId(merchantId);
		if (configs != null && configs.size() > 0) {
			merchantConfigurationDao.delete(configs);
		}

		// delete geo zones
		// delete zones to geozones
		// dynamic label

		ReferenceService referenceService = (ReferenceService) ServiceFactory
				.getService(ServiceFactory.ReferenceService);
		// referenceService.deleteGeoZones(merchantId);
		referenceService.deleteAllDynamicLabel(merchantId);

		// delete products
		// delete categories

		CatalogService cservice = (CatalogService) ServiceFactory
				.getService(ServiceFactory.CatalogService);
		cservice.deleteAllProducts(merchantId);
		cservice.deleteAllCategories(merchantId);

		// delete orders
		// delete orders_account

		OrderService oservice = (OrderService) ServiceFactory
				.getService(ServiceFactory.OrderService);
		oservice.deleteAllOrders(merchantId);

		// delete customers
		CustomerService custservice = (CustomerService) ServiceFactory
				.getService(ServiceFactory.CustomerService);
		custservice.deleteAllCustomers(merchantId);

		// tax class
		// tax rates
		TaxService taxService = (TaxService) ServiceFactory
				.getService(ServiceFactory.TaxService);
		taxService.deleteTaxConfiguration(merchantId);

	}

	@Transactional
	public List<MerchantStoreHeader> getAllMerchantStores() {
		List<MerchantStoreHeader> merchantStoreList = new ArrayList<MerchantStoreHeader>();
		MerchantStoreHeader header = null;
		// List<MerchantId> merchantIdList = merchantIdDao.loadAll();
		List<MerchantStore> storeList = merchantStoreDao.loadAll();
		for (MerchantStore store : storeList) {
			header = new MerchantStoreHeader();
			header.setMerchantId(store.getMerchantId());
			header.setAdminEmail(store.getStoreemailaddress());
			Collection<MerchantUserInformation> userInfo = merchantUserInformationDao.findByMerchantId(store.getMerchantId());
			if (userInfo != null && userInfo.size()>0) {
				//header.setAdminEmail(userInfo.getAdminEmail());
				MerchantUserInformation mUserInfo = (MerchantUserInformation)((List)userInfo).get(0);
				header.setAdminName(mUserInfo.getAdminName());
			}
			header.setStorename(store.getStorename());
			merchantStoreList.add(header);
		}
		return merchantStoreList;
	}

	/**
	 * Return a collection of roles for a given user
	 * 
	 * @param request
	 * @param role
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Collection<MerchantUserRole> getUserRoles(String userName)
			throws Exception {
		return merchantUserRoleDao.findByUserName(userName);
	}
	
	/**
	 * Deletes all user roles
	 * @param userName
	 * @throws Exception
	 */
	@Transactional
	public void deleteUserRoles(String userName) throws Exception {
			merchantUserRoleDao.deleteByUserName(userName);
	}
	
	@Transactional
	public void saveOrUpdateRoles(Collection<MerchantUserRole> roles) throws Exception {
			merchantUserRoleDao.saveOrUpdateAll(roles);
	}

	public MerchantUserInformation adminLogon(HttpServletRequest request, String user, String password)
			throws ServiceException {

		Class clz = null;

		try {

			AdministrationLogonModule module = (AdministrationLogonModule) SpringUtil
					.getBean("merchantLogon");

			return module.logon(request, user, password);

		} catch (Exception e) {
			if (e instanceof ServiceException) {
				throw (ServiceException) e;
			}
			throw new ServiceException(e);
		}

	}

	/**
	 * Get configuration for a given merchant ConfigurationRequestVO constructor
	 * accept: int merchantId, String configurationKey and int merchantId,
	 * boolean like, String configurationKey if boolean like=true then a request
	 * is made for configuration key like %configurationKey% and int merchnatId
	 * 
	 * @param ConfigurationRequest
	 *            request
	 * @return ConfigurationVO response
	 * @throws MerchantException
	 */
	public ConfigurationResponse getConfiguration(ConfigurationRequest request)
			throws MerchantException {

		try {

			MerchantConfigurationImpl impl = (MerchantConfigurationImpl) SpringUtil
					.getBean("merchantConfigurationImpl");
			ConfigurationResponse vo = impl.getConfigurationVO(request);
			return vo;
		} catch (Exception e) {
			throw new MerchantException(e);
		}
	}

	/**
	 * Get configurations for a given module for a given merchant
	 * ConfigurationRequest requires a merchantId
	 * 
	 * @param request
	 * @return
	 * @throws MerchantException
	 */

	public ConfigurationResponse getConfigurationByModule(
			ConfigurationRequest request, String moduleName)
			throws MerchantException {

		try {

			MerchantConfigurationImpl impl = (MerchantConfigurationImpl) SpringUtil
					.getBean("merchantConfigurationImpl");
			ConfigurationResponse vo = impl.getConfigurationVOByModule(request,
					moduleName);
			return vo;
		} catch (Exception e) {
			throw new MerchantException(e);
		}
	}

	/**
	 * Returns the registration details for a given account
	 * 
	 * @param merchantid
	 * @return
	 * @throws MerchantException
	 */
	@Transactional
	public MerchantRegistration getMerchantRegistration(int merchantid)
			throws MerchantException {
		return merchantRegistrationDao.findByMerchantId(merchantid);
	}

	/**
	 * Retreives MerchantProfile Entity Object based on the merchant id
	 * 
	 * @param merchantid
	 * @return MerchantProfile or null
	 * @throws MerchantException
	 */
	// @Transactional
	// public MerchantProfile getMerchantProfile(int merchantid) throws
	// MerchantException {

	// return merchantProfileDao.findById(merchantid);

	// }

	@Transactional
	public MerchantStore getMerchantStore(int merchantid)
			throws MerchantException {
		return merchantStoreDao.findByMerchantId(merchantid);
	}

	/**
	 * Retreives MerchantUserInformation Entity Object based on the
	 * administration name
	 * 
	 * @param merchantid
	 * @return MerchantProfile or null
	 * @throws MerchantException
	 */
	@Transactional
	public MerchantUserInformation getMerchantUserInformation(String adminName)
			throws Exception {
		return merchantUserInformationDao.findByUserName(adminName);
	}
	
	/**
	 * 
	 * @param long merchantUserInformationId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public MerchantUserInformation getMerchantUserInformation(long merchantUserInformationId)
			throws Exception {
		return merchantUserInformationDao.findById(merchantUserInformationId);
	}

	// @Transactional
	// public MerchantProfile getMerchantProfile(String adminName) throws
	// MerchantException {

	// return merchantProfileDao.findByAdminName(adminName);

	// }

	/**
	 * Get MerchantUserInformation by username and password. This is used for
	 * custom authentication
	 * 
	 * @param username
	 * @param password
	 * @return
	 */
	@Transactional
	public MerchantUserInformation getMerchantInformationByUserNameAndPassword(
			String username, String password) throws Exception {

		// encrypt password
		String key = EncryptionUtil.generatekey(String
				.valueOf(SecurityConstants.idConstant));
		String enc = EncryptionUtil.encrypt(key, password);

		return merchantUserInformationDao.findByUserNameAndPassword(username,
				enc);

	}

	/**
	 * Returns a list of CentralIntegrationServices
	 * 
	 * @param integrationtype
	 * @param countryid
	 * @return
	 * @throws MerchantException
	 */
	public List<CoreModuleService> getModuleServices(int integrationtype,
			String countryIsoCode) throws MerchantException {

		switch (integrationtype) {
		case INTEGRATION_TYPE_SHIPPING:// shipping
			return ServicesUtil
					.getShippingRealTimeQuotesMethods(countryIsoCode);

		case INTEGRATION_TYPE_PAYMENT:// shipping

			return ServicesUtil.getPaymentMethodsList(countryIsoCode);
		}

		return null;

	}

	/**
	 * Will return a MerchantUserInformation for a given merchantId. There can
	 * be more than one MerchantUserInformation entity per merchantId, so it
	 * will return the latest...
	 * 
	 * @param merchantId
	 * @return
	 */
	@Transactional
	public Collection<MerchantUserInformation> getMerchantUserInfo(int merchantId) {
		return merchantUserInformationDao.findByMerchantId(merchantId);
	}

	/**
	 * Removes a MerchantConfiguration entity
	 * 
	 * @param config
	 * @throws MerchantException
	 */
	@Transactional
	public void deleteMerchantConfiguration(MerchantConfiguration config)
			throws MerchantException {

		merchantConfigurationDao.delete(config);

	}

	/**
	 * Removes a collection of MerchantConfiguration
	 * 
	 * @param configs
	 * @throws MerchantException
	 */
	@Transactional
	public void deleteMerchantConfigurations(
			Collection<MerchantConfiguration> configs) throws MerchantException {
		merchantConfigurationDao.delete(configs);
	}

	@Transactional
	public void cleanConfigurationKeys(String likeKey, int merchantid)
			throws MerchantException {

		merchantConfigurationDao.deleteLike(likeKey, merchantid);

	}

	@Transactional
	public void cleanConfigurationKey(String key, int merchantid)
			throws MerchantException {

		merchantConfigurationDao.deleteKey(key, merchantid);

	}

	@Transactional
	public void cleanConfigurationLikeKey(String likeKey, int merchantid)
			throws MerchantException {

		merchantConfigurationDao.deleteLike(likeKey, merchantid);

	}

	@Transactional
	public void cleanConfigurationLikeKeyModule(String likeKey,
			String moduleid, int merchantid) throws MerchantException {

		merchantConfigurationDao
				.deleteLikeModule(likeKey, moduleid, merchantid);

	}

	/**
	 * Returns a ConfigurationVO object for a given module / merchantId
	 * 
	 * @param moduleName
	 * @param merchantId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public ConfigurationResponse getConfigurationByModule(String moduleName,
			int merchantId) throws Exception {

		MerchantConfigurationImpl impl = (MerchantConfigurationImpl) SpringUtil
				.getBean("merchantConfigurationImpl");
		ConfigurationResponse vo = impl.getConfigurationVO(moduleName,
				merchantId);
		return vo;

	}

	@Transactional
	public void saveOrUpdateMerchantStore(MerchantStore store) throws Exception {
		merchantStoreDao.saveOrUpdate(store);
	}

	@Transactional
	public void saveOrUpdateMerchantUserInformation(MerchantUserInformation info)
			throws Exception {
		this.merchantUserInformationDao.saveOrUpdate(info);
	}

	@Transactional
	public void saveOrUpdateMerchantConfiguration(
			MerchantConfiguration configuration) throws MerchantException {

		merchantConfigurationDao.saveOrUpdate(configuration);
	}

	@Transactional
	public void saveOrUpdateMerchantConfigurations(
			List<MerchantConfiguration> configurations)
			throws MerchantException {

		if (configurations != null) {
			merchantConfigurationDao.saveOrUpdateAll(configurations);

		}

	}

}



```
