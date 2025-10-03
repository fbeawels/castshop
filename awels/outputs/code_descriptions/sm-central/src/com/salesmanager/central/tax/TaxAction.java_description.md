# TaxAction.java

## Review

## 1. Summary

**Purpose**  
`TaxAction` is an abstract Struts‑2 action that prepares all data required for tax‑related pages (e.g. add/edit tax rates). It gathers tax rules, tax classes, supported languages and country information for a particular merchant and stores them in request/session attributes for use by JSPs.

**Key components**

| Component | Role |
|-----------|------|
| `prepare()` | Implements `Preparable`; sets up language lists, merchant store details, and calls abstract `setup()` for subclass‑specific logic. |
| `gatherParameters()` | Reads request parameters that identify the tax scheme and basis; populates instance fields and request attributes. |
| `setupTax()` | Loads tax rates and tax classes for the current merchant via `TaxService`, builds maps for the UI, and stores them in the request. |
| Fields (`schemeid`, `taxbasis`, `taxclass`, …) | Hold the values selected by the user or returned by the service layer. |
| `languages`, `reflanguages` | Provide the list of supported languages and a mapping from list index → language ID for the UI. |

**Notable design patterns & libraries**

* **Service/DAO layer** – `ServiceFactory` gives access to `MerchantService` and `TaxService`.  
* **Struts‑2/XWork** – implements `Preparable`, extends `CountrySelectBaseAction` (presumably another Struts action).  
* **Log4j** – used for error logging.  
* **Java Collections** – mostly `Map`, `Collection`, `List`, with a mix of raw and generic types.

---

## 2. Detailed Description

### Execution flow

1. **Instantiation**  
   A concrete subclass of `TaxAction` is instantiated by the Struts dispatcher.

2. **Preparation** (`prepare()`)  
   * Calls `setupTax()` to load tax data.  
   * Retrieves the `MerchantService` and fetches the current `MerchantStore`.  
   * If the store is missing, an error message is queued; otherwise, it obtains the supported languages map.  
   * Builds a list of `Language` objects (`languages`) and a reference map (`reflanguages`) that maps the list index to the language ID.  
   * Finally, `setup()` is invoked – this abstract method lets subclasses perform additional initialisation (e.g. populating form objects).

3. **Parameter gathering** (`gatherParameters()`)  
   When an action method is executed, it can call `gatherParameters()` to read the `SCHEMEID` and `taxbasis` request parameters, populate instance fields and set them as request attributes.

4. **Tax data setup** (`setupTax()`)  
   * Reads the `merchantid` from the session context.  
   * Calls `TaxService.getTaxRates(merchantid)` and `TaxService.getTaxClasses(merchantid)`.  
   * Builds a `TreeMap<String,Integer>` (`classtaxesid`) mapping tax class IDs to their titles, adding a default “Tax” entry with key `"1"`.  
   * Stores the list and map in request attributes (`taxlist`, `taxclassmap`) and keeps a reference copy (`taxmap`).  
   * Sets the `countryId` and the country session attribute.

5. **Action execution**  
   The subclass’s action method (e.g. `execute`, `save`, `delete`) will operate on the fields populated above and then return a Struts result string.

### Assumptions & constraints

| Assumption | Impact |
|------------|--------|
| `Context` bean is present in session | Action fails silently if missing |
| `MerchantStore.getGetSupportedLanguages()` returns a `Map<Integer, Language>` | The code relies on key/value order being stable; the list `languages` is derived from `values()` which is not ordered |
| `TaxService` returns non‑null collections or `null` is handled | Null‑checked but may still lead to empty UI |
| Request parameters `SCHEMEID` and `taxbasis` are always present | `gatherParameters()` throws generic `Exception` otherwise |
| Struts‑2 `Preparable` contract is honoured | The action will only work correctly when `prepare()` is called before any action method |

### Architecture & design choices

* **Separation of concerns** – The class handles only data preparation; actual business logic resides in `TaxService`.  
* **Extensibility** – `setup()` is abstract, enabling subclasses to extend behaviour without touching the common setup.  
* **Coupling to Struts** – By extending `CountrySelectBaseAction` and implementing `Preparable`, the class is tightly bound to the Struts 2 request lifecycle.  
* **Logging** – Uses Log4j; errors are logged but not propagated, which may mask failures in the UI.

---

## 3. Functions/Methods

| Method | Purpose | Parameters | Returns | Side‑effects |
|--------|---------|------------|---------|--------------|
| `public void prepare()` | Initializes data before any action; loads tax, language, and merchant information; invokes `setup()`. | none | void | Sets instance fields; populates request/session attributes; logs errors. |
| `protected Map gatherParameters() throws Exception` | Reads `SCHEMEID` and `taxbasis` from request, validates, stores in fields and request. | none | `Map` of parameters | Sets instance fields `schemeid`, `taxbasis`; sets request attributes. |
| `public String setupTax() throws Exception` | Loads tax rates and tax classes for the current merchant; builds maps for UI. | none | `SUCCESS` or `ERROR` string | Sets `taxlist`, `taxmap`; stores in request; sets `countryId`. |
| `protected abstract void setup() throws Exception` | Hook for subclasses to perform additional setup. | none | void | – |
| Getter/Setter methods (`getSchemeid`, `setSchemeid`, …) | Standard JavaBean accessors for action properties. | – | – | – |
| `public int getCountryId()` / `setCountryId(int)` | Accessor for the merchant’s country. | – | – | – |

