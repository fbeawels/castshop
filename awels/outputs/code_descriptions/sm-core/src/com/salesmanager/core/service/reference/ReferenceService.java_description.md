# ReferenceService.java

## Review

## 1. Summary  

**Purpose**  
`ReferenceService` is a Spring‑managed service that centralises access to a wide range of lookup/reference data used by the SalesManager platform.  The service exposes CRUD and query operations for entities such as **Country, Zone, Currency, OrderStatus, DynamicLabel, ModuleConfiguration, Page, Portlet**, etc.  Most methods are thin wrappers over DAO calls, occasionally performing a few orchestration steps (e.g. deleting related portlet configurations when a portlet is removed).

**Key components**

| Component | Role |
|-----------|------|
| **DAO injection** (`@Autowired`) | All data access is delegated to DAO objects (Hibernate or JPA). |
| **Transactional boundaries** | Methods are annotated with `@Transactional`; most are read‑only but not explicitly marked so. |
| **Reference lookup helpers** | Methods such as `getProductTypes()`, `getCountries()`, `getCurrencies()` provide convenient access to core lookup tables. |
| **Dynamic label handling** | CRUD + bulk operations for `DynamicLabel` and `DynamicLabelDescription`. |
| **Module/Portlets** | Helpers to manage core module services, module configurations, pages, and portlets. |
| **Utility calls** | Uses `CountryUtil`, `LanguageUtil`, `ConfigurationFieldUtil`, and `ServicesUtil` for helper logic. |

The code relies on Spring (transactional support, dependency injection) and a custom DAO layer.  No major design pattern beyond DAO + service is employed, but the service layer mixes data access, orchestration, and business‑logic responsibilities in a single large class.

---

## 2. Detailed Description  

### 2.1 Flow of execution  

1. **Service discovery** – Spring creates the singleton `ReferenceService` and injects all DAOs.  
2. **Request handling** – The service is called by controllers or other services.  
3. **Transaction** – `@Transactional` starts a transaction. Most methods are read‑only but lack the `readOnly = true` hint, which could lead to unnecessary write locks.  
4. **DAO delegation** – The method delegates to the appropriate DAO, sometimes combining results (e.g. `getDynamicLabelsByIds`).  
5. **Orchestration** – In a few cases the service performs additional logic:
   * **Deletion of dynamic labels** – removes label descriptions and portlet references.  
   * **Portlet deletion** – removes the portlet and its associated merchant configuration.  
6. **Cleanup** – Transaction is committed or rolled back automatically by Spring.  

### 2.2 Architecture & Design Choices  

* **Thin Service Layer** – The service mostly forwards calls to DAOs, with minimal business logic.  
* **No Interfaces** – The class is concrete; no `ReferenceService` interface is defined, which limits testability and polymorphism.  
* **Legacy Generics** – Many methods return raw `Collection`, `Map`, or use `Iterator` instead of generics, compromising type safety.  
* **Manual DAO Queries** – The DAO layer presumably implements custom queries; the service never re‑implements any filtering logic.  
* **Mixed Read/Write** – All `@Transactional` methods are identical; read‑only methods are not flagged as such.  
* **Utility Coupling** – Direct usage of static helper classes (`ServiceFactory`, `ConfigurationFieldUtil`) inside the service tightens coupling and hampers unit testing.

### 2.3 Assumptions & Constraints  

* The DAOs are correctly implemented and provide the necessary `findBy...`, `saveOrUpdate`, `delete`, `deleteAll` operations.  
* All entities are Hibernate/JPA managed and properly mapped.  
* The `ServiceFactory` static lookup works in the current deployment environment.  
* The caller guarantees that the provided `Locale` objects are non‑null and supported.  
* The code assumes that `CountryUtil.getCountryIsoCodeById` returns a valid ISO code; a missing country will lead to a `NullPointerException`.

---

## 3. Functions/Methods  

Below is a high‑level description of each public method.  Parameters, return types, and side‑effects are highlighted.

