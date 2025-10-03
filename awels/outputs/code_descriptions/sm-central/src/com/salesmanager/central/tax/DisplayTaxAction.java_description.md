# DisplayTaxAction.java

## Review

## 1. Summary  

**Purpose** –  
`DisplayTaxAction` is a Struts‑style action class that drives the “Set Up Tax” wizard in a merchant‑centric application. It handles the presentation of tax‑basis and tax‑scheme options, stores the chosen configuration, and creates the necessary tax line templates for the merchant.

**Key components**  

| Component | Role |
|-----------|------|
| `displayTaxBasis()` / `displayTaxClass()` | Convenience wrappers that simply forward to `displayTax()`. |
| `setup()` | Loads the current tax configuration from the database and populates request attributes. |
| `displayTax()` | Determines whether the merchant has a tax configuration; if not, it selects a default scheme (US/CA/EU/None) based on the merchant’s country and shows the wizard’s first page. |
| `configureTax()` | Cleans any existing tax configuration, then, based on the selected scheme, creates the appropriate tax line templates (US, CA, EU or No‑Scheme) and forwards to the next wizard step. |
| `resetTax()` / `removeTax()` | Helper actions that delete the merchant’s tax configuration. |
| `setInitialSetup()` | Persists the chosen scheme and basis in `MerchantConfiguration` objects. |
| `persistConfiguration()` | Utility to save or update a `MerchantConfiguration` via `MerchantService`. |

**Design patterns / frameworks**  

* **Service Factory** – `ServiceFactory.getService()` retrieves EJB‑style service objects.  
* **Template Method** – The wizard flow is linear (display → configure → reset) with each step implemented in a separate method.  
* **Action/Servlet** – Extends `TaxAction`, implying integration with Struts (or a similar MVC framework).  

## 2. Detailed Description  

### Execution Flow  

1. **Request starts** – The user requests one of the wizard pages (e.g., “select tax basis”).  
2. **`setup()`** is called first to load any existing tax configuration into request attributes.  
3. **`displayTax()`** decides if the wizard needs to start from scratch or continue an existing configuration.  
4. **User selects a scheme** – The form submits to `configureTax()`.  
5. **`configureTax()`** cleans previous config, records the new scheme/basis, creates the tax line templates, and forwards to the next step.  
6. **`resetTax()` / `removeTax()`** can be invoked to delete the configuration and return the wizard to its initial state.  

### Initialization / Cleanup  

* `setup()` runs before the wizard page is rendered, ensuring the session contains the latest configuration.  
* `cleanupTax()` (invoked by `removeTax()`, `resetTax()`, and `configureTax()`) deletes all tax‑related configuration entries for the merchant.  
* There is no explicit resource cleanup beyond the service calls; all resources are managed by the container.

### Dependencies & Assumptions  

* The merchant’s context (`Context`) is stored in the session under `ProfileConstants.context`.  
* `MerchantService` and `TaxService` are available via `ServiceFactory`.  
* Tax constants (`TaxConstants`) are used to identify schemes and configuration keys.  
* The code assumes that the action instance is **request‑scoped**; otherwise, shared fields (`configurationScheme`, `configuration`) could cause race conditions.

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑effects |
|--------|---------|------------|--------|--------------|
| `displayTaxBasis()` | Entry point for the “tax basis” page. Calls `displayTax()`. | None | `String` – “SUCCESS” | Sets request attributes via `displayTax()` |
| `displayTaxClass()` | Entry point for the “tax class” page. Calls `displayTax()`. | None | `String` – “SUCCESS” | Sets request attributes via `displayTax()` |
| `setup()` | Loads current tax configuration into request attributes. | None | void | Sets `SCHEMEID`, `taxbasis` in request; populates `configuration` field |
| `displayTax()` | Core wizard logic: shows options if no config, otherwise proceeds. | None | `String` – “showoptions” or “SUCCESS” | Populates request attributes; sets `configurationScheme` |
| `removeTax()` | Public entry that cleans tax config. | None | `String` – “SUCCESS” or “error” | Invokes `cleanupTax()` |
| `cleanupTax()` | Deletes all tax‑related config for the current merchant. | None | void | Calls `TaxService.deleteTaxConfiguration()` |
| `configureTax()` | Finalizes user’s scheme selection, creates tax lines, and forwards. | None | `String` – “SUCCESS” or “ERROR” | Persists scheme/basis, creates tax templates, sets request attributes |
| `resetTax()` | Alias for `cleanupTax()` used in the UI. | None | `String` – “SUCCESS” or “ERROR” | Invokes `cleanupTax()` |
| `persistConfiguration(MerchantConfiguration)` | Persists a single configuration entry. | `MerchantConfiguration` | void | Calls `MerchantService.saveOrUpdateMerchantConfiguration()` |
| `setInitialSetup(String scheme, String basis)` | Writes scheme and basis into merchant configuration. | `String` scheme, `String` basis | void | Calls `persistConfiguration()` twice if needed |
| `getConfigurationScheme()` | Getter. | None | `int` | – |
| `setConfigurationScheme(int)` | Setter. | `int` | – | – |

