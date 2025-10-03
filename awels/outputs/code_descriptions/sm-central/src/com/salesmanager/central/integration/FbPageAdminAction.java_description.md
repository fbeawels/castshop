# FbPageAdminAction.java

## Review

## 1. Summary  
**Purpose & Functionality**  
`FbPageAdminAction` is a Struts‑style action class that manages the creation, display, and editing of a Facebook‑integrated page within the SalesManager e‑commerce platform. It interacts with core services (`ReferenceService`, `MerchantService`) to persist `Page`, `Portlet`, and configuration data, and exposes various attributes to the view layer via the servlet request.

**Key Components**  
| Class | Role |
|-------|------|
| `FbPageAdminAction` | Action handler for CRUD operations on the Facebook page. |
| `Page` | Domain entity representing a page (title, properties, visibility flags). |
| `Portlet` | Domain entity representing widgets or labels that can be placed on the page. |
| `ReferenceService` | Service layer to access and persist reference data (pages, modules, labels). |
| `MerchantService` | Service layer to handle merchant‑specific configurations. |
| `ConfigurationFieldUtil` | Utility for parsing and serialising field configurations. |

**Design Patterns / Frameworks**  
* **Service Locator / Factory** – `ServiceFactory.getService()` is used to obtain service instances.  
* **DAO‑like persistence** – Entities are persisted via service methods such as `saveOrUpdatePage`.  
* **MVC** – The action prepares data and forwards to a view (JSP) via the servlet request.  
* **Apache Commons Lang** – Used for `StringUtils.isBlank`.  
* **Log4j** – Logging is referenced but not actively used in the class.  

---

## 2. Detailed Description  
### Execution Flow  
1. **`createPage()`**  
   * Sets a page title.  
   * Instantiates a new `Page` object with default properties (disabled, invisible, style = 1).  
   * Persists it via `ReferenceService.saveOrUpdatePage`.  
   * Returns `SUCCESS`.

2. **`displayPage()`**  
   * Retrieves the merchant store and the associated country ISO code.  
   * Loads the Facebook page entity by title (`FB_PAGE`).  
   * If present, copies various property values into the request for the view.  
   * Builds a list of *module* portlets (from `CoreModuleService`) and a list of *label* portlets (from `DynamicLabel`).  
   * Loads field configurations for the modules (via `ConfigurationFieldUtil`) and, if merchant‑specific values exist, parses them into `fieldValues`.  
   * Loads configured portlets for the page and groups them by column ID.  
   * Determines the list of *available* portlets that are not yet configured (`portletList`).  
   * Sets multiple request attributes (`paageTemplate`, `ApplicationID`, etc.) for the JSP.  
   * Returns `SUCCESS`.

3. **`editPageHeader()` & `editPageConfig()`**  
   * These methods update specific fields of an existing page: header, visibility, security, and all property fields (`property1`‑`property10`).  
   * They load the current page, copy fields from the incoming `Page` instance (`this.page`), persist, and return `SUCCESS`.

4. **Getters / Setters**  
   * Standard bean properties to expose fields to the view and framework.

### Assumptions & Constraints  
* The `page` field is expected to be populated by the framework (likely via form parameters).  
* The `displayPage()` method assumes a `page` exists; if not, it simply returns `SUCCESS`.  
* No explicit validation of input data – potential for null or malformed values.  
* The code uses raw `Collection` and raw `List`/`Map` without generics, which can lead to unchecked warnings.  
* The class relies heavily on legacy services and a pre‑defined schema for page properties (property1‑10).  

### Architecture & Design Choices  
* **Legacy Service Layer** – The design mirrors older Struts 1 action patterns, with service lookups done at runtime.  
* **Hardcoded Property Names** – `property1`–`property10` are used instead of a more expressive DTO.  
* **Mixed Responsibility** – The action handles both presentation logic (request attributes) and data persistence, which can be difficult to test.  

---

## 3. Functions/Methods  

| Method | Purpose | Parameters | Return | Side‑Effects |
|--------|---------|------------|--------|--------------|
| `createPage()` | Initializes a new Facebook page with default settings. | none | `String` (`SUCCESS` or exception) | Persists a new `Page` entity. |
| `displayPage()` | Loads the Facebook page and its associated portlets, fields, and configurations, preparing data for the view. | none | `String` (`SUCCESS` or exception) | Sets many request attributes; populates `portletList`, `portletsColumn`. |
| `editPageHeader()` | Updates the page header field. | none | `String` | Persists updated `Page`. |
| `editPageConfig()` | Updates page visibility, security, and all property fields. | none | `String` | Persists updated `Page`. |
| `getHtmlCode()`, `setHtmlCode(String)` | Bean property for potential HTML snippets. | none / `String` | `String` | none. |
| `getPortletList()`, `setPortletList(List)` | Exposes the list of available portlets. | none / `List` | `List` | none. |
| `getDisplayedPortlets()`, `setDisplayedPortlets(Map)` | Exposes the map of displayed portlets per column. | none / `Map` | `Map` | none. |
| `getPage()`, `setPage(Page)` | Getter/setter for the `Page` bean used in form submission. | none / `Page` | `Page` | none. |