| Method | Purpose | Inputs | Outputs | Side‑Effects |
|--------|---------|--------|---------|--------------|
| `Collection<ProductType> getProductTypes()` | Retrieve all product types. | – | List of `ProductType` | None |
| `Map getSupportedCreditCards()` | Return map of credit card ID → entity. | – | `Map<Integer, CreditCard>` | None |
| `Collection<Country> getCountries()` | All countries. | – | `Collection<Country>` | None |
| `Collection<Language> getLanguages()` | All languages. | – | `Collection<Language>` | None |
| `Collection<Zone> getZones()` | All zones. | – | `Collection<Zone>` | None |
| `Collection<CentralMeasureUnits> getMeasureUnits()` | All measurement units. | – | `Collection<CentralMeasureUnits>` | None |
| `Collection<Currency> getCurrencies()` | All currencies. | – | `Collection<Currency>` | None |
| `Collection<OrderStatus> getOrderStatus()` | All order status values. | – | `Collection<OrderStatus>` | None |
| `Collection<CentralCountryStatus> getCountryStatus()` | All shipping country status. | – | `Collection<CentralCountryStatus>` | None |
| `CountryDescription getCountryDescriptionByIsoCode(String isocode, int languageId)` | Lookup by ISO code + language. | ISO code, language ID | `CountryDescription` | None |
| `CountryDescription getCountryDescriptionByCountryId(int countryId, int languageId)` | Lookup by country ID + language. | ID, language ID | `CountryDescription` | None |
| `CountryDescription getCountryDescriptionByName(String name, int languageId)` | Lookup by name + language. | Name, language ID | `CountryDescription` | None |
| `Country getCountryByName(String name, int languageId)` | Country by name & language. | Name, language ID | `Country` | None |
| `Country getCountryByIsoCode(String isocode)` | Country by ISO code. | ISO code | `Country` | None |
| `Zone getZoneByIsoCode(String isocode, int languageId)` | Zone by ISO + language. | ISO, language ID | `Zone` | None |
| `Zone getZoneByName(String name, int languageId)` | Zone by name + language. | Name, language ID | `Zone` | None |
| `void deleteGeoZones(int merchantId)` | Deletes all geo‑zones & zone‑to‑geo relations for a merchant. | Merchant ID | None |
| `Collection<DynamicLabel> getDynamicLabels(int merchantId, int sectionId)` | All dynamic labels for a merchant & section. | Merchant ID, section | `Collection<DynamicLabel>` | None |
| `Collection<DynamicLabel> getDynamicLabels(int merchantId, int sectionId, Locale locale)` | Same as above + language. | Merchant ID, section, Locale | `Collection<DynamicLabel>` | None |
| `Collection<DynamicLabel> getDynamicLabels(int merchantId, List<Integer> sections, Locale locale)` | Multi‑section query. | Merchant ID, section list, Locale | `Collection<DynamicLabel>` | None |
| `Collection<DynamicLabel> getDynamicLabelsByIds(int merchantId, List<Long> ids, Locale locale)` | Query by label IDs. | Merchant ID, ID list, Locale | `Collection<DynamicLabel>` | None |
| `Collection<DynamicLabel> getDynamicLabelsByTitles(int merchantId, List<String> ids, Locale locale)` | Query by title list. | Merchant ID, title list, Locale | `Collection<DynamicLabel>` | None |
| `DynamicLabel getDynamicLabel(long id)` | Find label by ID. | ID | `DynamicLabel` | None |
| `Collection<DynamicLabel> getDynamicLabels(int merchantId)` | All dynamic labels for a merchant. | Merchant ID | `Collection<DynamicLabel>` | None |
| `Collection<DynamicLabel> getDynamicLabelsByLanguage(int merchantId, String language)` | Labels for a language. | Merchant ID, language code | `Collection<DynamicLabel>` | None |
| `DynamicLabel getDynamicLabelByMerchantIdAndSeUrlAndLanguageId(int merchantId, String url, Locale locale)` | Find label by URL and language. | Merchant ID, URL, Locale | `DynamicLabel` | None |
| `void deleteAllDynamicLabel(int merchantId)` | Delete all labels (and descriptions, portlets) for a merchant. | Merchant ID | None |
| `void deleteAllDynamicLabel(Collection<DynamicLabel> labels)` | Delete given label collection. | Label collection | None |
| `void deleteDynamicLabel(DynamicLabel label)` | Delete a single label (and its portlet references). | `DynamicLabel` | None |
| `DynamicLabel getDynamicLabel(int merchantId, String title, Locale locale)` | Find by merchant + title + language. | Merchant ID, title, Locale | `DynamicLabel` | None |
| `DynamicLabelDescription getDynamicLabelDescription(int merchantId, int sectionId, Locale locale)` | Get description for first label of section. | Merchant ID, section ID, Locale | `DynamicLabelDescription` | None |
| `void deleteDynamicLabelDescriptions(Collection<DynamicLabelDescription> coll)` | Bulk delete of descriptions. | Collection | None |
| `void saveDynamicLabel(Collection<DynamicLabel> labels)` | Persist multiple labels. | Label collection | None |
| `void getModuleConfigurations(String configurationKey, int countryId)` | Find module config by key & country. | Key, country ID | `Collection<ModuleConfiguration>` | None |
| `Collection<ModuleConfiguration> getModuleConfigurations(List<String> ids)` | Find config by module ID list. | List of module IDs | `Collection<ModuleConfiguration>` | None |
| `Map<String, String> getModuleConfigurationsKeyValue(String configurationKey, int countryId)` | Return map `key → value` for all matching configs. | Key, country ID | `Map` | None |
| `ModuleConfiguration getModuleConfigurations(String configurationKey, int countryId)` | (Deprecated) alias for `findByConfigurationKeyAndCountryCode`. | Key, country ID | `Collection<ModuleConfiguration>` | None |
| `Map<String, CoreModuleService> getPaymentMethodsMap(int countryid)` | Return a map of payment method name → `CoreModuleService`. | Country ID | `Map<String, CoreModuleService>` | None |
| `ModuleConfiguration getModuleConfigurations(String configurationKey, int countryId)` – *see above* | – | – | – | – |
| `Portlet getPortlet(long portletId)` | Retrieve portlet by primary key. | ID | `Portlet` | None |
| `void deletePortlet(Portlet portlet)` | Delete a portlet and its merchant configuration. | `Portlet` | Removes merchant config via `MerchantService` | Side‑effects: DB delete, config removal |
| `Collection<Portlet> getPortlets(long pageId, int merchantId)` | Portlets on a page for a merchant. | Page ID, merchant ID | `Collection<Portlet>` | None |
| `Collection<Portlet> getPortlets(long pageId, String columnId, int merchantId)` | Portlets filtered by column. | Page ID, column, merchant | `Collection<Portlet>` | None |
| `void saveOrUpdateAllPortlets(Collection<Portlet> instances)` | Persist/merge a collection. | Portlet collection | None |
| `void saveOrUpdatePortlet(Portlet instance)` | Persist/merge a single portlet. | Portlet | None |
| `Page getPage(long pageId, int merchantId)` | Page lookup by ID & merchant. | Page ID, merchant | `Page` | None |
| `Page getPage(String title, int merchantId)` | Page lookup by title & merchant. | Title, merchant | `Page` | None |
| `void saveOrUpdatePage(Page page)` | Persist/merge a page. | `Page` | None |