**Reusable / Utility methods** – `persistConfiguration()` and `setInitialSetup()` are thin wrappers around the service layer and could be extracted to a dedicated helper class if reused elsewhere.

## 4. Dependencies  

| Library / API | Type | Notes |
|---------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party | Standard logging. |
| `com.salesmanager.core.service.*` | Third‑party | Service layer façade. |
| `com.salesmanager.core.entity.*` | Third‑party | JPA entities for merchant configuration and tax templates. |
| `com.salesmanager.core.util.*` | Third‑party | Localization (`LabelUtil`) and messaging (`MessageUtil`). |
| `com.salesmanager.central.profile.*` | Custom | Session context holder. |
| `com.salesmanager.central.tax.*` | Custom | Helper (`TaxTemplateHelper`) used for creating tax lines. |
| Java SE (Collection, Iterator) | Standard | Raw types used; generics could improve type safety. |

No platform‑specific dependencies are visible; the code should run on any Java EE container that supports the referenced services.

## 5. Additional Notes & Recommendations  

### Code‑quality & Safety  

* **Thread‑safety** – The action stores mutable state (`configuration`, `configurationScheme`) in instance fields. In a typical Struts‑2 configuration actions are **request‑scoped**, but if the container ever reuses instances, these fields could leak data between users. Consider making them local variables or using request/session scopes explicitly.  
* **Deprecated API** – `new Integer(0)` and the use of raw `Collection`/`Iterator` types should be updated to modern generics (`List<...>`, `for (TaxRateTaxTemplate tsv : tsvlist)`).
* **Exception handling** – `cleanupTax()` swallows all exceptions after logging, potentially hiding issues from the caller. Either propagate the exception or at least set an error message for the user.  
* **Missing `@Override`** – All methods that override base class methods should be annotated for clarity and compile‑time safety.  
* **Duplicate logic** – `displayTaxBasis()` and `displayTaxClass()` simply delegate to `displayTax()`. They could be removed, and the mapping could be handled at the framework level instead of having empty methods.  
* **Magic strings** – `"MODULE_TAX_"` and `"errors.technical"` should be constants or resources.  
* **Null‑checks** – In `displayTax()`, the code accesses `tsvlist.size()` and then `i.next()` without confirming the iterator has a next element. This can throw `NoSuchElementException`.  

### Functional Enhancements  

1. **Support multiple tax rates per scheme** – The current implementation only considers the first entry from `findByGeoZoneCountryId`. If multiple tax rates exist, the wizard should present a selection.  
2. **Persisting existing configuration** – `setInitialSetup()` overwrites the scheme and basis each time but does not delete old entries. This could result in duplicate keys. Adding a check before persisting would prevent that.  
3. **User feedback** – When a configuration is removed or reset, a success message could be shown to the user.  
4. **Internationalisation** – The page title uses a label key; other UI strings (e.g., error messages, button labels) should also be fetched via `LabelUtil`.  
5. **Testing** – Unit tests for the wizard logic would help catch regressions, especially around the conditional flow in `displayTax()` and `configureTax()`.  

### Performance & Security  

* The service calls (`getConfiguration`, `deleteTaxConfiguration`) are executed synchronously; if the database is slow, the UI may lag. Consider async background processing for heavy operations.  
* No explicit input validation is present. All user‑supplied data (e.g., selected scheme) originates from the session or request parameters; ensure that the framework validates these values before invoking the action.  

---

**Overall**, the class implements the required wizard flow but can benefit from modern Java practices, better thread safety, and more robust error handling. The design is straightforward and integrates cleanly with the service layer, but refactoring for clarity and safety would improve maintainability.

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