**Reusable / Utility Methods** – The class does not expose any dedicated helper methods; repeated logic (e.g., retrieving `ReferenceService`) is duplicated across actions.

---

## 4. Dependencies  

| Dependency | Type | Notes |
|------------|------|-------|
| `org.apache.commons.lang.StringUtils` | Third‑party | Utility for string checks. |
| `org.apache.log4j.Logger` | Third‑party | Logger declared but unused. |
| `com.salesmanager.central.BaseAction` | Project | Base Struts‑style action providing context and helpers (`setPageTitle`, `setSuccessMessage`). |
| `com.salesmanager.core.*` | Project | Core domain entities (`Page`, `Portlet`, etc.) and services (`ReferenceService`, `MerchantService`, `ConfigurationFieldUtil`). |
| `com.salesmanager.core.util.*` | Project | Utilities for country lookups and configuration parsing. |
| `ServiceFactory` | Project | Service locator pattern to obtain service instances. |

**Platform Specific** – None; all dependencies are pure Java and framework‑agnostic, aside from the custom SalesManager infrastructure.

---

## 5. Additional Notes  

### Edge Cases & Missing Robustness  
* **Null/Empty Values** – The code assumes non‑null values for many properties (e.g., `page.getProperty1()`), which could throw `NullPointerException` if the page is incomplete.  
* **Thread Safety** – Instance fields (`page`, `portletList`, `displayedPortlets`) are shared across requests if the action is not scoped per request; this can lead to data leakage in a multi‑threaded container.  
* **Raw Types** – The use of raw collections (`List`, `Map`) without generics increases the risk of `ClassCastException` and obscures the intended type contracts.  
* **Error Handling** – Methods declare `throws Exception` but do not provide meaningful user‑facing error messages or logging.  
* **Hardcoded Property Keys** – The class uses property indices (1‑10) rather than a named structure; future changes to property layout would require code changes.

### Potential Improvements  
1. **Use Generics** – Replace raw collections with parameterised types (`List<Portlet>`, `Map<String, Field>`) to improve type safety.  
2. **Encapsulate Page Properties** – Introduce a `PageConfig` DTO or use a `Map<String, String>` to avoid magic property indices.  
3. **Refactor Repeated Service Lookups** – Create private helper methods to fetch `ReferenceService` or `MerchantService`.  
4. **Add Validation** – Validate `Page` fields before persisting (e.g., non‑blank title, required properties).  
5. **Scope Management** – Ensure the action is request‑scoped (e.g., by using Struts2 `@Result` or Spring MVC) to prevent shared mutable state.  
6. **Logging** – Enable and use Log4j for debugging and audit trails.  
7. **Unit Tests** – The class is currently difficult to test due to tight coupling; extract service interaction into a separate service layer and mock dependencies.  
8. **Internationalisation** – Use `i18n` keys consistently (the code already sets page title via key).  

### Summary  
The `FbPageAdminAction` serves a clear business purpose—managing a Facebook‑specific page within the SalesManager platform. However, it exhibits several code‑quality issues typical of legacy Struts 1 actions: raw collections, hardcoded property indices, and shared mutable state. Addressing these concerns would improve maintainability, testability, and robustness, especially if the platform evolves toward more modern frameworks or container environments.

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
package com.salesmanager.central.integration;


import java.util.ArrayList;
import java.util.Collection;
import java.util.HashMap;
import java.util.List;
import java.util.Map;



import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;



import com.salesmanager.central.BaseAction;
import com.salesmanager.core.constants.Constants;
import com.salesmanager.core.constants.IntegrationConstants;
import com.salesmanager.core.constants.LabelConstants;
import com.salesmanager.core.entity.merchant.MerchantConfiguration;
import com.salesmanager.core.entity.merchant.MerchantStore;
import com.salesmanager.core.entity.reference.CoreModuleService;
import com.salesmanager.core.entity.reference.DynamicLabel;
import com.salesmanager.core.entity.reference.ModuleConfiguration;
import com.salesmanager.core.entity.reference.Page;
import com.salesmanager.core.entity.reference.Portlet;
import com.salesmanager.core.entity.system.Field;
import com.salesmanager.core.service.ServiceFactory;
import com.salesmanager.core.service.merchant.ConfigurationRequest;
import com.salesmanager.core.service.merchant.ConfigurationResponse;
import com.salesmanager.core.service.merchant.MerchantService;
import com.salesmanager.core.service.reference.ReferenceService;
import com.salesmanager.core.util.CountryUtil;
import com.salesmanager.core.util.ConfigurationFieldUtil;


