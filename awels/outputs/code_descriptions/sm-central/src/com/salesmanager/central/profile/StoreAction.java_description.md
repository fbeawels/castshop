# StoreAction.java

## Review

## 1. Summary  

**Purpose**  
`StoreAction` is a Struts 2 action that handles CRUD‑style interactions for a merchant’s store profile. It loads the current profile (`fetchProfile`), displays it (`display`) and persists changes (`saveStore`). The action is part of the *Sales Manager* central profile module and relies on the underlying services (`MerchantService`, `RefCache`) to retrieve and persist store data.

**Key Components**  

| Class/Interface | Role |
|------------------|------|
| `StoreAction` | Web layer controller – orchestrates profile loading, editing, and persistence. |
| `CountrySelectBaseAction` | Base action providing helpers such as `prepareSelections`, `setPageTitle`, and context handling. |
| `MerchantStore` | Domain model representing the store’s configuration. |
| `MerchantService` | Service layer for CRUD operations on `MerchantStore` and user info. |
| `RefCache` | Cache of reference data (countries, zones). |
| `LanguageHelper`, `LanguageUtil` | Helpers for language/locale handling. |
| `MessageUtil`, `LabelUtil` | I18n and message handling. |

**Design Patterns & Libraries**  

* **Singleton Service Factory** – `ServiceFactory.getService(...)` provides a service instance.  
* **Cache Pattern** – `RefCache` caches reference data for quick lookup.  
* **Struts 2** – action lifecycle, `ActionContext`, request/response handling.  
* **Apache Commons Lang** – `StringUtils` utilities.  
* **Log4j** – logging.  
* **Java .util.Date** – legacy date handling (should consider Java 8 `java.time`).  

---

## 2. Detailed Description  

### Flow of Execution  

1. **Request** – A user navigates to the *Store* profile page.  
2. **`fetchProfile`**  
   * Reads the current merchant context from the session.  
   * Retrieves the merchant’s `MerchantStore` from the database.  
   * If missing, creates a default profile with the request host as domain.  
   * Populates language and country defaults from the logged‑in user.  
   * Formats the “in business since” date and prepares country/zone selections.  
   * Stores the populated profile in the `merchantProfile` field and returns `SUCCESS`.  
3. **`display`** – Simply ensures `merchantProfile` is populated (calls `fetchProfile` if null).  
4. **`saveStore`** – Triggered when the user submits changes.  
   * Loads the existing `MerchantStore` or creates a new one.  
   * Gathers supported languages from the UI into a semicolon‑separated string.  
   * Validates mandatory fields (e.g., supported languages).  
   * Copies all profile fields from `merchantProfile` to the domain object.  
   * Handles zone/region resolution via `RefCache`.  
   * Persists the store with `mservice.saveOrUpdateMerchantStore`.  
   * Refreshes the session context (country, language, currency).  
   * Sets the locale for the session and action context.  
   * Adds a success message and returns `SUCCESS`.  

### Assumptions & Constraints  

| Area | Assumption | Impact |
|------|------------|--------|
| Session | `Context` attribute (`ProfileConstants.context`) is always present. | Failure to find it results in NPE. |
| User | The logged‑in user exists and has `MerchantUserInformation`. | Missing user leads to NPE. |
| Language | Supported languages are stored as a semicolon‑delimited string. | Hard‑coded format; no type safety. |
| Date | Uses legacy `java.util.Date`. | Hard to test/time‑zone‑aware; better use `LocalDate`. |
| Services | `ServiceFactory` provides a singleton. | Tight coupling, difficult to mock in tests. |

### Architecture & Design Choices  

* **Action‑First** – Business logic is largely in the action rather than a dedicated service; this simplifies the controller but couples persistence logic to the UI layer.  
* **Manual Validation** – Validation of required fields is performed in `saveStore`; a dedicated validation framework (Struts 2 XML or annotations) would be cleaner.  
* **String‑Based Config** – Supported languages, currencies, etc., are persisted as raw strings; this keeps the DB schema simple but sacrifices type safety.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `fetchProfile()` | Loads or creates the merchant’s `MerchantStore` and populates the UI context. | None (reads from session & request). | Returns `SUCCESS` or `ERROR`. | Sets `merchantProfile`, prepares selections, logs errors. |
| `display()` | Entry point for the profile view. | None. | `SUCCESS`. | Calls `fetchProfile` if needed. |
| `saveStore()` | Persists changes to the merchant store. | None (uses `merchantProfile` and UI fields). | `SUCCESS` (or `INPUT` if language missing). | Updates DB, refreshes session context, sets locale, logs errors. |
| `getCountryCode()`, `setCountryCode(Integer)` | Getter/Setter for `countryCode` (unused). | - | Integer | None |
| `getSupportedLanguages()`, `setSupportedLanguages(List)` | Access to language list for UI. | - | List | None |
| `getInBusinessSince()`, `setInBusinessSince(String)` | Access to formatted business‑since date. | - | String | None |