> **Note** – Some helper methods (`getPaymentMethodsMap`, `getPaymentMetodsMap`) are *not* annotated `@Transactional` and currently use a static `ServiceFactory` lookup.  This deviates from the rest of the class.

---

## 4. Dependencies  

| Library | Purpose |
|---------|---------|
| **Spring Framework** (`@Service`, `@Autowired`, `@Transactional`) | Bean lifecycle, dependency injection, transaction management. |
| **Hibernate / JPA** (implied by DAO signatures) | ORM mapping and persistence. |
| **Apache Commons Lang** (`StringUtils`) | Null/blank checks. |
| **SalesManager core** (`com.salesmanager.*`) | Domain entities (Country, Currency, etc.) and DAO interfaces. |
| **Custom utilities** (`CountryUtil`, `LanguageUtil`, `ConfigurationFieldUtil`, `ServicesUtil`) | Helper logic for country codes, language codes, and configuration strings. |

The code is tightly coupled to the *SalesManager* data model and to static helper factories.  All DAOs are assumed to be correctly wired via Spring.

---

## 4.1 Things that can be improved

| Problem | Why it matters | Recommendation |
|---------|----------------|----------------|
| **Raw types & untyped collections** | Compromises compile‑time type safety, can lead to `ClassCastException` at runtime. | Use generics everywhere (`Map<String, String>`, `Collection<DynamicLabel>`, etc.). |
| **No read‑only transaction hints** | All read‑only queries still acquire a write lock, which hurts scalability. | Add `@Transactional(readOnly = true)` to pure query methods. |
| **Missing interface** | Harder to mock and unit‑test; no polymorphism. | Define a `ReferenceService` interface and inject that instead of the concrete class. |
| **Static `ServiceFactory` lookup** | Breaks Spring’s dependency‑injection mechanism, making the class hard to test and increasing coupling. | Autowire `MerchantService` (or use `@Inject`), remove `ServiceFactory` call. |
| **Duplicate deletion logic** (`deleteAllDynamicLabel`, `deleteAllDynamicLabel(Collection)`, `deleteDynamicLabel`) | Code duplication, potential for inconsistencies. | Move the orchestration to a dedicated helper/DAO, keep the service thin. |
| **Hard‑coded constant usage** (`ServicesUtil.getPaymentMetodsMap`) | Harder to change; might not be transactional. | Inject any required services and use method parameters only. |
| **Potential NPEs** (e.g. `CountryUtil.getCountryIsoCodeById` returning `null`) | Unhandled `null` leads to runtime failures. | Add defensive checks or throw meaningful exceptions. |
| **Legacy loops & Iterators** | Inefficient, verbose. | Replace with Java 8 streams or simple DAO queries. |
| **No Javadoc** | Limits maintainability. | Add clear Javadoc for public API. |
| **Large, monolithic class** | Hard to read, maintain, test. | Split into focused services: `CountryService`, `DynamicLabelService`, `PageService`, etc. |

---

## 4. Additional Notes  

### 4.1 Edge Cases & Error Handling  

* **Null Locale** – Methods accepting a `Locale` assume non‑null.  A caller passing `null` will cause a `NullPointerException` when `getLanguage()` or `getCountryIsoCodeById` is invoked.  
* **Empty Result Sets** – Some methods (e.g. `getDynamicLabelDescription`) pick the *first* element from a list without validating that the list is unique or non‑empty beyond a simple size check.  
* **Bulk Deletion Race Conditions** – `deleteAllDynamicLabel` deletes labels, then immediately deletes related portlets via `portletDao.getDynamicLabels(ids, merchantId)`.  If another thread inserts a new portlet for the same label during the same transaction, the new portlet may be lost.  
* **Hard‑coded Strings** – `ConfigurationFieldUtil.getMerchantConfigurationKey(page.getTitle(), portlet.getTitle())` builds a config key.  If the key format changes, the whole method will silently break.  