import java.util.Collection;
import java.util.Iterator;

import org.apache.log4j.Logger;

import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.constants.TaxConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.tax.TaxRateTaxTemplate;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.tax.TaxService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;

public class DisplayTaxAction extends TaxAction {

	private Logger log = Logger.getLogger(DisplayTaxAction.class);

	private int configurationScheme = TaxConstants.NO_SCHEME;

	private ConfigurationResponse configuration = null;

	/**
	 * Displays tax basis options
	 * 
	 * @return
	 * @throws Exception
	 */
	public String displayTaxBasis() throws Exception {
		String displayTax = this.displayTax();// will set all required working
												// variables

		return SUCCESS;
	}

	/**
	 * Display tax class options
	 * 
	 * @return
	 * @throws Exception
	 */
	public String displayTaxClass() throws Exception {
		String displayTax = this.displayTax();// will set all required working
												// variables

		return SUCCESS;
	}

	public void setup() throws Exception {

		super.getServletRequest().setAttribute("SCHEMEID", new Integer(0));

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		ConfigurationRequest request = new ConfigurationRequest(merchantid,
				true, "MODULE_TAX_");

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		configuration = mservice.getConfiguration(request);

		if (configuration != null
				&& configuration.getMerchantConfigurations().size() > 0) {

			MerchantConfiguration conf = (MerchantConfiguration) configuration
					.getMerchantConfiguration(TaxConstants.MODULE_TAX_SCHEME);
			if (conf != null) {

				String value = conf.getConfigurationValue();
				try {
					int schemeid = Integer.parseInt(value);
					this.setConfigurationScheme(schemeid);

					// super.getServletRequest().setAttribute("taxbasis",taxbasis);
					super.getServletRequest().setAttribute("SCHEMEID",
							new Integer(this.getConfigurationScheme()));
					super.setSchemeid(schemeid);

				} catch (NumberFormatException nfe) {
					log.error("Cannot parse " + value + " for merchantid "
							+ merchantid);
				}

			}

			conf = (MerchantConfiguration) configuration
					.getMerchantConfiguration(TaxConstants.MODULE_TAX_BASIS);
			if (conf != null) {
				super.getServletRequest().setAttribute("taxbasis",
						conf.getConfigurationValue());
				super.setTaxbasis(conf.getConfigurationValue());

			}

		}

	}

	/**
	 * Step 1 Display the user's tax option if no tax scheme has been selected.
	 * Will check first if the user is configured for tax and then what kind of
	 * scheme the user has
	 * 
	 * @return
	 * @throws Exception
	 */
	public String displayTax() throws Exception {

		Context ctx = super.getContext();
		Integer merchantid = ctx.getMerchantid();

		try {
			
			super.setPageTitle("label.setuptax.title");

			if (configuration == null
					|| configuration.getMerchantConfigurations().size() == 0) {// no
																				// tax
																				// configured
																				// yet

				TaxService tservice = (TaxService) ServiceFactory
						.getService(ServiceFactory.TaxService);
				Collection<TaxRateTaxTemplate> tsvlist = tservice
						.findByGeoZoneCountryId(ctx.getCountryid());

				if (tsvlist != null && tsvlist.size() > 0) {
					Iterator i = tsvlist.iterator();
					TaxRateTaxTemplate tsv = (TaxRateTaxTemplate) i.next();
					this.setConfigurationScheme(tsv.getZoneToGeoZone()
							.getGeoZone().getSchemeid());
				} else {
					this.setConfigurationScheme(0);
				}

				super.getServletRequest().setAttribute("SCHEMEID",
						new Integer(this.getConfigurationScheme()));
				return "showoptions";

			}

			// need to get the taxrates !!!!!!

			return SUCCESS;

		} catch (Exception e) {
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			log.error(e);
			return "showoptions";
		}

	}

	public String removeTax() throws Exception {
		try {
			cleanupTax();
			return SUCCESS;
		} catch (Exception e) {
			return "error";
		}
	}

