# EditTaxAction.java

## Review

## 1. Summary  
**Purpose** – `EditTaxAction` is a Struts‑style action that allows an administrator to view and modify tax configuration data for a merchant. It supports:  
- Changing the *tax basis* (shipping, billing, store).  
- Adding, updating or deleting **tax classes**.  
- Adding, editing or deleting **tax rates** for specific countries/zones.  

**Key components**  
| Component | Role |
|-----------|------|
| `TaxService` | CRUD on tax classes, rates and descriptions. |
| `MerchantService` | Persist/lookup merchant‑specific configuration (tax basis). |
| `ServiceFactory` | Service lookup (static factory). |
| `TaxAction` (superclass) | Provides common fields (`getServletRequest`, `getLocale`, `gatherParameters`, `setupTax`, etc.) and utility methods (`getLanguages`, `getTaxclass`, etc.). |

**Design patterns / frameworks**  
- *Service Locator* via `ServiceFactory`.  
- *Struts‑type action* (extends a base action, uses request/response objects).  
- *DTO / entity* objects (`TaxClass`, `TaxRate`, `TaxRateDescription`, `TaxRateDescriptionId`).  

---

## 2. Detailed Description  

### Execution Flow

| Phase | What Happens | Notes |
|-------|--------------|-------|
| **Initialization** | `setup()` (empty) – placeholder for future init logic. | |
| **editTaxBasis()** | *Read* the selected basis from request parameters. <br> *Retrieve* current configuration via `MerchantService`. <br> *Update* configuration value and persist. <br> *Re‑populate* tax UI with `setupTax()` and add a success message. | Handles only `TaxConstants.MODULE_TAX_BASIS`. |
| **editTaxClass()** | *Validate* the incoming tax class title.<br> *Decide* action (`create`, `update`, `delete`) via `taxclassaction`. <br> *Create* new `TaxClass` or fetch existing one, set fields, and persist.<br> *Delete* if requested. <br> *Reload* tax UI and success message. | Uses `taxclassaction` values `-1`/`0`/others to mean create, update, delete. |
| **addTaxRate()** | *Validate* rate value, parse as `BigDecimal` via `CurrencyUtil`. <br> *Collect* descriptions for all supported languages. <br> *Create* `TaxRate` (default class id, piggyback flag). <br> *Persist* via `TaxService`. <br> *Reload* UI and show success. | Hard‑codes `TaxConstants.DEFAULT_TAX_CLASS_ID`; no validation of country/zone selection. |
| **editTaxRate()** | *Validate* rate and priority (if editing). <br> *Collect* descriptions like in `addTaxRate()`. <br> *Fetch* existing `TaxRate` by id. <br> *Delete* if `taxlineaction == 1`, otherwise update fields and persist. <br> *Reload* UI and show success. | Uses `taxlineaction` to decide delete vs. update; `0` means edit. |

### Assumptions & Dependencies  

* The action relies on the `Context` object stored in the HTTP session under `ProfileConstants.context`.  
* It assumes the presence of languages (`getLanguages()`) – otherwise it aborts early.  
* Service lookup uses a *static* `ServiceFactory`. No dependency injection is employed.  
* The UI is expected to set the action properties (`taxclassaction`, `taxclassid`, etc.) before invoking the action methods.  
* Logging uses Log4j 1.x.

### Architecture Choices  

* **Action‑centric**: All business logic lives inside the action, tightly coupled to HTTP request/response.  
* **Service Locator**: Provides global access to services but makes unit testing harder.  
* **Raw collections**: `List`, `HashSet`, `Iterator` are used without generics, leading to unchecked casts.  

---

## 3. Functions/Methods  

| Method | Purpose | Inputs | Outputs | Side Effects |
|--------|---------|--------|---------|--------------|
| `setup()` | Placeholder for initialization. | None | None | None |
| `editTaxBasis()` | Update merchant’s tax basis. | Request parameters (TAX_BASIS). | `SUCCESS` Struts result. | Persists config, sets request attributes, triggers `setupTax()` |
| `editTaxClass()` | Add / update / delete a tax class. | `taxclass`, `taxclassaction`, `taxclassid`. | `SUCCESS`. | Persists/deletes `TaxClass`, reloads tax UI |
| `addTaxRate()` | Create a new tax rate. | `taxlinerate`, `choosecountry`, `choosezone`, descriptions per language, `piggyback`. | `SUCCESS`. | Persists `TaxRate`, reloads tax UI |
| `editTaxRate()` | Update or delete an existing tax rate. | `taxlineid`, `taxlineaction`, `taxlinerate`, `taxlineorder`, `taxlineclassid`, `choosecountry`, `choosezone`, descriptions, `piggyback`. | `SUCCESS`. | Persists or deletes `TaxRate`, reloads tax UI |
| Getter/Setter pairs | Store action parameters and state. | - | - | Set internal fields |