### 4.2 Performance  

* **Read‑only transactions** are not flagged; thus each read query may lock the underlying tables, impacting concurrent writes.  
* **Bulk deletes** (`deleteAllDynamicLabel`) fetch all labels and then delete all portlet references one by one.  For large merchants this can be expensive; consider a single JPQL/Hibernate bulk delete.  

### 4.3 Testability  

Because the service is concrete and heavily uses static factories, unit tests must either:

1. Load the Spring context (integration tests), or  
2. Mock the DAOs manually and set them via reflection (painful).

Introducing an interface and using constructor injection would make the class trivial to mock.

### 4.4 Security  

The service does **not** perform any role‑based checks; it trusts the caller to supply a correct `merchantId`.  In a multi‑tenant environment, this is acceptable only if the calling layer guarantees the tenant.  A missing check could allow one merchant to delete another’s labels or portlets.

---

## 5. Recommendations  

| Area | Suggested Change | Expected Benefit |
|------|------------------|-----------------|
| **Generics** | Replace all raw `Collection`, `Map`, `Iterator` usages with typed generics. | Compile‑time safety, fewer casts. |
| **Transactional hints** | Annotate read‑only methods with `@Transactional(readOnly = true)`. | Reduced lock contention, clearer intent. |
| **Interface** | Create a `ReferenceService` interface; have this class implement it. | Easier mocking, clearer contract. |
| **Dependency injection** | Autowire `MerchantService` and other static utilities; remove `ServiceFactory.getService()`. | Improves testability, follows Spring best‑practice. |
| **Code duplication** | Consolidate delete logic (e.g. `deleteAllDynamicLabel`) into a helper method or DAO. | Less maintenance overhead. |
| **Bulk operations** | Use Hibernate/JPA bulk delete where possible (`DELETE FROM DynamicLabelDescription WHERE id IN (…)`). | Faster execution for large sets. |
| **Exception handling** | Wrap DAO calls in try/catch where appropriate; translate checked exceptions into a custom unchecked `ReferenceException`. | Clearer failure modes. |
| **Documentation** | Add Javadoc to all public methods, especially those with complex orchestration. | Easier onboarding. |
| **Logging** | Add debug/trace logs for operations that modify data (e.g., deletion of portlets). | Easier troubleshooting. |
| **Unit‑testable** | Split the class into smaller services (`CountryService`, `DynamicLabelService`, `PageService`) each with a clear responsibility. | Improved maintainability and test coverage. |
| **Security** | Validate `merchantId` against the current authenticated merchant in any mutating method. | Prevent cross‑tenant data leaks. |

---

### Bottom line  

`ReferenceService` fulfils its role as a façade to reference data, but the implementation is a legacy‑style, monolithic service with numerous coupling and type‑safety issues.  Refactoring towards a clean, interface‑driven design with proper generics, read‑only transaction hints, and better unit‑testability would greatly improve the codebase’s robustness and maintainability.

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
package com.salesmanager.core.service.reference;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.salesmanager.core.constants.ShippingConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.orders.OrderStatus;
import com.salesmanager.core.entity.reference.CentralCountryStatus;
import com.salesmanager.core.entity.reference.CentralMeasureUnits;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.CountryDescription;
import com.salesmanager.core.entity.reference.Currency;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.DynamicLabelDescription;
import com.salesmanager.core.entity.reference.DynamicLabelDescriptionId;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.entity.reference.ModuleConfiguration;
import com.salesmanager.core.entity.reference.ModuleConfigurationId;
import com.salesmanager.core.entity.reference.Page;
import com.salesmanager.core.entity.reference.Portlet;
import com.salesmanager.core.entity.reference.ProductType;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.entity.system.Field;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.common.impl.ServicesUtil;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.impl.dao.ICoreModuleServiceDao;
import com.salesmanager.core.service.reference.impl.dao.ICountryDao;
import com.salesmanager.core.service.reference.impl.dao.ICountryDescriptionDao;
import com.salesmanager.core.service.reference.impl.dao.IDynamicLabelDao;
import com.salesmanager.core.service.reference.impl.dao.IDynamicLabelDescriptionDao;
import com.salesmanager.core.service.reference.impl.dao.IGeoZoneDao;
import com.salesmanager.core.service.reference.impl.dao.IGlobalReferenceDao;
import com.salesmanager.core.service.reference.impl.dao.IModuleConfigurationDao;
import com.salesmanager.core.service.reference.impl.dao.IPageDao;
import com.salesmanager.core.service.reference.impl.dao.IPortletDao;
import com.salesmanager.core.service.reference.impl.dao.IZoneDao;
import com.salesmanager.core.service.reference.impl.dao.IZoneToGeoZoneDao;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.ConfigurationFieldUtil;
import com.salesmanager.core.util.LanguageUtil;

@Service
public class ReferenceService {

	@Autowired
	private IZoneToGeoZoneDao zoneToGeoZoneDao;