	/**
	 * Remove entries from MerchantConfiguration ->Tax Basis, Tax Scheme
	 * 
	 * @throws Exception
	 */
	private void cleanupTax() throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		try {

			TaxService service = (TaxService) ServiceFactory
					.getService(ServiceFactory.TaxService);
			service.deleteTaxConfiguration(merchantid);

		} catch (Exception e) {
			log.error(e);
		}

	}

	/**
	 * Step 2 - WIZARD - Forward the request to the appropriate configuration
	 * page
	 * 
	 * @return
	 * @throws Exception
	 */
	public String configureTax() throws Exception {
		
		super.setPageTitle("label.setuptax.title");
		int selectedScheme = this.getConfigurationScheme();
		this.cleanupTax();

		try {

			// TaxLineBO line = null;
			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);

			if (selectedScheme == TaxConstants.US_SCHEME) {
				this.setInitialSetup(String.valueOf(TaxConstants.CA_SCHEME),
						TaxConstants.SHIPPING_TAX_BASIS);

				TaxTemplateHelper.createUSTaxLines(ctx);

				// get merchant country and region
				super.getServletRequest().setAttribute("taxbasis",
						TaxConstants.SHIPPING_TAX_BASIS);
				super.getServletRequest().setAttribute("scheme",
						String.valueOf(TaxConstants.US_SCHEME));

			} else if (selectedScheme == TaxConstants.CA_SCHEME) {
				this.setInitialSetup(String.valueOf(TaxConstants.CA_SCHEME),
						TaxConstants.SHIPPING_TAX_BASIS);

				TaxTemplateHelper.createCATaxLines(ctx);

				super.getServletRequest().setAttribute("taxbasis",
						TaxConstants.SHIPPING_TAX_BASIS);
				super.getServletRequest().setAttribute("scheme",
						String.valueOf(TaxConstants.CA_SCHEME));
			} else if (selectedScheme == TaxConstants.EU_SCHEME) {
				this.setInitialSetup(String.valueOf(TaxConstants.EU_SCHEME),
						TaxConstants.SHIPPING_TAX_BASIS);

				TaxTemplateHelper.createEUTaxLines(ctx);

				super.getServletRequest().setAttribute("taxbasis",
						TaxConstants.SHIPPING_TAX_BASIS);
				super.getServletRequest().setAttribute("scheme",
						String.valueOf(TaxConstants.EU_SCHEME));

			} else {
				this.setInitialSetup(String.valueOf(TaxConstants.NO_SCHEME),
						TaxConstants.SHIPPING_TAX_BASIS);
				// line = new NoSchemeTaxLineBO();

				super.getServletRequest().setAttribute("scheme",
						String.valueOf(TaxConstants.NO_SCHEME));
			}

			return SUCCESS;

		} catch (Exception e) {
			log.error(e);
			MessageUtil.addErrorMessage(super.getServletRequest(), LabelUtil
					.getInstance().getText("errors.technical"));
			return ERROR;
		}

	}

	public String resetTax() throws Exception {
		try {
			this.cleanupTax();
			return SUCCESS;
		} catch (Exception e) {
			log.error(e);
			return ERROR;
		}
	}

	private void persistConfiguration(MerchantConfiguration obj)
			throws Exception {

		MerchantService mservice = (MerchantService) ServiceFactory
				.getService(ServiceFactory.MerchantService);
		mservice.saveOrUpdateMerchantConfiguration(obj);

	}

	/**
	 * Step 3 Initial setup will set tax scheme and tax basis User selects
	 * custom from the initial choices
	 * 
	 * @throws Exception
	 */
	private void setInitialSetup(String scheme, String basis) throws Exception {

		Context ctx = (Context) super.getServletRequest().getSession()
				.getAttribute(ProfileConstants.context);
		Integer merchantid = ctx.getMerchantid();

		if (scheme != null) {

			// tax scheme
			MerchantConfiguration obj = new MerchantConfiguration();
			obj
					.setConfigurationKey(com.salesmanager.core.constants.TaxConstants.MODULE_TAX_SCHEME);
			obj.setConfigurationValue(scheme);
			obj.setMerchantId(ctx.getMerchantid());

			this.persistConfiguration(obj);

		}

		if (basis != null) {

			// tax basis
			MerchantConfiguration obj = new MerchantConfiguration();
			obj
					.setConfigurationKey(com.salesmanager.core.constants.TaxConstants.MODULE_TAX_BASIS);
			obj.setConfigurationValue(basis);
			obj.setMerchantId(ctx.getMerchantid());

			this.persistConfiguration(obj);

		}

	}

	public int getConfigurationScheme() {
		return configurationScheme;
	}

	public void setConfigurationScheme(int configurationScheme) {
		this.configurationScheme = configurationScheme;
	}

}



```