**Utility patterns** – Most methods share a common flow: gather parameters → validate → service interaction → reload UI → add message. This duplication could be refactored into helper methods.

---

## 4. Dependencies  

| Library / Framework | Purpose | Is it Standard? |
|---------------------|---------|-----------------|
| `java.math.BigDecimal` | Currency arithmetic | Standard |
| `java.util` collections | Core data structures | Standard |
| `org.apache.commons.lang.StringUtils` | String utilities | Third‑party |
| `org.apache.log4j.Logger` | Logging | Third‑party (Log4j 1.x) |
| `com.salesmanager.*` | Domain model, services, utilities | Third‑party (in‑house) |
| `javax.servlet` (via `getServletRequest`) | Web layer | Standard in servlet containers |

The code does not declare any external dependencies beyond the ones mentioned. However, it uses Log4j 1.x which is legacy; modern applications prefer SLF4J or Log4j 2.x.

---

## 5. Additional Notes & Recommendations  

### 5.1 Code Quality Issues  

| Issue | Impact | Suggested Fix |
|-------|--------|---------------|
| **Raw types** (`List`, `HashSet`, `Iterator`) | Compile‑time warnings, possible `ClassCastException`. | Use generics (`List<TaxClass>`, `HashSet<TaxRateDescription>`, `Iterator<TaxClass>`) |
| **Magic numbers** (`taxclassaction == -1`, `0`, `1`) | Hard to understand intent. | Replace with an enum (`TaxClassAction { CREATE, UPDATE, DELETE }`). |
| **Duplicate validation logic** (rate parsing, description checks) | Code duplication, harder to maintain. | Extract into private helper methods (`validateRate`, `collectDescriptions`). |
| **Static ServiceFactory** | Tight coupling, no easy unit testing. | Use dependency injection (Spring, CDI). |
| **String constants** (`TAX_BASIS`) | Spelling errors, hard to refactor. | Keep constants in a dedicated enum or `TaxConstants`. |
| **No transaction management** | Inconsistent persistence state if one operation fails. | Wrap service calls in a transactional boundary. |
| **Logging level** (`log.error(e)`) | Swallows stack trace in messages. | Use `log.error("message", e)` for full stack trace. |
| **Error handling** | `MessageUtil` usage is mixed with logging; potential duplication. | Centralize error handling via an interceptor or a helper method. |
| **Hard‑coded default tax class ID** | Makes the code less flexible. | Pass the class ID as a parameter or look it up dynamically. |
| **Use of `super` for request/locale** | Could hide overridden behaviour. | Prefer dependency injection or use of `HttpServletRequest` directly. |

### 5.2 Functional Edge Cases  

* **Missing `choosecountry`/`choosezone`** – The methods `addTaxRate` and `editTaxRate` accept these as primitives but never validate that they are positive. Passing `0` could create a tax rate that applies to “all” zones, which may or may not be intended.  
* **Null `TaxClass` during update** – If `tservice.getTaxClass(this.getTaxclassid())` returns `null`, the code logs an error and shows a generic technical message but still calls `tservice.saveOrUpdateTaxClass(tc)` on a `null` reference? Actually it checks before, so safe.  
* **Concurrent modifications** – Multiple administrators editing the same tax class/rate could race. No optimistic locking is visible.  
* **Invalid locale handling** – The code assumes all languages have descriptions. If a language description is missing, it aborts; no fallback to default language.  

### 5.3 Future Enhancements  