	@Autowired
	private IGeoZoneDao geoZoneDao;

	@Autowired
	private IModuleConfigurationDao moduleConfigurationDao;

	@Autowired
	private ICoreModuleServiceDao coreModuleServiceDao;

	@Autowired
	private IDynamicLabelDao dynamicLabelDao;

	@Autowired
	private IDynamicLabelDescriptionDao dynamicLabelDescriptionDao;

	@Autowired
	private ICountryDescriptionDao countryDescriptionDao;

	@Autowired
	private ICountryDao countryDao;

	@Autowired
	private IGlobalReferenceDao globalReferenceDao;

	@Autowired
	private IZoneDao zoneDao;
	
	@Autowired
	private IPortletDao portletDao;
	
	@Autowired
	private IPageDao pageDao;

	/** Reference Data **/
	/**
	 * This list is used as a reference Collection representing Product Types
	 * 
	 * @return
	 */
	@Transactional
	public Collection<ProductType> getProductTypes() {
		return globalReferenceDao.getProductTypes();
	}

	/**
	 * Returns a Map containing creditCardId and CreditCard entity
	 * 
	 * @return
	 */
	@Transactional
	public Map getSupportedCreditCards() {
		return globalReferenceDao.getSupportedCreditCards();
	}

	/**
	 * Returns the complete country list
	 * 
	 * @return
	 */
	@Transactional
	public Collection<Country> getCountries() {
		return countryDao.getCountries();
	}

	/**
	 * Returns the complete list of languages
	 * 
	 * @return
	 */
	@Transactional
	public Collection<Language> getLanguages() {
		return globalReferenceDao.getLanguages();
	}

	/**
	 * Returns the complete zone list
	 * 
	 * @return
	 */
	@Transactional
	public Collection<Zone> getZones() {
		return globalReferenceDao.getZones();
	}

	/**
	 * Returns the complete measure units list
	 * 
	 * @return
	 */
	@Transactional
	public Collection<CentralMeasureUnits> getMeasureUnits() {
		return globalReferenceDao.getMeasureUnits();
	}

	/**
	 * Returns the complete measure currency list
	 * 
	 * @return
	 */
	@Transactional
	public Collection<Currency> getCurrencies() {
		return globalReferenceDao.getCurrencies();
	}

	/**
	 * Returns the complete order status list
	 * 
	 * @return
	 */
	@Transactional
	public Collection<OrderStatus> getOrderStatus() {
		return globalReferenceDao.getOrderStatus();
	}

	/**
	 * Returns a complete list of shipping country status
	 * 
	 * @return
	 */
	@Transactional
	public Collection<CentralCountryStatus> getCountryStatus() {
		return globalReferenceDao.getCountryStatus();
	}

	@Transactional
	public CountryDescription getCountryDescriptionByIsoCode(String isocode,
			int languageId) throws Exception {
		return countryDescriptionDao.findByIsoCode(isocode, languageId);
	}

	@Transactional
	public CountryDescription getCountryDescriptionByCountryId(int countryId,
			int languageId) throws Exception {
		return countryDescriptionDao.findByCountryId(countryId, languageId);
	}

	@Transactional
	public CountryDescription getCountryDescriptionByName(String name,
			int languageId) throws Exception {
		return countryDescriptionDao.findByCountryName(name, languageId);
	}

	@Transactional
	public Country getCountryByName(String name, int languageId)
			throws Exception {
		return countryDao.findByName(name, languageId);
	}

	@Transactional
	public Country getCountryByIsoCode(String isocode) throws Exception {
		return countryDao.findByIsoCode(isocode);
	}

	@Transactional
	public Zone getZoneByIsoCode(String isocode, int languageId)
			throws Exception {
		return zoneDao.findByCode(isocode, languageId);
	}

	@Transactional
	public Zone getZoneByName(String name, int languageId) throws Exception {
		return zoneDao.findByName(name, languageId);
	}

	@Transactional
	public void deleteGeoZones(int merchantId) throws Exception {
		Collection geoZonesColl = geoZoneDao.findByMerchantId(merchantId);

		if (geoZonesColl != null && geoZonesColl.size() > 0) {
			geoZoneDao.deleteAll(geoZonesColl);
		}

		Collection zoneToGeoCollection = zoneToGeoZoneDao
				.findByMerchantId(merchantId);
		if (zoneToGeoCollection != null && zoneToGeoCollection.size() > 0) {
			zoneToGeoZoneDao.deleteAll(zoneToGeoCollection);
		}
	}

	@Transactional
	public Collection<DynamicLabel> getDynamicLabels(int merchantId,
			int sectionId) throws Exception {
		return dynamicLabelDao.findByMerchantIdAndSectionId(merchantId,
				sectionId);
	}

	@Transactional
	public Collection<DynamicLabel> getDynamicLabels(int merchantId,
			int sectionId, Locale locale) throws Exception {
		return dynamicLabelDao.findByMerchantIdAndSectionIdAndLanguageId(
				merchantId, sectionId, LanguageUtil
						.getLanguageNumberCode(locale.getLanguage()));
	}
	