### Reusable / Utility Methods  

* `LanguageHelper.setLanguages(...)` – Configures language context.  
* `DateUtil.formatDate(Date)` / `DateUtil.getDate(String)` – String/Date conversions.  
* `MessageUtil.addMessage(...)`, `MessageUtil.addErrorMessage(...)` – Centralized i18n messaging.  
* `RefCache.getInstance().getAllZonesmap(...)` – Cached zone lookup.  

---

## 4. Dependencies  

| Library / Framework | Version (inferred) | Role | Standard / Third‑Party |
|---------------------|--------------------|------|------------------------|
| `com.opensymphony.xwork2` | Struts 2 | Action framework, `ActionContext` | Third‑Party |
| `org.apache.commons.lang.StringUtils` | Commons Lang | String utilities | Third‑Party |
| `org.apache.log4j.Logger` | Log4j | Logging | Third‑Party |
| `com.salesmanager.core` packages | Custom | Domain, services, utilities | Internal |
| `java.util` (Date, List, Map, etc.) | JDK | Core collections & date | Standard |

No external API calls beyond the internal services.

---

## 5. Additional Notes  

### Potential Issues & Edge Cases  

1. **Null Safety**  
   * `ctx` or `userInfo` could be `null` if the session has expired or the user is not logged in.  
   * No checks for `profile.getSupportedlanguages()` being `null` before `equals("")` may throw `NullPointerException`.  

2. **Thread Safety**  
   * Struts 2 actions are instantiated per request, so the class is effectively thread‑safe.  
   * However, shared objects like `RefCache` are used; ensure that the cache implementation is thread‑safe.  

3. **Validation Flow**  
   * When no supported languages are selected, the method adds an error message but still returns `SUCCESS`.  
     This may result in the UI not showing the validation error correctly.  
   * Ideally, return `INPUT` or use Struts 2 validation annotations.  

4. **Legacy Date API**  
   * Mixing `java.util.Date` with `DateUtil` (likely custom) can lead to timezone bugs.  
   * Consider migrating to `java.time.LocalDate` / `ZonedDateTime`.  

5. **Hard‑coded Strings & Magic Numbers**  
   * `"message.confirmation.languagerequired"` etc. are hard‑coded; moving to constants improves maintainability.  
   * The default `bgcolorcode` of `1` and default `templateModule` string are magic values.  

6. **Service Instantiation**  
   * `ServiceFactory.getService(ServiceFactory.MerchantService)` returns a new instance each time.  
     Dependency injection (e.g., Spring) would simplify testing and configuration.  

7. **Locale Handling**  
   * Locale is constructed using `new Locale("en", c.getCountryIsoCode2())`, hard‑coding `"en"` for language.  
     The application should respect the user’s actual language preference.  

8. **Redundant Code**  
   * `merchantProfile.getTemplateModule()` is validated but the validation is commented out.  
   * The check for `store == null` in `saveStore` could be simplified by using `ObjectUtils.defaultIfNull`.  

### Suggested Enhancements  

| Area | Recommendation |
|------|----------------|
| **Null Checks** | Wrap all session/context retrievals in optional checks; propagate meaningful errors. |
| **Validation** | Move field validation to a separate `StoreValidator` or use Struts 2 XML/annotation validation. |
| **Generics** | Replace raw `List`, `Map` types with parameterized ones (`List<String>`, `Map<Integer, Country>`) to avoid unchecked warnings. |
| **DTO / Service Layer** | Extract persistence logic into a dedicated service method (`updateMerchantStore(MerchantStore store)`), decoupling the action from DB operations. |
| **Locale** | Store and use the actual user language code; avoid hard‑coding `"en"` when setting the locale. |
| **Date Handling** | Migrate to `java.time` APIs; use `LocalDate` for “in business since”. |
| **Constants** | Define all hard‑coded strings and default values in a `StoreConstants` class. |
| **Unit Tests** | With dependency injection, write unit tests for `StoreAction` that mock `MerchantService` and `RefCache`. |
| **Error Handling** | Return `ERROR` or `INPUT` appropriately instead of `SUCCESS` after a validation failure. |
| **Logging** | Include request identifiers or merchant ID in log statements for easier tracing. |

### Summary  

`StoreAction` serves its basic purpose of loading, displaying, and persisting a merchant’s store profile. However, it contains several areas that could benefit from modernization (generics, type safety, dependency injection) and stricter validation. Addressing these points would improve maintainability, testability, and user experience.

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
package com.salesmanager.central.profile;