1. **Refactor to MVC** – Move business logic into a dedicated service layer; keep the action as a thin controller.  
2. **Use of DTOs** – Introduce request/response DTOs for tax class/rate operations to decouple from entity objects.  
3. **Internationalization** – Centralize message keys and make the action locale‑agnostic.  
4. **Validation Framework** – Use a bean‑validation library (e.g., Hibernate Validator) to annotate fields and centralize constraints.  
5. **Unit Tests** – With dependency injection, write unit tests for each action method.  
6. **Modern Logging** – Migrate to SLF4J + Log4j2 or Logback.  
7. **Security** – Ensure only authorized merchants can invoke these actions (role checks, CSRF protection).  

### 5.4 Summary  

`EditTaxAction` is a functional, albeit dated, implementation for managing tax configuration in a merchant‑centric e‑commerce system. It directly couples request handling, validation, and persistence, which is common in older Struts applications. The code would benefit greatly from modern Java practices: generics, dependency injection, clean separation of concerns, and more robust error handling. Addressing the above issues will improve maintainability, testability, and security of the tax administration module.

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
package com.salesmanager.central.tax;

import java.math.BigDecimal;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.TaxConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.tax.TaxClass;
import com.salesmanager.core.entity.tax.TaxRate;
import com.salesmanager.core.entity.tax.TaxRateDescription;
import com.salesmanager.core.entity.tax.TaxRateDescriptionId;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.tax.TaxService;
import com.salesmanager.core.util.CurrencyUtil;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;

public class EditTaxAction extends TaxAction {

	private int taxlineaction = 0;
	private long taxlineid = 0;// geozoneid
	private String taxlineorder = "0";
	private long taxlineclassid = -1;
	private String taxlinerate = null;

	private int taxclassaction = -1;
	private long taxclassid = 1;

	private int choosecountry;
	private int choosezone;

	private boolean piggyback = false;

	private Logger log = Logger.getLogger(EditTaxAction.class);

	public void setup() throws Exception {

	}

	/**
	 * Edit tax basis option
	 * 
	 * @return
	 * @throws Exception
	 */
	public String editTaxBasis() throws Exception {

		super.setPageTitle("label.tax.taxbasis.setup");
		
		Map p = this.gatherParameters();

		if (super.getLanguages() == null || super.getLanguages().size() == 0) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.profile.storenotcreated"));
			return SUCCESS;
		}

		String taxbasis = (String) p.get(super.TAX_BASIS);

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			MerchantService mservice = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			ConfigurationRequest request = new ConfigurationRequest(merchantid,
					TaxConstants.MODULE_TAX_BASIS);
			ConfigurationResponse response = mservice.getConfiguration(request);

			MerchantConfiguration config = response
					.getMerchantConfiguration(TaxConstants.MODULE_TAX_BASIS);

			if (config != null && taxbasis != null) {// tax configured
				// if the user is 'shipping' or 'billing' and is asked for
				// store, clean the tax

				config.setConfigurationValue(taxbasis);

				mservice.saveOrUpdateMerchantConfiguration(config);

			}

			super.getServletRequest().setAttribute("taxbasis", taxbasis);