	@Transactional
	public Collection<DynamicLabel> getDynamicLabels(int merchantId,
			List<Integer> sections, Locale locale) throws Exception {
		return dynamicLabelDao.findByMerchantIdAnsSectionIdsAndLanguageId(
				merchantId, sections, LanguageUtil
						.getLanguageNumberCode(locale.getLanguage()));
	}

	@Transactional
	public Collection<DynamicLabel> getDynamicLabelsByIds(int merchantId, List<Long> ids, Locale locale) throws Exception {
		return dynamicLabelDao.findByMerchantIdAndLabelIdAndLanguageId(merchantId, ids, LanguageUtil
				.getLanguageNumberCode(locale.getLanguage()));
	}
	
	@Transactional
	public Collection<DynamicLabel> getDynamicLabelsByTitles(int merchantId, List<String> ids, Locale locale) throws Exception {
		return dynamicLabelDao.findByMerchantIdAndTitleAndLanguageId(merchantId, ids, LanguageUtil
				.getLanguageNumberCode(locale.getLanguage()));
	}
	
	@Transactional
	public DynamicLabel getDynamicLabel(long id) throws Exception {
		return dynamicLabelDao.findById(id);
	}

	@Transactional
	public Collection<DynamicLabel> getDynamicLabels(int merchantId)
			throws Exception {
		return dynamicLabelDao.findByMerchantId(merchantId);
	}

	@Transactional
	public Collection<DynamicLabel> getDynamicLabelsByLanguage(int merchantId,
			String language) throws Exception {

		int l = LanguageUtil.getLanguageNumberCode(language);

		return dynamicLabelDao.findByMerchantIdAndLanguageId(merchantId, l);
	}

	@Transactional
	public DynamicLabel getDynamicLabelByMerchantIdAndSeUrlAndLanguageId(
			int merchantId, String url, Locale locale) {
		return dynamicLabelDao.findByMerchantIdAndSeUrlAndLanguageId(
				merchantId, url, LanguageUtil.getLanguageNumberCode(locale
						.getLanguage()));
	}

	/**
	 * Deletes all DynamicLabel entities associated to a given merchantId
	 * 
	 * @param merchantId
	 * @throws Exception
	 */
	@Transactional
	public void deleteAllDynamicLabel(int merchantId) throws Exception {
		Collection labels = dynamicLabelDao.findByMerchantId(merchantId);
		List ids = new ArrayList();
		if (labels != null && labels.size() > 0) {
			Iterator labelsIterator = labels.iterator();
			while (labelsIterator.hasNext()) {
				DynamicLabel label = (DynamicLabel) labelsIterator.next();
				ids.add(label.getDynamicLabelId());
				Set descriptions = label.getDescriptions();
				if (descriptions != null) {
					dynamicLabelDescriptionDao.deleteAll(descriptions);
					label.setDescriptions(null);
				}
			}
			dynamicLabelDao.deleteAll(labels);
			
			if(ids.size()>0) {
				Collection portlets = portletDao.getDynamicLabels(ids, merchantId);
				portletDao.deleteAll(portlets);
			}
			
		}

	}

	@Transactional
	public void deleteAllDynamicLabel(Collection<DynamicLabel> labels)
			throws Exception {

		if (labels != null && labels.size() > 0) {
			
			int merchantId = 0;
			
			List ids = new ArrayList();
			Iterator labelsIterator = labels.iterator();
			while (labelsIterator.hasNext()) {
				DynamicLabel label = (DynamicLabel) labelsIterator.next();
				merchantId = label.getMerchantId();
				ids.add(label.getDynamicLabelId());
				Set descriptions = label.getDescriptions();
				if (descriptions != null) {
					dynamicLabelDescriptionDao.deleteAll(descriptions);
					label.setDescriptions(null);
				}
			}
			dynamicLabelDao.deleteAll(labels);
			
			if(ids.size()>0) {
				Collection portlets = portletDao.getDynamicLabels(ids, merchantId);
				portletDao.deleteAll(portlets);
			}
		}

	}

	@Transactional
	public void deleteDynamicLabel(DynamicLabel label) throws Exception {

		
		long id = label.getDynamicLabelId();
		List ids = new ArrayList();
		ids.add(id);
		Collection<Portlet> ps = portletDao.getDynamicLabels(ids, label.getMerchantId());
		if(ps!=null && ps.size()>0) {
			portletDao.deleteAll(ps);
		}
		
		Set descriptions = label.getDescriptions();
		label.setDescriptions(null);
		if (descriptions != null) {
			dynamicLabelDescriptionDao.deleteAll(descriptions);
		}
		dynamicLabelDao.delete(label);
	}
	
	@Transactional
	public DynamicLabel getDynamicLabel(
			int merchantId, String title, Locale locale) {
		return dynamicLabelDao.findByMerchantIdAndTitleAndLanguageId(merchantId, title, LanguageUtil.getLanguageNumberCode(locale.getLanguage()));
		
	}