public class FbPageAdminAction extends BaseAction {
	
	//private Logger log = Logger.getLogger(PortletPageAdminAction.class);
	private String htmlCode = null;
	
	private Page page;
	
	private List portletList = new ArrayList();//menu list
	private Map displayedPortlets = new HashMap();//portlets arranged in the display
	
	
	/**
	 * Create a new facebook page
	 * @return
	 */
	public String createPage() throws Exception {
		
		super.setPageTitle("integration.fbadmin.title");
		Page page = new Page();
		page.setTitle(IntegrationConstants.FB_PAGE);
		page.setMerchantId(super.getContext().getMerchantid());
		page.setProperty1("basic.jsp");
		page.setEnabled(false);
		page.setVisible(false);
		page.setSecured(false);
		page.setStyle(1);
		
		ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		
		rservice.saveOrUpdatePage(page);
		
		return SUCCESS;

	}
	
	
	public String displayPage() throws Exception {
		
		
		super.setPageTitle("integration.fbadmin.title");
		
  
		  
		MerchantService mservice = (MerchantService)ServiceFactory.getService(ServiceFactory.MerchantService);
		MerchantStore store = mservice.getMerchantStore(super.getContext().getMerchantid());
		
		
		String countryCode = CountryUtil.getCountryIsoCodeById(store.getCountry());
		ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		
		
		//get Page
		page = rservice.getPage(IntegrationConstants.FB_PAGE, store.getMerchantId());//facebook page
		
		if(page != null && !StringUtils.isBlank(page.getProperty1())) {
			super.getServletRequest().setAttribute("paageTemplate", page.getProperty1());
		} else {
			super.getServletRequest().setAttribute("paageTemplate", "basic.jsp");
		}
				
		if(page==null) {
			return SUCCESS;
		}
		
		
		super.getServletRequest().setAttribute("ApplicationID", page.getProperty2());
		super.getServletRequest().setAttribute("APIKey", page.getProperty5());
		super.getServletRequest().setAttribute("ApplicationSecret", page.getProperty4());
		
		
		List portlets = new ArrayList();
		List portletNames = new ArrayList();
		Map portletsFields = new HashMap();//configurable fields MODULE,MAP<String(fieldName),Field>
		Map<String,List<Field>> fieldValues = null;//configurable fields values
		
		//getting module portlets from sm-core-modules
		Collection services = rservice.getCoreModules(LabelConstants.FB_PAGE, countryCode);
		for(Object o: services) {
			
			CoreModuleService service = (CoreModuleService)o;
			Portlet p = new Portlet();
			p.setPortletType(LabelConstants.PORTLET_TYPE_MODULE);
			p.setTitle(service.getCoreModuleName());
			p.setName(service.getCoreModuleServiceDescription());
			portlets.add(p);
			portletNames.add(p.getTitle());
		}
		
		
		//get fields
		//create a new method for getting multiple config
		Collection confs = rservice.getModuleConfigurations(portletNames);
		if(confs!=null && confs.size()>0) {
			
			for(Object o: confs) {
				ModuleConfiguration conf = (ModuleConfiguration)o;
				if(conf.getId().getConfigurationKey().equals("fields")) {
					Map<String,Field> fields = ConfigurationFieldUtil.parseFields(conf.getConfigurationValue());
					portletsFields.put(conf.getId().getConfigurationModule(), fields);
				}
			}
		}
		

		
		if(portletsFields.size()>0) {
			
			
			//get fields values
			//merchant_configuration
			//config_key = page-name
			//config_value = {"fields":[{"module":"moduleName","values":[{"name":"fieldName","value":"fieldValue"}...]}...]}
			ConfigurationRequest configRequest = new ConfigurationRequest(store.getMerchantId(),true,ConfigurationFieldUtil.getMerchantConfigurationKeyLike(page.getTitle()));
			ConfigurationResponse configResponse = mservice.getConfiguration(configRequest);
			
			List<MerchantConfiguration> configs = configResponse.getMerchantConfigurationList();
			
			//MerchantConfiguration conf = configResponse.getMerchantConfiguration("PAAGE_" + String.valueOf(page.getPageId()));
			if(configs!=null && configs.size()>0) {
				
				List sArrayList = new ArrayList();
				for(Object o: configs) {
					
					MerchantConfiguration conf = (MerchantConfiguration)o;
					sArrayList.add(conf.getConfigurationValue());
					
				}
				

				fieldValues = ConfigurationFieldUtil.parseFieldsValues(sArrayList);
			
			}
			
			if(fieldValues!=null && fieldValues.size()>0) {
				
/*				for(Object o : portletsFields.keySet()) {
					String module = (String)o;
					Map configurableFields = (Map)portletsFields.get(module);
					if(configurableFields!=null) {
						List fieldsList = (List)fieldValues.get(module);
						for(Object of: fieldsList) {
							Field f =(Field)of;
							Field configurableField = (Field)configurableFields.get(f.getName());
							if(configurableField!=null) {
								configurableField.setFieldValue(f.getFieldValue());
							}
						}
					}
				}*/
				
			}
		}
		
		super.getServletRequest().setAttribute("fields", portletsFields);
		super.getServletRequest().setAttribute("fieldsvalues", fieldValues);
		
		
		
		//get portlets from Dynamic Label (also present in portlets table)
		Collection labels = rservice.getDynamicLabels(store.getMerchantId(), 200);
		for(Object o: labels) {
			
			DynamicLabel label = (DynamicLabel)o;
			if(label.isVisible()) {
				Portlet p = new Portlet();
				p.setPortletType(LabelConstants.PORTLET_TYPE_LABEL);
				p.setTitle(label.getTitle());
				p.setName(label.getTitle());
				p.setLabelId(label.getDynamicLabelId());
				portlets.add(p);
			}
		}
		
		//get configured portlets
		Collection configuredPortlets = rservice.getPortlets(page.getPageId(), super.getContext().getMerchantid());
		
		//Map modulesColumn = new HashMap();
		Map portletsColumn = new HashMap();
		
		Map portletsTitle = new HashMap();
		
		
		for(Object o: configuredPortlets) {
			
			Portlet p = (Portlet)o;
			portletsTitle.put(p.getTitle(), p);
			
			List list = (List)portletsColumn.get(p.getColumnId());
			if(list==null) {
				list = new ArrayList();
				portletsColumn.put(p.getColumnId(), list);
			}
			
			list.add(p);
			

		}
		
		/** Portlet configured cannot appear on the deck **/
		for(Object o: portlets) {
			Portlet p = (Portlet)o;
			if(!portletsTitle.containsKey(p.getTitle())) {
				portletList.add(p);
			}
		}
		
		
		super.getServletRequest().setAttribute("poortlets", portletsColumn);

		
		return SUCCESS;
	}
	