			this.setupTax();

			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
		}

		return SUCCESS;

	}

	/**
	 * Add, Modify and Remove a TaxClass
	 * 
	 * @return
	 * @throws Exception
	 */
	public String editTaxClass() throws Exception {
		
		super.setPageTitle("label.tax.taxclass");

		if (super.getLanguages() == null || super.getLanguages().size() == 0) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.profile.storenotcreated"));
			return SUCCESS;
		}

		try {

			if (super.getTaxclass() == null || super.getTaxclass().equals("")) {

				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"message.error.taxclass.title"));
				setupTax();
				return SUCCESS;
			}

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			TaxService tservice = (TaxService) ServiceFactory
					.getService(ServiceFactory.TaxService);
			List txscl = tservice.getTaxClasses(merchantid);

			java.util.Date dt = new java.util.Date();

			if (taxclassaction == -1 || taxclassaction == 0) {// create
				if (txscl != null) {
					Iterator it = txscl.iterator();
					while (it.hasNext()) {
						TaxClass x = (TaxClass) it.next();
						if (x.getTaxClassTitle().equals(super.getTaxclass())) {
							MessageUtil
									.addErrorMessage(
											super.getServletRequest(),
											LabelUtil
													.getInstance()
													.getText(
															"message.error.taxclass.alreadyexist"));
							this.setupTax();
							return SUCCESS;
						}
					}
				}
			}

			if (taxclassaction == -1) {// create

				if (StringUtils.isBlank(super.getTaxclass())) {
					super.addFieldError("taxclasstitle",
							getText("errors.profile.storenotcreated"));
					return SUCCESS;
				}

				TaxClass tc = new TaxClass();
				tc.setDateAdded(new java.util.Date(dt.getTime()));
				tc.setMerchantId(merchantid);
				tc.setTaxClassDescription(super.getTaxclass());
				tc.setTaxClassTitle(super.getTaxclass());

				tservice.saveOrUpdateTaxClass(tc);

			} else if (taxclassaction == 0) {// update

				if (StringUtils.isBlank(super.getTaxclass())) {
					super.addFieldError("taxclasstitle",
							getText("errors.profile.storenotcreated"));
					return SUCCESS;
				}

				TaxClass tc = tservice.getTaxClass(this.getTaxclassid());

				if (tc == null) {
					log.error("TaxClass does not exist for id "
							+ this.getTaxclassid());
					MessageUtil
							.addErrorMessage(super.getServletRequest(),
									LabelUtil.getInstance().getText(super.getLocale(),
											"errors.technical"));

				}

				tc.setTaxClassTitle(super.getTaxclass());
				tservice.saveOrUpdateTaxClass(tc);

			} else {// delete

				TaxClass tc = tservice.getTaxClass(this.getTaxclassid());
				tservice.deleteTaxClass(tc);

			}

			this.setupTax();
			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));

		} catch (Exception e) {

			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));

		}

		return SUCCESS;

	}

	public String addTaxRate() throws Exception {

		if (super.getLanguages() == null || super.getLanguages().size() == 0) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.profile.storenotcreated"));
			return SUCCESS;
		}

		BigDecimal amount = null;

		try {

			super.gatherParameters();

			// int schemeid = schemeid;

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			TaxService tservice = (TaxService) ServiceFactory
					.getService(ServiceFactory.TaxService);
			HashSet descriptionsset = new HashSet();

			Integer priority = 0;

			// validate amount
			try {

				amount = CurrencyUtil.validateCurrency(this.getTaxlinerate(),
						ctx.getCurrency());

			} catch (Exception e) {

				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"message.error.rate.format"));
				this.setupTax();
				return SUCCESS;
			}

			// descriptions

			Iterator i = super.reflanguages.keySet().iterator();
			while (i.hasNext()) {
				int langcount = (Integer) i.next();
				String description = (String) this.getDescriptions().get(
						langcount);

				if (StringUtils.isBlank(description)) {
					MessageUtil.addErrorMessage(super.getServletRequest(),
							LabelUtil.getInstance().getText(super.getLocale(),
									"message.error.description.required"));
					return SUCCESS;
				}

				int submitedlangid = (Integer) reflanguages.get(langcount);

				TaxRateDescription desc = new TaxRateDescription();
				TaxRateDescriptionId id = new TaxRateDescriptionId();
				id.setLanguageId(submitedlangid);
				desc.setTaxDescription(description);
				desc.setId(id);

				descriptionsset.add(desc);

			}

			TaxRate taxRate = new TaxRate();
			taxRate.setMerchantId(ctx.getMerchantid());
			taxRate.setDescriptions(descriptionsset);
			taxRate.setTaxClassId(TaxConstants.DEFAULT_TAX_CLASS_ID);
			taxRate.setTaxRate(amount);
			taxRate.setPiggyback(this.isPiggyback());

			tservice.saveOrUpdateTaxRate(taxRate, this.getChoosecountry(), this
					.getChoosezone(), ctx.getMerchantid());

			setupTax();
			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));
			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			return SUCCESS;
		}

	}

	/**
	 * Edit an existing tax rate line
	 * 
	 * @return
	 * @throws Exception
	 */
	public String editTaxRate() throws Exception {

		if (super.getLanguages() == null || super.getLanguages().size() == 0) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.profile.storenotcreated"));
			return SUCCESS;
		}

		BigDecimal amount = null;

		try {

			super.gatherParameters();

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			TaxService tservice = (TaxService) ServiceFactory
					.getService(ServiceFactory.TaxService);
			HashSet descriptionsset = new HashSet();

			Integer priority = 0;

			if (taxlineaction == 0) {// edit

				// validate amunt
				try {

					amount = CurrencyUtil.validateCurrency(this
							.getTaxlinerate(), ctx.getCurrency());

				} catch (Exception e) {

					MessageUtil.addErrorMessage(super.getServletRequest(),
							LabelUtil.getInstance().getText(super.getLocale(),
									"message.error.rate.format"));
					this.setupTax();
					return SUCCESS;
				}

				// validate priority
				try {

					priority = new Integer(this.getTaxlineorder());

				} catch (Exception e) {

					MessageUtil.addErrorMessage(super.getServletRequest(),
							LabelUtil.getInstance().getText(super.getLocale(),
									"message.error.priority.title"));
					this.setupTax();
					return SUCCESS;
				}

				// descriptions

				Iterator i = super.reflanguages.keySet().iterator();
				while (i.hasNext()) {
					int langcount = (Integer) i.next();
					String description = (String) this.getDescriptions().get(
							langcount);

					if (StringUtils.isBlank(description)) {
						MessageUtil.addErrorMessage(super.getServletRequest(),
								LabelUtil.getInstance().getText(super.getLocale(),
										"message.error.description.required"));
						return SUCCESS;
					}

					int submitedlangid = (Integer) reflanguages.get(langcount);

					TaxRateDescription desc = new TaxRateDescription();
					TaxRateDescriptionId id = new TaxRateDescriptionId();
					id.setLanguageId(submitedlangid);
					id.setTaxRateId(this.getTaxlineid());
					desc.setTaxDescription(description);
					desc.setId(id);

					descriptionsset.add(desc);

				}

			}

			long taxRateId = this.getTaxlineid();

			TaxRate taxRate = null;

			if (taxRateId > 0) {// modify
				taxRate = tservice.getTaxRate(taxRateId);

			}

			if (taxRate == null) {// throw a tech difficulties
				log.error("No tax rate id defined for this tax rate edition");
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),"errors.technical"));
				return SUCCESS;
			}

			if (this.getTaxlineaction() == 1) {
				tservice.deleteTaxRate(taxRate);
			} else {
				taxRate.setDescriptions(descriptionsset);
				taxRate.setTaxClassId(this.getTaxlineclassid());
				taxRate.setTaxRate(amount);
				taxRate.setTaxPriority(priority);
				taxRate.setPiggyback(this.isPiggyback());
				tservice.saveOrUpdateTaxRate(taxRate, this.getChoosecountry(),
						this.getChoosezone(), ctx.getMerchantid());
			}

			setupTax();
			MessageUtil.addMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("message.confirmation.success"));
			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			return SUCCESS;
		}

	}

	public int getTaxlineaction() {
		return taxlineaction;
	}

	public void setTaxlineaction(int taxlineaction) {
		this.taxlineaction = taxlineaction;
	}

	public long getTaxlineid() {
		return taxlineid;
	}

	public void setTaxlineid(long taxlineid) {
		this.taxlineid = taxlineid;
	}

	public long getTaxlineclassid() {
		return taxlineclassid;
	}

	public void setTaxlineclassid(long taxlineclassid) {
		this.taxlineclassid = taxlineclassid;
	}

	public String getTaxlinerate() {
		return taxlinerate;
	}

	public void setTaxlinerate(String taxlinerate) {
		this.taxlinerate = taxlinerate;
	}

	public String getTaxlineorder() {
		return taxlineorder;
	}

	public void setTaxlineorder(String taxlineorder) {
		this.taxlineorder = taxlineorder;
	}

	public int getTaxclassaction() {
		return taxclassaction;
	}

	public void setTaxclassaction(int taxclassaction) {
		this.taxclassaction = taxclassaction;
	}

	public long getTaxclassid() {
		return taxclassid;
	}

	public void setTaxclassid(long taxclassid) {
		this.taxclassid = taxclassid;
	}

	public int getChoosecountry() {
		return choosecountry;
	}

	public void setChoosecountry(int choosecountry) {
		this.choosecountry = choosecountry;
	}

	public int getChoosezone() {
		return choosezone;
	}

	public void setChoosezone(int choosezone) {
		this.choosezone = choosezone;
	}

	public boolean isPiggyback() {
		return piggyback;
	}

	public void setPiggyback(boolean piggyback) {
		this.piggyback = piggyback;
	}

}



```