	@Transactional
	public DynamicLabelDescription getDynamicLabelDescription(int merchantId,
			int sectionId, Locale locale) throws Exception {

		int langId = LanguageUtil.getLanguageNumberCode(locale.getLanguage());

		// get DynamicLabel
		DynamicLabel lbl = null;
		List lbls = (List) dynamicLabelDao.findByMerchantIdAndSectionId(
				merchantId, sectionId);

		if (lbls != null && lbls.size() > 0) {
			lbl = (DynamicLabel) lbls.get(0);
		}

		if (lbl != null) {
			Set descriptions = lbl.getDescriptions();
			if (descriptions != null) {
				Iterator i = descriptions.iterator();
				DynamicLabelDescription returnDesc = null;
				while (i.hasNext()) {
					DynamicLabelDescription desc = (DynamicLabelDescription) i
							.next();
					returnDesc = desc;
					DynamicLabelDescriptionId id = desc.getId();
					if (id.getLanguageId() == langId) {
						return desc;
					}

				}
				return returnDesc;
			}

		}
		return null;
	}

	@Transactional
	public void deleteDynamicLabelDescriptions(
			Collection<DynamicLabelDescription> coll) throws Exception {
		dynamicLabelDescriptionDao.deleteAll(coll);
	}

	@Transactional
	public void saveDynamicLabel(Collection<DynamicLabel> labels)
			throws Exception {
		// dynamicLabelDao.saveOrUpdateAll(labels);
		if (labels != null && labels.size() > 0) {
			Iterator i = labels.iterator();
			while (i.hasNext()) {
				DynamicLabel dl = (DynamicLabel) i.next();
				saveOrUpdateDynamicLabel(dl);
			}
		}

	}

	@Transactional
	public void saveOrUpdateDynamicLabel(DynamicLabel label) throws Exception {

		Set descriptions = label.getDescriptions();
		label.setDescriptions(null);
		dynamicLabelDao.saveOrUpdate(label);

		if (descriptions != null) {

			Iterator i = descriptions.iterator();
			while (i.hasNext()) {

				DynamicLabelDescription desc = (DynamicLabelDescription) i
						.next();
				DynamicLabelDescriptionId id = desc.getId();
				if (id == null) {
					throw new Exception("DynamicLabelDescriptionId is null");
				}
				id.setDynamicLabelId(label.getDynamicLabelId());
			}
			dynamicLabelDescriptionDao.saveOrUpdateAll(descriptions);

		}

	}

	/**
	 * Returns a Module Configuration line
	 * 
	 * @param moduleId
	 * @param configurationKey
	 * @param countryIsoCode
	 * @return
	 */
	@Transactional
	public ModuleConfiguration getModuleConfiguration(String moduleId,
			String configurationKey, String countryIsoCode) throws Exception {

		ModuleConfigurationId id = new ModuleConfigurationId();

		id.setConfigurationKey(configurationKey);
		id.setConfigurationModule(moduleId);
		id.setCountryIsoCode2(countryIsoCode);

		return moduleConfigurationDao.findById(id);

	}

	@Transactional
	public CoreModuleService getCoreModuleService(int countryId,
			String moduleName) throws Exception {
		return coreModuleServiceDao.findByModuleAndRegion(moduleName,
				CountryUtil.getCountryIsoCodeById(countryId));
	}

	@Transactional
	public CoreModuleService getCoreModuleService(String countryIsoCode,
			String moduleName) throws Exception {
		return coreModuleServiceDao.findByModuleAndRegion(moduleName,
				countryIsoCode);
	}

	@Transactional
	public Collection<CoreModuleService> getCoreModuleServices()
			throws Exception {
		return coreModuleServiceDao.getCoreModulesServices();
	}

	/**
	 * Returns a Collection of CoreModuleService for shipping and a given
	 * subType and a given country code
	 * 
	 * @param subType
	 * @param countryIsoCode
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Collection<CoreModuleService> getShippingModules(int subType,
			String countryIsoCode) throws Exception {
		return coreModuleServiceDao.findByServiceTypeAndSubTypeByRegion(
				ShippingConstants.INTEGRATION_SERVICE_SHIPPING_RT_QUOTE,
				subType, countryIsoCode);
	}

	/**
	 * Returns a list of services for a given type (core_modules_service_code)
	 * and a Country iso code
	 * 
	 * @param type
	 * @param countryIsoCode
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Collection<CoreModuleService> getCoreModules(int type,
			String countryIsoCode) throws Exception {
		return coreModuleServiceDao.findByServiceTypeAndByRegion(type,
				countryIsoCode);
	}

	public Collection<CoreModuleService> getPaymentMethodsList(int countryid) {
		String country = CountryUtil.getCountryIsoCodeById(countryid);
		return ServicesUtil.getPaymentMethodsList(country);
	}

	@Transactional
	public Collection<ModuleConfiguration> getModuleConfigurations(
			String configurationKey, int countryId) {
		return moduleConfigurationDao.findByConfigurationKeyAndCountryCode(
				configurationKey, CountryUtil.getCountryIsoCodeById(countryId));
	}
	
	@Transactional
	public Collection<ModuleConfiguration> getModuleConfigurations(
			List<String> ids) {
		return moduleConfigurationDao.findByModuleIds(ids);
	}

	@Transactional
	public Map<String, String> getModuleConfigurationsKeyValue(
			String configurationKey, int countryId) {
		Collection values = moduleConfigurationDao
				.findByConfigurationModuleAndCountryCode(configurationKey,
						CountryUtil.getCountryIsoCodeById(countryId));
		Map returnMap = new HashMap();
		if (values != null && values.size() > 0) {
			Iterator i = values.iterator();
			while (i.hasNext()) {
				ModuleConfiguration conf = (ModuleConfiguration) i.next();
				returnMap.put(conf.getId().getConfigurationKey(), conf
						.getConfigurationValue());
			}
		}
		return returnMap;
	}

	/**
	 * Returns a Map of moduleName,CoreModuleService
	 * 
	 * @param countryid
	 * @return
	 */
	// @Transactional
	public Map<String, CoreModuleService> getPaymentMethodsMap(int countryid) {

		String country = CountryUtil.getCountryIsoCodeById(countryid);

		return ServicesUtil.getPaymentMetodsMap(country);

	}
	