	public String editPageHeader() throws Exception {
		
		Page localPage = this.getPage();
		
		this.displayPage();
		
		ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		
		if(localPage==null) {
			throw new Exception("editPaageHeader - Paage is null");
		}
		
		Page editPage = rservice.getPage(localPage.getPageId(), super.getContext().getMerchantid());
		
		if(editPage == null) {
			throw new Exception("editPaageHeader - editPaage is null");
		}
		
		editPage.setHeader(localPage.getHeader());
		
		rservice.saveOrUpdatePage(editPage);
		
		super.setSuccessMessage();
		
		return SUCCESS;
	}
	
	public String editPageConfig() throws Exception {
		
		Page localPage = this.getPage();
		
		this.displayPage();
		
		
		ReferenceService rservice = (ReferenceService)ServiceFactory.getService(ServiceFactory.ReferenceService);
		
		if(localPage==null) {
			throw new Exception("editPaageHeader - Paage is null");
		}
		
		Page editPage = rservice.getPage(localPage.getPageId(), super.getContext().getMerchantid());
		
		if(editPage == null) {
			throw new Exception("editPaageHeader - editPaage is null");
		}
		
		editPage.setVisible(localPage.getVisible());
		editPage.setSecured(localPage.getSecured());
		editPage.setProperty1(localPage.getProperty1());
		editPage.setProperty2(localPage.getProperty2());
		editPage.setProperty3(localPage.getProperty3());
		editPage.setProperty4(localPage.getProperty4());
		editPage.setProperty5(localPage.getProperty5());
		editPage.setProperty6(localPage.getProperty6());
		editPage.setProperty7(localPage.getProperty7());
		editPage.setProperty8(localPage.getProperty8());
		editPage.setProperty9(localPage.getProperty9());
		editPage.setProperty10(localPage.getProperty10());
		
		rservice.saveOrUpdatePage(editPage);
		
		super.setSuccessMessage();
		
		return SUCCESS;
	}

	public String getHtmlCode() {
		return htmlCode;
	}

	public void setHtmlCode(String htmlCode) {
		this.htmlCode = htmlCode;
	}

	public List getPortletList() {
		return portletList;
	}

	public void setPortletList(List portletList) {
		this.portletList = portletList;
	}

	public Map getDisplayedPortlets() {
		return displayedPortlets;
	}

	public void setDisplayedPortlets(Map displayedPortlets) {
		this.displayedPortlets = displayedPortlets;
	}

	public Page getPage() {
		return page;
	}

	public void setPage(Page page) {
		this.page = page;
	}

}



```