import java.util.ArrayList;
import java.util.Date;
import java.util.Iterator;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.opensymphony.xwork2.ActionContext;
import com.salesmanager.central.CountrySelectBaseAction;
import com.salesmanager.central.util.LanguageHelper;
import com.salesmanager.central.web.Constants;
import com.salesmanager.core.constants.CatalogConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.merchant.MerchantUserInformation;
import com.salesmanager.core.entity.reference.Country;
import com.salesmanager.core.entity.reference.Zone;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.cache.RefCache;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.util.DateUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.LanguageUtil;
import com.salesmanager.core.util.MessageUtil;

public class StoreAction extends CountrySelectBaseAction {

	private static final long serialVersionUID = 7448329639550683806L;

	private Logger log = Logger.getLogger(StoreAction.class);


	private String inBusinessSince;

	private MerchantStore merchantProfile;

	public MerchantStore getMerchantProfile() {
		return merchantProfile;
	}

	public void setMerchantProfile(MerchantStore merchantProfile) {
		this.merchantProfile = merchantProfile;
	}

	private Integer countryCode;
	private List supportedLanguages = new ArrayList();

	/**
	 * invoked when the page loads / refresh
	 * 
	 * @throws Exception
	 */
	public String fetchProfile() throws Exception {

		super.setPageTitle("label.menu.group.store");
		MerchantStore profile = null;

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			profile = mservice.getMerchantStore(merchantid.intValue());
			
			String user = super.getPrincipal().getRemoteUser();
			MerchantUserInformation userInfo = mservice.getMerchantUserInformation(user);

			//MerchantUserInformation userInfo = mservice
			//		.getMerchantUserInfo(merchantid.intValue());

			if (profile == null) {// should be created from the original
									// subscribtion process
				profile = new MerchantStore();
				String serverName = super.getServletRequest().getServerName();
				int serverPort = super.getServletRequest().getServerPort();
				
				if(serverPort>0) {
					serverName = serverName + ":" + String.valueOf(serverPort);
				}
				
				profile.setDomainName(serverName);
				
				
				profile.setTemplateModule(CatalogConstants.DEFAULT_TEMPLATE);

			}

			if (profile.getSupportedlanguages() != null
					&& !profile.getSupportedlanguages().equals("")) {

				LanguageHelper.setLanguages(profile.getSupportedlanguages(),
						ctx);
				Map m = ctx.getSupportedlang();
				if (m != null && m.size() > 0) {
					Set s = m.keySet();
					Iterator i = s.iterator();
					while (i.hasNext()) {
						String key = (String) i.next();
						supportedLanguages.add(key);
					}
				}
			}

			// set at least the user country code
			if (profile.getCountry() == 0) {
				profile.setCountry(userInfo.getUsercountrycode());
			}
			// set a default background
			if (profile.getBgcolorcode() == 0) {
				profile.setBgcolorcode(1);
			}

			if (profile.getStoreaddress() == null) {
				profile.setStoreaddress(userInfo.getUseraddress());
			}

			if (profile.getStorecity() == null) {
				profile.setStorecity(userInfo.getUsercity());
			}

			if (profile.getStorepostalcode() == null) {
				profile.setStorepostalcode(userInfo.getUserpostalcode());
			}

			if (profile.getBgcolorcode() == 0) {
				profile.setBgcolorcode(new Integer(1));// set to white
			}

			profile.setTemplateModule(profile.getTemplateModule());

			Date businessDate = profile.getInBusinessSince();
			if (businessDate == null) {
				businessDate = new Date();
			}
			this.setInBusinessSince(DateUtil.formatDate(businessDate));

			super.prepareSelections(profile.getCountry());

			this.merchantProfile = profile;
			return SUCCESS;

		} catch (Exception e) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error(e);
			return ERROR;
		}
	}

	/**
	 * For display in the page
	 * 
	 * @throws Exception
	 */
	public String display() throws Exception {
		super.setPageTitle("label.menu.group.store");
		try {

			if (merchantProfile == null) {
				this.fetchProfile();
			}

		} catch (Exception e) {
			log.error(e);
		}

		return SUCCESS;
	}

	/**
	 * Invoked from CRUD actions
	 * 
	 * @return
	 */
	public String saveStore() {
		super.setPageTitle("label.menu.group.store");

		MerchantStore store = null;
		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);
			store = mservice.getMerchantStore(merchantid.intValue());
			
			//validation