	/** Content Portlets and Page **/
	
	/**
	 * Retrieves a single Portlet based on its portletId
	 */
	@Transactional
	public Portlet getPortlet(long portletId) throws Exception {
		return portletDao.findById(portletId);
	}
	
	/**
	 * Delete a Portlet instance
	 * @param instance
	 * @throws Exception
	 */
	@Transactional
	public void deletePortlet(Portlet portlet) throws Exception {
		portletDao.delete(portlet);
		
		//need to remove portlet configuration
		//get page
		Page page = this.getPage(portlet.getPage(), portlet.getMerchantId());
		
		//get merchant configuration
		MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
		ConfigurationRequest configRequest = new ConfigurationRequest(portlet.getMerchantId(),ConfigurationFieldUtil.getMerchantConfigurationKey(page.getTitle(),portlet.getTitle()));
		ConfigurationResponse configResponse = mservice.getConfiguration(configRequest);
		
		
		
		if(page!=null) {
		
			MerchantConfiguration conf = configResponse.getMerchantConfiguration(ConfigurationFieldUtil.getMerchantConfigurationKey(page.getTitle(),portlet.getTitle()));
			if(conf!=null) {
/*				Map updatebleMap = new HashMap();
				String f = conf.getConfigurationValue();
				if(!StringUtils.isBlank(f)) {
					Map fieldValues = FieldUtil.parseFieldsValues(f);
					for(Object o : fieldValues.keySet()) {
						String module = (String)o;
						if(!module.equals(portlet.getTitle())) {
							List fields = (List)fieldValues.get(module);
							updatebleMap.put(module, fields);
						}
					}
				}
				//update entries
				String line = FieldUtil.buildFieldValuesString(updatebleMap);
				conf.setConfigurationValue(line);*/
				mservice.deleteMerchantConfiguration(conf);
			}
		
		}
		
		
	}
	
	/**
	 * Retrieves a Collection of Portlet for a given page (pageId) and a given merchantId
	 * @param pageId
	 * @param merchantId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Collection<Portlet> getPortlets(long pageId, int merchantId) throws Exception {
		return portletDao.getPortlets(pageId, merchantId);
	}
	
	
	/**
	 * Retrieves a Collection of Portlet for a given page (pageId) 
	 * a given position (columnId) and a merchantId
	 * @param pageId
	 * @param columnId
	 * @param merchantId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Collection<Portlet> getPortlets(long pageId, String columnId, int merchantId) throws Exception {
		return portletDao.getPortlets(pageId, columnId, merchantId);
	}
	
	
	/**
	 * Save or update a Collection of Portlet instances
	 * @param instances
	 * @throws Exception
	 */
	@Transactional
	public void saveOrUpdateAllPortlets(Collection<Portlet> instances) throws Exception {
		portletDao.saveOrUpdateAll(instances);
	}
	
	/**
	 * Save or update a single Portlet instance
	 * @param instance
	 * @throws Exception
	 */
	@Transactional
	public void saveOrUpdatePortlet(Portlet instance) throws Exception {
		portletDao.saveOrUpdate(instance);
	}
	
	/**
	 * Returns a Page instance based on the pageId and merchantId
	 * @param pageId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Page getPage(long pageId, int merchantId) throws Exception {
		return pageDao.getPage(pageId, merchantId);
	}
	
	/**
	 * Returns a Page instance based on the pageId and merchantId
	 * @param pageId
	 * @return
	 * @throws Exception
	 */
	@Transactional
	public Page getPage(String title, int merchantId) throws Exception {
		return pageDao.getPage(title, merchantId);
	}
	
	/**
	 * Saves a new Page instance or update an existing Page instance
	 * @param page
	 * @throws Exception
	 */
	@Transactional
	public void saveOrUpdatePage(Page page) throws Exception {
		pageDao.saveOrUpdate(page);
	}

}



```