**Utility / reusable snippets**

* `setupTax()` could be extracted to a helper service or utility class to avoid duplicating tax‑loading logic in multiple actions.
* The language list and `reflanguages` mapping logic is duplicated in several actions; a reusable method would reduce boilerplate.

---

## 4. Dependencies

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.log4j.Logger` | Third‑party logging | Classic Log4j (not SLF4J). |
| `org.hibernate.HibernateException` | Third‑party exception | Indicates use of Hibernate for persistence. |
| `com.opensymphony.xwork2.Preparable` | Struts‑2 / XWork | Standard interface for actions that need pre‑processing. |
| `com.salesmanager.central.BaseAction` / `CountrySelectBaseAction` | Local framework | Provides common action utilities (e.g. `getServletRequest()`). |
| `com.salesmanager.central.profile.Context` | Local session bean | Holds `merchantid` and `countryid`. |
| `com.salesmanager.core.entity.*` | Local JPA/Hibernate entities | Tax, MerchantStore, Language. |
| `com.salesmanager.core.service.*` | Local service layer | `MerchantService`, `TaxService`. |
| `com.salesmanager.core.util.*` | Local utilities | `LabelUtil`, `MessageUtil`. |
| Java EE `HttpServletRequest` / `HttpSession` | Platform | Implicitly used via `getServletRequest()` inherited from `BaseAction`. |

All dependencies are either third‑party libraries that are standard in a Java EE / Struts application or internal to the Sales Manager codebase.

---

## 5. Additional Notes

### Edge cases / potential bugs

1. **Raw types** – `Collection`, `Map`, and `List` are used without generics (except for `languages`). This leads to unchecked conversions, potential `ClassCastException`s, and loss of type safety.  
2. **NPE risk** – `getServletRequest()` or session attributes may be `null`; the code does not guard against it.  
3. **Silent failures** – Exceptions in `prepare()` are logged but not rethrown; a user could see an empty page without an error message.  
4. **Hard‑coded tax class** – The line `classtaxesid.put("1", "Tax")` may clash if a real tax class has ID `1`.  
5. **Locale‑dependent message keys** – `MessageUtil.addErrorMessage` uses `ProfileConstants.context`; if the context is missing, the message queue may not be available.  
6. **Performance** – `setupTax()` loads all tax rates and classes on every request; for merchants with many rates this could be heavy. Lazy loading or pagination could be considered.  

### Potential improvements

| Area | Suggestion |
|------|------------|
| **Generics** | Replace raw collections with typed generics: `Map<String,Integer>`, `List<Language>`, etc. |
| **Exception handling** | Throw specific exceptions (e.g. `MissingParameterException`) and let Struts handle them via result mappings. |
| **Logging** | Use SLF4J façade; add request context to logs (merchantid, countryid). |
| **Decouple from Struts** | Move tax‑loading logic to a dedicated service (e.g. `TaxPreparationService`) so that `TaxAction` is thin. |
| **Internationalization** | Use constants for message keys; externalise "Tax" default string. |
| **Thread‑safety** | Remove any static mutable state; ensure per‑request objects are used. |
| **Unit tests** | Write tests for `setupTax()` and `gatherParameters()` with mock services. |
| **Code style** | Replace `new Integer(scheme)` with `Integer.parseInt(scheme)`. |

### Future extensions

* **Tax rules editing UI** – Additional actions for creating, updating, deleting tax rates; reuse `prepare()` logic.  
* **Audit trail** – Record changes to tax configuration per merchant.  
* **Validation** – Add server‑side validation for tax rate ranges, overlaps, and duplicate classes.  
* **Performance tuning** – Cache tax classes per merchant; use async loading for large lists.  

Overall, `TaxAction` provides the essential plumbing for tax‑related pages but would benefit from modern Java best practices, improved error handling, and clearer separation of concerns.

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

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;
import java.util.TreeMap;

import org.apache.log4j.Logger;
import org.hibernate.HibernateException;

import com.opensymphony.xwork2.Preparable;
import com.salesmanager.central.BaseAction;
import com.salesmanager.central.CountrySelectBaseAction;
import com.salesmanager.central.profile.Context;
import com.salesmanager.central.profile.ProfileConstants;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.Language;
import com.salesmanager.core.entity.tax.TaxClass;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.tax.TaxService;
import com.salesmanager.core.util.LabelUtil;
import com.salesmanager.core.util.MessageUtil;

public abstract class TaxAction extends CountrySelectBaseAction implements
		Preparable {

	protected final static String SCHEME = "S";
	protected final static String TAX_BASIS = "T";

	private int schemeid = 0;
	private String taxbasis = "";
	private String taxclass = "";

	private String descen = "";
	private String descfr = "";

	private Collection taxlist;
	private Map taxmap;

	private Collection<Language> languages;// used in the page as an index
	protected Map<Integer, Integer> reflanguages = new HashMap();// reference
																	// count -
																	// languageId

	// descriptions
	private List<String> descriptions = new ArrayList<String>();

	private int countryId;

	private Logger log = Logger.getLogger(TaxAction.class);

	protected abstract void setup() throws Exception;

	public void prepare() {

		try {

			setupTax();

			MerchantService service = (MerchantService) ServiceFactory
					.getService(ServiceFactory.MerchantService);

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			super.prepareSelections(ctx.getCountryid());

			MerchantStore mstore = service.getMerchantStore(merchantid);

			if (mstore == null) {
				MessageUtil.addErrorMessage(super.getServletRequest(),
						LabelUtil.getInstance().getText(super.getLocale(),
								"errors.profile.storenotcreated"));
			} else {

				Map languagesMap = mstore.getGetSupportedLanguages();

				languages = languagesMap.values();// collection reverse the map

				super.getServletRequest().setAttribute("laanguages", languages);

				// int count = languagesMap.size()-1;
				int count = 0;
				Iterator langit = languagesMap.keySet().iterator();
				while (langit.hasNext()) {
					Integer langid = (Integer) langit.next();
					Language lang = (Language) languagesMap.get(langid);
					reflanguages.put(count, langid);
					count++;
				}

			}

			setup();

		} catch (Exception e) {
			log.error(e);
		}

	}

	protected Map gatherParameters() throws Exception {

		String scheme = super.getServletRequest().getParameter("SCHEMEID");

		if (scheme == null)
			throw new Exception("gatherParameters() Did not received scheme");

		String taxbasis = super.getServletRequest().getParameter("taxbasis");

		if (taxbasis == null)
			throw new Exception("gatherParameters() Did not received taxbasis");

		Map p = new HashMap();
		p.put(SCHEME, scheme);
		p.put(TAX_BASIS, taxbasis);

		int schid = new Integer(scheme);
		this.schemeid = schid;
		this.taxbasis = taxbasis;

		super.getServletRequest().setAttribute("SCHEMEID", schid);
		super.getServletRequest().setAttribute("taxbasis", taxbasis);

		return p;

	}

	/**
	 * -- ALWAYS INVOKED -- Get tax configured for a given merchantid
	 * 
	 * @return
	 * @throws Exception
	 */
	public String setupTax() throws Exception {

		try {

			Context ctx = (Context) super.getServletRequest().getSession()
					.getAttribute(ProfileConstants.context);
			Integer merchantid = ctx.getMerchantid();

			// set basic information
			this.setCountryId(ctx.getCountryid());
			super.getServletRequest().getSession().setAttribute("COUNTRY",
					ctx.getCountryid());

			TaxService taxService = (TaxService) ServiceFactory
					.getService(ServiceFactory.TaxService);

			taxlist = taxService.getTaxRates(merchantid);

			if (taxlist == null) {
				taxlist = new ArrayList();
			}

			// iterate and get tax class ids and name
			Map classtaxesid = new TreeMap();

			List txscl = taxService.getTaxClasses(merchantid);

			if (txscl != null) {
				Iterator it = txscl.iterator();
				while (it.hasNext()) {
					TaxClass x = (TaxClass) it.next();
					classtaxesid.put(String.valueOf(x.getTaxClassId()), x
							.getTaxClassTitle());
				}
			}

			classtaxesid.put("1", "Tax");

			super.getServletRequest().setAttribute("taxclassmap", classtaxesid);
			super.getServletRequest().setAttribute("taxlist", taxlist);

			this.setTaxmap(classtaxesid);

			return SUCCESS;

		} catch (HibernateException e) {
			log.error(e);
			return ERROR;
		}

	}

	public int getSchemeid() {
		return schemeid;
	}

	public void setSchemeid(int schemeid) {
		this.schemeid = schemeid;
	}

	public String getTaxbasis() {
		return taxbasis;
	}

	public void setTaxbasis(String taxbasis) {
		this.taxbasis = taxbasis;
	}

	public String getTaxclass() {
		return taxclass;
	}

	public void setTaxclass(String taxclass) {
		this.taxclass = taxclass;
	}

	public String getDescen() {
		return descen;
	}

	public void setDescen(String descen) {
		this.descen = descen;
	}

	public String getDescfr() {
		return descfr;
	}

	public void setDescfr(String descfr) {
		this.descfr = descfr;
	}

	public Collection getTaxlist() {
		return taxlist;
	}

	public void setTaxlist(Collection taxlist) {
		this.taxlist = taxlist;
	}

	public Map getTaxmap() {
		return taxmap;
	}

	public void setTaxmap(Map taxmap) {
		this.taxmap = taxmap;
	}

	public Collection<Language> getLanguages() {
		return languages;
	}

	public void setLanguages(Collection<Language> languages) {
		this.languages = languages;
	}

	public List<String> getDescriptions() {
		return descriptions;
	}

	public void setDescriptions(List<String> descriptions) {
		this.descriptions = descriptions;
	}

	public int getCountryId() {
		return countryId;
	}

	public void setCountryId(int countryId) {
		this.countryId = countryId;
	}

}



```