/*			if (StringUtils.isBlank(merchantProfile.getTemplateModule())) {
				super.setErrorMessage("errors.store.emptytemplate");
				return INPUT;
			} */

			if (store == null) {
				store = new MerchantStore();
				store.setTemplateModule(CatalogConstants.DEFAULT_TEMPLATE);
			}else {
				store.setTemplateModule(merchantProfile.getTemplateModule());
			}
			


			java.util.Date dt = new java.util.Date();

			StringBuffer languages = new StringBuffer();
			List langs = this.getSupportedLanguages();
			if (langs != null && langs.size() > 0) {
				int sz = 0;
				Iterator i = langs.iterator();

				while (i.hasNext()) {
					String lang = (String) i.next();
					languages.append(lang);

					if (sz < langs.size() - 1) {
						languages.append(";");
					}
					sz++;

				}
				store.setSupportedlanguages(languages.toString());
			} else {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(
								"message.confirmation.languagerequired"));
				store.setSupportedlanguages(Constants.ENGLISH_CODE);
				return SUCCESS;
			}

			store.setStorename(merchantProfile.getStorename());
			store.setStoreemailaddress(merchantProfile.getStoreemailaddress());
			store.setStoreaddress(merchantProfile.getStoreaddress());
			store.setStorecity(merchantProfile.getStorecity());
			store.setStorepostalcode(merchantProfile.getStorepostalcode());
			store.setCountry(merchantProfile.getCountry());
			store.setZone(merchantProfile.getZone());
			store.setCurrency(merchantProfile.getCurrency());
			

			if (!StringUtils.isBlank(merchantProfile.getWeightunitcode())) {
				store.setWeightunitcode(merchantProfile.getWeightunitcode()
						.trim());
			}
			if (!StringUtils.isBlank(merchantProfile.getSeizeunitcode())) {
				store.setSeizeunitcode(merchantProfile.getSeizeunitcode()
						.trim());
			}
			store.setStorelogo(merchantProfile.getStorelogo());
			store.setStorephone(merchantProfile.getStorephone());
			store.setBgcolorcode(merchantProfile.getBgcolorcode());
			store.setContinueshoppingurl(merchantProfile
					.getContinueshoppingurl());
			store.setUseCache(merchantProfile.isUseCache());
			store.setDomainName(merchantProfile.getDomainName());

			store.setMerchantId(merchantid.intValue());
			store.setLastModified(new java.util.Date(dt.getTime()));

			if (!StringUtils.isNumeric(merchantProfile.getZone())) {
				store.setStorestateprovince(merchantProfile
						.getStorestateprovince());
				ctx.setZoneid(0);
			} else {// get the value from zone
				ctx.setZoneid(Integer.parseInt(merchantProfile.getZone()));
				Map zones = RefCache.getInstance().getAllZonesmap(
						LanguageUtil.getLanguageNumberCode(ctx.getLang()));
				Zone z = (Zone) zones.get(Integer.parseInt(merchantProfile
						.getZone()));
				if (z != null) {
					store.setStorestateprovince(z.getZoneName());// @todo,
																	// localization
				} else {
					store.setStorestateprovince("N/A");
				}
			}

			if (!StringUtils.isBlank(this.getInBusinessSince())) {
				Date businessDate = DateUtil.getDate(this.getInBusinessSince());
				store.setInBusinessSince(businessDate);
			}

			super.prepareSelections(store.getCountry());
			mservice.saveOrUpdateMerchantStore(store);

			super.getContext().setExistingStore(true);

			// refresh context

			ctx.setCountryid(merchantProfile.getCountry());
			ctx.setSizeunit(merchantProfile.getSeizeunitcode());
			ctx.setWeightunit(merchantProfile.getWeightunitcode());
			LanguageHelper.setLanguages(languages.toString(), ctx);
			ctx.setCurrency(merchantProfile.getCurrency());

			// refresh the locale
			Map countries = RefCache.getAllcountriesmap(LanguageUtil
					.getLanguageNumberCode(ctx.getLang()));
			Country c = (Country) countries.get(merchantProfile.getCountry());
			Locale locale = new Locale("en", c.getCountryIsoCode2());
			ActionContext.getContext().setLocale(locale);
			Map sessions = ActionContext.getContext().getSession();
			sessions.put("WW_TRANS_I18N_LOCALE", locale);


			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {
			log.error(e);
			super.setTechnicalMessage();
		}

		return SUCCESS;

	}

	public Integer getCountryCode() {
		return countryCode;
	}

	public void setCountryCode(Integer countryCode) {
		this.countryCode = countryCode;
	}

	public List getSupportedLanguages() {
		return supportedLanguages;
	}

	public void setSupportedLanguages(List supportedLanguages) {
		this.supportedLanguages = supportedLanguages;
	}


	public String getInBusinessSince() {
		return inBusinessSince;
	}

	public void setInBusinessSince(String inBusinessSince) {
		this.inBusinessSince = inBusinessSince;
	}

}



```
